# pbi-stress-tester

Python tooling for load-testing Power BI Embedded reports: it embeds one or
more reports behind a local Flask server, uses Selenium to open them in
multiple browser tabs to simulate concurrent viewers, and (in the advanced
notebook) correlates each report open against the Power BI Embedded
capacity's live CPU usage.

There are two notebooks here, not one tool. They share the same embedding
approach but cover different scenarios.

| | `simple_pbi_loadtesting.ipynb` | `pbi-embed-performance-check.ipynb` |
|---|---|---|
| Report navigation | Opens the report's default page only | Navigates to a specific page inside a paginated report |
| Filters | Not applied | Filter payload is scaffolded in the embed config, commented out by default |
| CPU monitoring | None | Queries the Power BI Embedded capacity's CPU metric via Azure Monitor on every report open |
| Logging | Console output only | Appends timestamped report + CPU usage lines to `Load_Testing.txt` |
| Use it when | You just need N tabs open at once to see if anything falls over | You want to know what a report open actually costs the capacity, page by page |

## How it works

Both notebooks follow the same pattern:

1. A small Flask app exposes `/report/<index>`, which looks up a report in
   a `REPORTS` list you provide, acquires an Azure AD access token via MSAL
   (client credentials flow), calls the Power BI REST API to fetch the
   report's embed URL, then calls `GenerateToken` to mint a scoped embed
   token for it.
2. That route renders an HTML page that loads the report with the
   [Power BI JavaScript SDK](https://github.com/microsoft/PowerBI-JavaScript),
   using the embed URL and token from step 1.
3. Selenium drives a real browser (the notebooks are set up for Brave, but
   any Chromium-based browser plus a matching chromedriver works) and opens
   `/report/0`, `/report/1`, ... in a new tab each, with a delay between
   tabs, so you end up with several reports rendering at once.
4. `pbi-embed-performance-check.ipynb` additionally: waits for the report to
   report `loaded`, calls `report.setPage()` to jump to the paginated report
   page you configured, and on the server side queries the target Power BI
   Embedded capacity's `cpu_metric` through `azure-monitor-query` right
   before serving each embed, logging the result alongside the report name
   and timestamp.

Nothing here calls a Power BI REST "run load test" endpoint. The load is
real browser tabs rendering real embedded reports; this is closer to a
manual load test than an automated benchmark harness; see
[Known limitations](#known-limitations) for what that means in practice.

## Requirements

- Python 3, with `flask`, `selenium`, `msal`, `requests` (all notebooks),
  plus `azure-identity` and `azure-monitor-query` for
  `pbi-embed-performance-check.ipynb` only.
- A Chromium-based browser and a matching chromedriver on the machine
  running Selenium. The notebooks default to Brave; point
  `brave_path`/`chromedriver_path` at Chrome instead if that's what you have.
- An Entra ID (Azure AD) app registration with API permissions to call the
  Power BI REST API, plus a client secret. Azure Portal → App registrations
  → New registration, then API permissions → Power BI Service.
- A Power BI workspace (group) containing the reports you want to test, with
  that app registration (or the account it represents) able to view them.
- For CPU tracking only: a Power BI Embedded capacity, and the app
  registration needs read access to that capacity's metrics in Azure Monitor
  (`Microsoft.Insights/metrics/read` on the capacity resource, typically via
  the Monitoring Reader role).

## Setup

1. Install dependencies:

   ```bash
   pip install flask selenium msal requests azure-identity azure-monitor-query
   ```

2. In whichever notebook you're running, fill in the configuration block:

   ```python
   CLIENT_ID = "..."       # App registration's Application (client) ID
   CLIENT_SECRET = "..."   # App registration's client secret
   TENANT_ID = "..."       # Directory (tenant) ID
   WORKSPACE_ID = "..."    # Power BI workspace (group) ID
   USERNAME = "..."        # Effective identity username for row-level security, if any
   ROLES = ["..."]         # Effective identity roles for row-level security; remove if unused
   ```

   `pbi-embed-performance-check.ipynb` also needs, for CPU tracking:

   ```python
   subscription_id = "..."
   resource_group = "..."
   capacity_name = "..."
   ```

3. Populate `REPORTS` with the reports to test. The two notebooks expect
   different shapes:

   ```python
   # simple_pbi_loadtesting.ipynb
   REPORTS = [
       {"report_id": "...", "dataset_id": "..."},
   ]

   # pbi-embed-performance-check.ipynb
   REPORTS = [
       {
           "report_name": "...",
           "report_id": "...",       # the report you embed
           "report_id_p": "...",     # the paginated report you navigate into
           "dataset_id": "...",
           "page_name": "...",       # internal page name, e.g. ReportSectionabc123
       },
   ]
   ```

   `page_name` is not the display name shown in the Power BI UI. Open the
   paginated report page you care about and read the internal name out of
   the URL, the segment right after the report ID.

4. Point `brave_path` and `chromedriver_path` in `run_selenium` at your
   actual browser and chromedriver locations.

## Running it

Each notebook's last cell is the entry point: it starts Flask on a
background thread, waits for it to come up, then drives Selenium to open
one tab per entry in `REPORTS`, `delay_time` seconds apart (10s by default).
Run the notebook's cells top to bottom; the last cell blocks while the tabs
open.

For `pbi-embed-performance-check.ipynb`, check `Load_Testing.txt` afterward.
Each line looks like:

```
[2025-01-01 12:00:00], Paginated Report opened: Q4 Summary, Report ID: <id>, Usage: (CPU Time: ..., Avg CPU: 42.1)
```

## Known limitations

This is closer to a manual load-testing rig than a repeatable benchmark
tool, and it's worth being direct about what that means:

- **Concurrency is Selenium tabs, not real concurrent users.** Tabs open
  sequentially with a fixed delay between them, so this measures "several
  tabs rendering at once after a staggered ramp-up," not a true simultaneous
  burst of N users.
- **`REPORTS` is not populated in-notebook.** Both notebooks assume you
  fill it in yourself, either by hand or from your own data source; there's
  a commented-out example in each showing the expected shape.
- **Filters and the adaptive backoff are scaffolded, not wired up.** The
  filter definition in `pbi-embed-performance-check.ipynb`'s embed config
  is commented out, as is the logic in `run_selenium` that would speed up
  or slow down tab opening based on the live CPU reading. Both are there to
  extend, not to use as-is.
- **Browser and driver paths are hardcoded** to a specific Windows Brave
  install and chromedriver location. Update them for your machine before
  running.
- **Credentials are plaintext placeholders in the notebook.** Don't commit
  a notebook with real `CLIENT_SECRET`, tenant, or workspace values filled
  in; load them from environment variables or a local, gitignored config
  instead if you're extending this for real use.
