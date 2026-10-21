# RDW-Pull-AWS

Weekly pipeline that downloads the RDW open data CSV **Gekentekende_voertuigen** (dataset `m9d7-ebf2`, about 16.9 million rows, about 13 GB) and loads it into DynamoDB. It runs entirely in AWS and is defined in one CloudFormation template, `rdw-pipeline.yaml`.

## How it works

```
EventBridge Scheduler ──> Fargate task ──> S3 ──> DynamoDB import ──> SSM parameter
   (Mondays, 04:30 UK)    (AWS CLI image)   (7 days)  (new table)      /rdw/current-table
```

Each run:

1. Checks the dataset's `rowsUpdatedAt` on RDW and stops if nothing has changed since the last import.
2. Gets RDW's own row count, then downloads the full CSV export from `/api/v3/views/<dataset>/export.csv`, gzipping it as it arrives. The uncompressed file never touches disk.
   - **No app token needed.** This is Socrata's SODA3 full export, which public datasets serve without one.
   - **Uncompressed transfer.** The transfer itself isn't compressed.
   - **4-hour cap.** It stops a connection that hangs without closing.
3. Rejects any download with fewer than 99.9% of the expected rows. That includes downloads that end early without an error. It retries up to 3 times.
4. Uploads the file to S3, named exactly as the portal's Export button names it, with `.gz` added (for example `Open_Data_RDW__Gekentekende_voertuigen_20261021.csv.gz`, dated with the download day). S3 keeps it for 7 days, with RDW's own update time and the row count attached as object metadata.
5. Imports it into a brand-new table, `rdw-vehicles-<UTC timestamp>`, with on-demand billing and `Kenteken` as the key.
6. Checks the number of items imported. If it looks right, it points the SSM parameter `/rdw/current-table` at the new table, and records the RDW file name it came from in `/rdw/current-source-file`.
7. Deletes all but the 2 newest tables, keeping the previous one to fall back to.

If RDW's dataset metadata (update time and name) can't be read, the run carries on anyway: it can't skip unchanged days, and it names the file `<dataset>_<YYYYMMDD>.csv`. A failed or incomplete run leaves the live table untouched. Apps should read the table name from `/rdw/current-table` rather than hard-coding it, so they switch to new data with no downtime.

## Deploy

You need the AWS CLI, signed in as a user who can create IAM roles and the stack's other resources (S3, ECS, EventBridge, SNS, CloudWatch), with your default region set to where the data should live. RDW's portal is hosted in AWS `eu-west-1` (Ireland), so deploying there gives the fastest download.

Find your default VPC and its subnets:

```
aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query "Vpcs[0].VpcId" --output text
aws ec2 describe-subnets --filters Name=default-for-az,Values=true --query "Subnets[].SubnetId" --output text
```

Deploy, with the subnets comma-separated:

```
aws cloudformation deploy --template-file rdw-pipeline.yaml --stack-name rdw-pipeline --capabilities CAPABILITY_IAM --parameter-overrides VpcId=vpc-xxxx SubnetIds=subnet-aaaa,subnet-bbbb AlertEmail=you@example.com
```

AWS then emails `AlertEmail` a subscription confirmation. Click the link in it, or no alerts will arrive.

The task gets a public IP to reach RDW, so it needs public subnets. The default VPC's subnets are public, and no NAT gateway is needed.

### Parameters

| Parameter | Default | What it does |
|---|---|---|
| `VpcId`, `SubnetIds` | (required) | Where the task runs |
| `ScheduleExpression` | `cron(30 4 ? * MON *)` | When it runs, in Europe/London time. Weekly on Mondays at 04:30 by default |
| `DatasetId` | `m9d7-ebf2` | RDW dataset to pull |
| `KeyAttribute` | `Kenteken` | CSV column used as the table key. Must match the header exactly (case-sensitive) |
| `TablePrefix` | `rdw-vehicles` | Prefix for the table names |
| `KeepTables` | `2` | How many of the newest tables to keep |
| `ParamPrefix` | `/rdw` | SSM path for `current-table`, `current-source-file` and `last-updated` |
| `AlertEmail` | (none) | Where alert emails go. Without it, alerts are sent to the SNS topic but nobody receives them |

## Alerts

Alerts go to the email address you deployed with:

- **A run failed.** Sent as soon as a run ends with an error, with the exit code and reason. That includes a download that was cut short three times, a failed import and too few items imported. The live table is left as it was.
- **A run couldn't start.** For example, the container image couldn't be pulled or the network setup is wrong.
- **RDW announced a deprecation.** If any RDW response carries a `Deprecation` or `Sunset` header, the email lists which request it was and the notice, including the sunset date and RDW's info link. You get it once, and again only if the notice changes. It's also written to the run's log every time.

To send alerts to more addresses, subscribe them to the SNS topic in the stack's `AlertTopicArn` output.

## Run it now

The stack outputs a ready-made command that starts a run immediately:

```
aws cloudformation describe-stacks --stack-name rdw-pipeline --query "Stacks[0].Outputs[?OutputKey=='RunNowCommand'].OutputValue" --output text
```

Run the command it prints, then follow the logs:

```
aws logs tail /ecs/rdw-pipeline --follow
```

A full run takes the download (about 40 minutes over a home connection, likely less from inside AWS) plus the import. To force a run when RDW's data hasn't changed, delete the `/rdw/last-updated` parameter first.

## Reading the data

```bash
TABLE=$(aws ssm get-parameter --name /rdw/current-table --query Parameter.Value --output text)
aws dynamodb get-item --table-name "$TABLE" --key '{"Kenteken":{"S":"AB123C"}}'

# Which RDW file the live table was built from
aws ssm get-parameter --name /rdw/current-source-file --query Parameter.Value --output text
```

## Notes

- **One row per plate is assumed.** The completeness check compares the items imported with the rows downloaded. If some plates appear more than once, DynamoDB merges them, and if that's more than 0.1% of rows every run is rejected with "Too few items imported".
- **Same format as the Export button.** The SODA3 export gives the portal's Export button format. A full laptop download was compared with a button export: they came out the same size, and a spot-checked record was identical, character for character. Numbers can carry thousands separators (`1,573`, and a year as `2,025`), and the DT date columns read like `2027 May 16 12:00:00 AM`. Anything that already handles the button's file handles this data the same way.
- **Columns come in as text.** Every column other than the key is imported as a string, exactly as written in the file, so convert dates and numbers in your app if you need them typed.
- **Removing the stack doesn't remove the data.** The tables are created by the task, not by the stack, so delete the `rdw-vehicles-*` tables yourself. Empty the S3 bucket first, or the stack deletion fails.
- **Not yet run against a live account.** The template's structure and references have been checked, and the embedded script was dry-run against fake RDW and AWS commands (first run, no change, new data, table clean-up, cut-off download, failed import, too few items, wrong key column, RDW's metadata being unavailable, and deprecation emails being sent once, repeated only when the notice changes, and sent even when the run fails). The first real deploy and run are still the real test.
