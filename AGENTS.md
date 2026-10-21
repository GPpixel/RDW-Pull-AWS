## Code Review Rules
- Scope: flag any feature, resource, alert, schedule or dependency the pull request
  description didn't ask for. Extras belong in their own pull request.
- External facts: flag claims about third-party APIs, limits, prices, data formats or
  dates with no source link, and any external URL that is deprecated or due to be removed.
- Evidence: flag "tested", "verified" or "identical" in the README or pull request
  without evidence of that test in the pull request.
- Readers: the README is for colleagues who deploy and use this. Flag costs, superseded
  approaches, debugging history and anything that only made sense during development.
- Copy-paste: flag commands that won't run as written: truncated URLs, unmarked
  placeholders, the wrong shell for the stated OS.
- Consistency: flag settings that contradict each other, such as an alert window shorter
  than the run interval, or file retention shorter than the gap between runs.
- Silent failure: flag any path where a failure leaves stale or wrong data without
  anyone being told.
