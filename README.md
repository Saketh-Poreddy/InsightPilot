# InsightPilot AI

[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/downloads/)
[![Live dashboard](https://img.shields.io/badge/demo-live-brightgreen.svg)](https://storage.googleapis.com/insightspilot-ai/Article_Insights_AI.html)

A self-refreshing dashboard that explains **why** a metric moved, not just that
it moved. It rebuilds itself weekly from BigQuery and runs for under a dollar a
year on Google Cloud.

Most dashboards report that the return rate moved a fifth of a point and stop
there. This one splits that change into the only two things it can be:

- **Rate effect** — customers inside each segment behaved differently.
- **Mix effect** — customers behaved the same, but the composition shifted.

Those point at different teams and different fixes, and a trend line cannot
tell them apart.

[![The InsightPilot dashboard](docs/cover.png)](https://storage.googleapis.com/insightspilot-ai/Article_Insights_AI.html)

**[Open the live dashboard →](https://storage.googleapis.com/insightspilot-ai/Article_Insights_AI.html)**
The page rebuilds weekly, so the figures there will differ from the ones
recorded below. That is the behavior being demonstrated.

## Contents

- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Using it on your own data](#using-it-on-your-own-data)
- [Reading the numbers](#reading-the-numbers)
- [Known pitfalls](#known-pitfalls)
- [Cost](#cost)
- [Citation](#citation)
- [License](#license)

---

## How it works

```
 BigQuery public dataset           Vertex AI scheduled job              Cloud Storage
 thelook_ecommerce       ──SQL──▶  (weekly schedule, runs as   ──PUT──▶  Article_Insights_AI.html
 orders · users · products         a service account)                    no-cache, one URL
                                   1. measure the return lag                  │
                                   2. run the grain query                     ▼
                                   3. inject rows into the page        a permanent link
```

No server, no BI license, no build step. The dashboard is a single HTML file
with its data embedded as a JavaScript array, so it opens instantly, works
offline, and runs anywhere a browser does.

| Path | What it is |
|---|---|
| `refresh_dashboard.py` | **The whole job, in one file.** Queries, checks, injects, publishes |
| `sql/weekly_grain.sql` | The weekly grain — week × traffic source × category, as counts |
| `sql/return_lag.sql` | Measures how late returns arrive, which sets the censoring guard |
| `dashboard/index.html` | The dashboard itself, and the template the job fills in |
| `dashboard/data.json` | A generated sample payload, so the page works with no cloud account |
| `scripts/deploy_template.sh` | Copies the page and the queries to the bucket for the scheduled job |
| `scripts/make_sample_data.py` | Generates `data.json` |
| `scripts/inject_data.py` | Puts a payload into the page locally |
| `prompts/` | The prompts used to build this, kept verbatim |

### One implementation, two places to read it from

The repository is the source of truth. A Vertex AI job has no checkout, so
`scripts/deploy_template.sh` copies the page template and the two queries to
`gs://BUCKET/template/`, and the job reads them from there.

`refresh_dashboard.py` resolves each input the same way every time: **a local
file wins if it exists, otherwise the deployed copy in the bucket.** Run it from
a clone and it uses your working tree; run it on Vertex AI and it uses what you
deployed. One authored file, one deployed copy of that file, and no third
version drifting out of sync.

The deployed template lives under `template/` while the published page sits at
the bucket root, so a bad run can overwrite the output but never the input it
was built from.

---

## Requirements

| | |
|---|---|
| Python | 3.9 or later |
| Dependencies | `google-cloud-bigquery`, `google-cloud-storage` — pinned in [`requirements.txt`](requirements.txt) |
| Google Cloud | A project with BigQuery, Cloud Storage and Vertex AI enabled |
| CLI | [`gcloud`](https://cloud.google.com/sdk/docs/install), for authentication and bucket creation |

```bash
pip install -r requirements.txt
```

The versions are pinned exactly rather than as ranges, because this repository
is published alongside an article and will be cloned long after it was written.
`requirements.txt` explains how to move them forward.

Viewing the dashboard with the committed sample payload requires none of the
above beyond Python itself.

---

## Quick start

See it before touching Google Cloud:

```bash
python refresh_dashboard.py --data dashboard/data.json --no-upload --out page.html
```

That fills the page from the committed sample payload and writes it to
`page.html` in the working directory. The sample is labeled as sample on screen,
so the layout can be judged without an account. Then:

1. **Create a Google Cloud project** and enable BigQuery, Cloud Storage and
   Vertex AI. Set a $10 budget alert before anything else runs.
2. **Authenticate for the client libraries**, which is not the same thing as
   authenticating the CLI:

   ```bash
   gcloud auth application-default login
   ```

3. **Set your own project and bucket** at the top of `refresh_dashboard.py`, or
   pass `IP_PROJECT` and `IP_BUCKET` as environment variables.
4. **Price the query first.** This is free, and it is the only opportunity to
   catch an expensive query before paying for it:

   ```bash
   python refresh_dashboard.py --estimate-only
   ```

5. **Create the bucket and publish.** Private by construction — uniform
   bucket-level access on, public access prevention enforced:

   ```bash
   python refresh_dashboard.py --create-bucket
   ```

6. **Deploy the inputs** so a job with no checkout can find them:

   ```bash
   ./scripts/deploy_template.sh
   ```

7. **Schedule it weekly** as a Vertex AI custom job running
   `python refresh_dashboard.py`. It must run as a service account rather than
   as you — see [Known pitfalls](#known-pitfalls), as this is the step that most
   commonly fails on a first attempt. The service account needs **BigQuery Job
   User** on the project and **Storage Object User** on the bucket.

### Publishing the page publicly

Two changes, in this order:

```bash
gcloud storage buckets update gs://YOUR-BUCKET --no-public-access-prevention
```

```bash
gcloud storage buckets add-iam-policy-binding gs://YOUR-BUCKET \
  --member=allUsers --role=roles/storage.objectViewer
```

The permanent link is then `storage.googleapis.com/BUCKET/OBJECT`.

**That grant is bucket-wide, and it cannot be narrowed.** The obvious instinct
is to scope it to the single object you want served, using an IAM condition on
`resource.name`. Cloud Storage refuses:

```
HTTPError 400: PublicResourceAllowConditionCheck
Error: Conditions are not allowed on public resources.
```

Per-object ACLs are the other route, and uniform bucket-level access — which
you want enabled, because it removes per-object ACLs as a leak path — disables
exactly that. So the accurate model is: **a bucket is public or it is not.**
Treat a public one as a public web root. Everything in it is world-readable,
including the `template/` prefix, which is why the template here carries sample
data and the queries are the same ones already published in this repository.

If the job's inputs must stay private, they do not belong in the same bucket as
the page. Put them in a second, private bucket, or — since this repository is
public — point `--template` and `--sql` at GitHub raw URLs and keep the public
bucket holding nothing but the published page.

For an internal deployment none of this applies: grant `domain:`, `group:` or
`user:` principals instead of `allUsers`, and the bucket stays private to your
organization. See [Known pitfalls](#known-pitfalls) for the one principal that
looks internal but is not.

### Creating the bucket manually

With `gcloud`, two commands:

```bash
gcloud storage buckets create gs://YOUR-BUCKET --location=us-central1 \
  --uniform-bucket-level-access --public-access-prevention
```

```bash
gcloud storage cp page.html gs://YOUR-BUCKET/Article_Insights_AI.html \
  --content-type="text/html; charset=utf-8" --cache-control="no-cache, max-age=0"
```

**Uploading through the Cloud Console works, but sets neither header.** A
console upload gives the object `text/html` with no charset and no cache policy,
which produces both of the display problems described under
[Known pitfalls](#known-pitfalls). After a manual upload, edit the object's
metadata and set both.

Bucket names are global across all of Google Cloud rather than per project, so
the name you want may be taken; the script reports this and suggests a suffix.
Lowercase only.

---

## Using it on your own data

Nothing downstream of the query knows what a "return" is. The page receives five
columns and treats them as *a numerator, a denominator, and three labels* — so
this works for any rate that can be split two ways, not only returns.

**The contract is five column names.** The query must return exactly these,
spelled exactly this way:

| Column | Type | What it is |
|---|---|---|
| `week_start` | DATE | the period bucket, one row per period per segment |
| `traffic_source` | STRING | your first dimension |
| `category` | STRING | your second dimension |
| `returned_items` | INT64 | the **numerator** — the thing being measured |
| `total_items` | INT64 | the **denominator** — everything it could have been |

Alias your own columns to those names and `refresh_dashboard.py` needs no
changes:

```sql
SELECT
  DATE_TRUNC(DATE(signed_up_at), WEEK(MONDAY)) AS week_start,
  acquisition_channel                          AS traffic_source,
  plan_tier                                    AS category,
  COUNTIF(churned)                             AS returned_items,
  COUNT(*)                                     AS total_items
FROM `your_project.your_dataset.subscriptions`
WHERE DATE(signed_up_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 112 WEEK)
  AND DATE(signed_up_at) <  CURRENT_DATE()
GROUP BY week_start, traffic_source, category
```

That is a churn dashboard. Same code, same page, same decomposition. Conversion
by channel × device, defect rate by line × shift, click-through by placement ×
audience — all the same shape.

**Two rules the query must follow**, both of which fail quietly rather than
loudly:

- **Counts, never rates.** Return `COUNTIF(...)` and `COUNT(*)`, not a ratio.
  Counts stay additive when the reader filters; a pre-computed rate means every
  filtered view silently averages averages.
- **One statement.** Two statements make it a BigQuery script, and a dry run of
  a script reports zero bytes, which disables the cost ceiling. The job refuses
  to run on a zero estimate for this reason.

**Then rename the labels**, which live in `dashboard/index.html` in two places:

- the two dimension buttons, around line 430 — `Traffic source` and `Category`
- the `DIMS` object, around line 530, which carries the singular and plural
  nouns used in the generated sentences (`noun`, `plural`, `other`,
  `otherPlural`)

Everything else on the page — every chart, table and sentence — is derived from
the data at render time, so there is nothing further to change.

**One judgment call remains yours**, and the dashboard cannot make it: the
return-lag guard in `sql/return_lag.sql` measures how late *returns* arrive. If
your event has no settling delay — a click, a login — set `--lag-weeks 0` and
delete that query. If it has a longer one, such as a 90-day refund window, the
measurement still works, but check that 112 weeks is enough history to leave two
full comparison windows once the guard is subtracted.

---

## Reading the numbers

The overall rate is a weighted average — each segment's rate, weighted by its
share of items — so a change in the total decomposes exactly:

```
Δ rate = Σ (share_now − share_before) × rate_before     ← mix
       + Σ  share_now  × (rate_now − rate_before)       ← rate
```

There is no residual. The two terms always sum to the observed change. This is
a **Kitagawa decomposition**, published by Evelyn Kitagawa in 1955 as
*"Components of a Difference Between Two Rates"*. Her symmetric variant splits
the interaction term evenly between the two components rather than attributing
all of it to rate; both forms close exactly, and differ only in which component
receives the overlap.

### What the data shows

Measured 2026-11-07, comparing 2025-10-27 → 2026-10-19 against
2024-10-28 → 2025-10-20, with a 2-week lag guard:

| | Last year | This year | Change |
|---|---|---|---|
| Return rate | 11.616% | 11.843% | **+0.227 pt** |
| Items | 3,624 of 31,199 | 6,526 of 55,105 | |

| Split by | Mix | Rate |
|---|---|---|
| Traffic source | +0.002 pt | +0.225 pt |
| Product category | −0.003 pt | +0.231 pt |

Both splits close to a residual of approximately 1e-15, which is floating-point
noise rather than a real gap.

**These figures are not stable.** Ten days earlier the same measurement returned
+0.742 pt. The underlying dataset is synthetic and regenerates continuously —
the item count nearly doubled year over year, so the comparison windows are not
sampling a stable population. Any figure quoted from here will be out of date
quickly; link to the live page instead.

What is stable is the shape of the result: **the mix effect is approximately
zero on both dimensions, on every run so far.** The rate rose because customers
inside the segments returned more, not because the mix of customers changed —
Search moved 11.6% → 12.4%, Facebook 11.5% → 13.1%, and only Email improved.
The largest shift in segment share over a full year is 0.2 pt.

Synthetic data tends to hold its proportions steady, so a flat mix term is the
expected result here. The method earns its keep on real business data, where mix
shifts are precisely what gets mistaken for behavior.

**This is why the choice of dimension matters.** A mix effect can only appear on
a dimension that actually shifted, so "is it rate or mix?" has no answer until
the split is named. Segment by something stable and the conclusion will always
be "behavior" — correct here, but not because anything was tested.

Three decisions are built into the query, each for a reason:

- **Counts, not rates.** Counts add across any grouping; rates do not. Every
  rate is computed at display time, so filtering never averages an average.
- **Cancelled items are excluded.** They never shipped, so they could never be
  returned. Leaving them in the denominator drags every rate down.
- **The most recent weeks are dropped.** Returns arrive late, so recent weeks
  always look good. `sql/return_lag.sql` measures the 90th-percentile return lag
  and the job drops that many weeks from *both* comparison windows, keeping them
  matched.

---

## Known pitfalls

Most of these fail by producing something that looks correct rather than an
error.

**A zero-byte dry run is a broken dry run, not a free query.** BigQuery reports
`0 bytes` for a multi-statement script, so the moment a `.sql` file here holds
two statements instead of one, the cost estimate reads zero and
`maximum_bytes_billed` stops protecting anything — silently, because zero passes
under every ceiling. Measured: one statement estimates at 8.4 MB; the same query
with a second statement appended estimates at 0. The job now refuses to run on a
zero estimate. A single statement ending in a semicolon is fine.

**A single object cannot be made public.** An IAM condition scoped to one object
is rejected outright — *"Conditions are not allowed on public resources"* — and
the per-object ACL route is closed off by the uniform bucket-level access you
want enabled. Public is a property of the bucket. Plan the bucket's contents
around that rather than discovering it when a demo link is needed.

**`allAuthenticatedUsers` is not an internal audience.** It means any Google
account, personal Gmail included, which is effectively public. For internal
access use `domain:your-company.com` or a Google Group. Google blocks `allUsers`
and `allAuthenticatedUsers` grants by default in organizations created since May
2024.

**The authenticated link is not the public link.** `storage.cloud.google.com`
requires every viewer to be signed in with access, and signed URLs expire. For
public data, make the object public and share the `storage.googleapis.com` URL,
which is permanent. Never do this with a bucket holding anything that is not
already public.

**A private HTML object does not render in the browser.** While the bucket is
private, `storage.cloud.google.com` serves the file as a *download* rather than
displaying it, so following the link saves the file instead of showing the
dashboard. That is the endpoint's behavior, not a problem with the file — the
downloaded copy works identically, because the page is self-contained.
In-browser rendering from a URL requires a public object.

**There are two different logins.** `gcloud auth login` authenticates the CLI;
`gcloud auth application-default login` writes the credentials that Python
client libraries read. The CLI can work perfectly while every script fails with
`DefaultCredentialsError`. If the consent screen grants only `openid` and
`userinfo.email`, the `cloud-platform` scope was not selected and the libraries
still will not work.

**A schedule runs as you by default.** A scheduled job using end-user
credentials breaks silently when your token, permissions or job change. Run it
as a service account. If you schedule a Colab Enterprise notebook rather than
this script, you must create a runtime template with **"Enable end-user
credentials" turned off** before the service-account option appears at all, then
grant that account **Code Viewer** on the notebook, or the first run fails
because it cannot open the notebook it is meant to run. Running a plain script,
as here, avoids the second problem entirely.

**Analytics belong in the template, not the published page.** Any snippet added
to the live object is overwritten by the next refresh without a trace. Add it to
`dashboard/index.html` and deploy.

Two smaller ones, both of which produce a page that looks broken rather than an
error message:

- Upload with `Content-Type: text/html; charset=utf-8`. Without the charset,
  every dash and minus sign renders as mojibake.
- Set `Cache-Control: no-cache`. The URL never changes, so otherwise viewers
  continue to see the previous week's numbers.

---

## Cost

Prices change and vary by region. These are US on-demand figures — check the
[BigQuery](https://cloud.google.com/bigquery/pricing) and
[Cloud Storage](https://cloud.google.com/storage/pricing) pricing pages before
quoting them. The job prints the real query figure on every run, and that is the
one to trust:

```bash
python refresh_dashboard.py --estimate-only
```

| Line | Rate | Free each month | A weekly refresh |
|---|---|---|---|
| BigQuery analysis | ~$6.25 / TiB scanned | the first 1 TiB | ~8 MB a week, under 0.1% of the free TiB |
| Vertex AI compute | ~$0.13–0.19 / hour | — | a few minutes a week, under 2¢ a run |
| Cloud Storage | ~$0.020 / GB-month | 5 GB | a 200 KB file, $0.000005 a month |
| Class A ops (uploads) | ~$0.005 / 1,000 | 5,000 | 52 a year, free |
| Class B ops (reads) | ~$0.0004 / 1,000 | 50,000 | one per page view |
| Egress | ~$0.12 / GB | 100 GB (N. America) | 200 KB per page view |

**Roughly $0.50–1.00 a year, nearly all of it compute.** The query and the
storage are rounding errors against their free tiers; the schedule is the only
line with a meaningful charge. Running daily instead of weekly multiplies that
line by about seven and still lands near $5 a year.

**The one line that scales with success is egress.** Every page view ships the
whole 200 KB file, data included. Ten thousand readers is 2 GB, inside the free
allowance; a hundred thousand is 20 GB, still inside it. A spike of a million
views is approximately 200 GB, so around $12 once. Embedding the data in the
page is what makes the hosting free, and also what makes a high-traffic day cost
money.

### Keeping it that way

Four controls, because a scheduled job bills you while you are not watching:

- **A $10 budget alert on the project**, set before anything else runs. It does
  not stop spending; it sends an email.
- **`--max-scan-mb`**, which the job passes to BigQuery as
  `maximum_bytes_billed`. A query that would scan more is refused by the server,
  so a bad edit to the SQL fails instead of billing.
- **A dry run on every execution.** Free, and the only opportunity to catch an
  expensive query before paying for it.
- **A floor under the dry run**, because a zero estimate means the ceiling is
  not protecting anything. See [Known pitfalls](#known-pitfalls).

One further control covers correctness rather than cost: the job validates the
finished payload before publishing — row count, plausible overall rate, no
segment returning more than it sold, nothing stale — and a failure exits
non-zero with the live page untouched. A schedule that silently publishes a
broken dashboard is worse than one that fails, because a job that reports
success is not investigated.

---

## Citation

Citation metadata is in [`CITATION.cff`](CITATION.cff), which GitHub renders as
a *Cite this repository* option in the sidebar.

To cite the *method* rather than this implementation, cite Kitagawa directly:

> Kitagawa, E. M. (1955). Components of a Difference Between Two Rates.
> *Journal of the American Statistical Association*, 50(272), 1168–1194.
> https://doi.org/10.1080/01621459.1955.10501299

---

## License

MIT. See [LICENSE](LICENSE).
