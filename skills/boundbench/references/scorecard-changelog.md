# Scorecard changelog

Every rule change bumps `scorecard_version`. Entries carry the version they were scored under; the website flags entries older than the current catalog. When a change can move ratings, rerun every existing scorecard under the new version and log rating changes in each `reaudit_log`.

## 0.9
First public release. All published scores are labelled 0.9.
