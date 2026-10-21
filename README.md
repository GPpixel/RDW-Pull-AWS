# RDW-Pull-AWS

Daily pipeline that downloads the RDW open data CSV **Gekentekende_voertuigen** (dataset `m9d7-ebf2`, about 17 million rows) and loads it into DynamoDB. It runs entirely in AWS and is defined in one CloudFormation template, `rdw-pipeline.yaml`.

## How it works

```
EventBridge Scheduler ──> Fargate task ──> S3 ──> DynamoDB import ──> SSM parameter
   (daily, 04:30 UK)      (AWS CLI image)   (7 days)  (new table)      /rdw/current-table
```

Each run:

1. Checks the dataset's `rowsUpdatedAt` on RDW and stops if nothing has changed since the last import.
2. Gets RDW's own row count, then downloads the full CSV export from `/api/v3/views/<dataset>/export.csv`, gzipping it as it arrives. The uncompressed file never touches disk.
   - **No app token needed.** This is Socrata's SODA3 full export, which public datasets serve without one.
   - **Not the old export URL.** The older `/api/views/<dataset>/rows.csv` export is deprecated, and Socrata removes it in December 2026.
   - **Uncompressed transfer.** The transfer itself isn't compressed, because a compressed transfer was seen to stop early.
   - **4-hour cap.** It stops a connection that hangs without closing.
3. Rejects any download with fewer than 99.9% of the expected rows. That includes downloads that end early without an error. It retries up to 3 times.
4. Uploads the file to S3, named like RDW's own downloads with RDW's update date added for versioning (for example `Open_Data_RDW__Gekentekende_voertuigen_2026-10-20.csv.gz`). S3 keeps it for 7 days, with RDW's update time and the row count attached as object metadata.
5. Imports it into a brand-new table, `rdw-vehicles-<UTC timestamp>`, with on-demand billing and `Kenteken` as the key.
6. Checks the number of items imported. If it looks right, it points the SSM parameter `/rdw/current-table` at the new table, and records the RDW file name it came from in `/rdw/current-source-file`.
7. Deletes all but the 2 newest tables, keeping the previous one to fall back to.

If RDW's dataset metadata (update time and name) can't be read, the run carries on anyway: it can't skip unchanged days, and it names the file `<dataset>_<run date>.csv`. A failed or incomplete run leaves the live table untouched. Apps should read the table name from `/rdw/current-table` rather than hard-coding it, so they switch to new data with no downtime.

## Deploy

You need the AWS CLI, signed in, with your default region set to where the data should live. RDW's portal is hosted in AWS `eu-west-1` (Ireland), so deploying there gives the fastest download.

Find your default VPC and its subnets:

```
aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query "Vpcs[0].VpcId" --output text
aws ec2 describe-subnets --filters Name=default-for-az,Values=true --query "Subnets[].SubnetId" --output text
```

Deploy, with the subnets comma-separated:

```
aws cloudformation deploy --template-file rdw-pipeline.yaml --stack-name rdw-pipeline --capabilities CAPABILITY_IAM --parameter-overrides VpcId=vpc-xxxx SubnetIds=subnet-aaaa,subnet-bbbb
```

The task gets a public IP to reach RDW, so it needs public subnets. The default VPC's subnets are public, and no NAT gateway is needed.

### Parameters

| Parameter | Default | What it does |
|---|---|---|
| `VpcId`, `SubnetIds` | (required) | Where the task runs |
| `ScheduleExpression` | `cron(30 4 * * ? *)` | When it runs, in Europe/London time |
| `DatasetId` | `m9d7-ebf2` | RDW dataset to pull |
| `KeyAttribute` | `Kenteken` | CSV column used as the table key. Must match the header exactly (case-sensitive) |
| `TablePrefix` | `rdw-vehicles` | Prefix for the table names |
| `KeepTables` | `2` | How many of the newest tables to keep |
| `ParamPrefix` | `/rdw` | SSM path for `current-table`, `current-source-file` and `last-updated` |

## Run it now

The stack outputs a ready-made command that starts a run immediately:

```
aws cloudformation describe-stacks --stack-name rdw-pipeline --query "Stacks[0].Outputs[?OutputKey=='RunNowCommand'].OutputValue" --output text
```

Run the command it prints, then follow the logs:

```
aws logs tail /ecs/rdw-pipeline --follow
```

A full run takes the download time (30+ minutes) plus the import time. To force a run when RDW's data hasn't changed, delete the `/rdw/last-updated` parameter first.

## Reading the data

```bash
TABLE=$(aws ssm get-parameter --name /rdw/current-table --query Parameter.Value --output text)
aws dynamodb get-item --table-name "$TABLE" --key '{"Kenteken":{"S":"AB123C"}}'

# Which RDW file the live table was built from
aws ssm get-parameter --name /rdw/current-source-file --query Parameter.Value --output text
```

## Notes

- **Columns come in as text.** Every column other than the key is imported as a string, so convert dates and numbers in your app if you need them typed.
- **Removing the stack doesn't remove the data.** The tables are created by the task, not by the stack, so delete the `rdw-vehicles-*` tables yourself. Empty the S3 bucket first, or the stack deletion fails.
- **Not yet run against a live account.** The template's structure and references have been checked, and the embedded script was dry-run against fake RDW and AWS commands (first run, no change, new data, table clean-up, cut-off download, failed import, too few items, wrong key column, and RDW's metadata being unavailable). The first real deploy and run are still the real test. A full download from the SODA3 export route still needs comparing with the portal's Export button file before deploying.
