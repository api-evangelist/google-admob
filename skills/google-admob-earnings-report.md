---
name: Pull an AdMob earnings report
description: Authenticate against the AdMob API, resolve the publisher account, and generate a network
  report of earnings by date and app — handling the microsValue money format, the 100,000-row ceiling and
  the per-project quota correctly.
api: openapi/google-admob-api-v1-openapi.yml
operations:
  - accounts_list
  - accounts_get
  - accounts_networkReport_generate
generated: '2026-08-13'
method: generated
source: openapi/google-admob-api-v1-openapi.yml, conventions/google-admob-conventions.yml, errors/google-admob-problem-types.yml,
  rate-limits/google-admob-rate-limits.yml
---

# Pull an AdMob earnings report

Use the stable **v1** channel. It is read-only, so nothing in this skill can mutate the
publisher's account.

## Before you start

- Base URL: `https://admob.googleapis.com`
- Auth: OAuth 2.0 **user** access token in `Authorization: Bearer <token>`. AdMob does not
  support service accounts — a human must have consented. If you do not hold a token, stop
  and ask; do not attempt any other credential type.
- Scope needed: `https://www.googleapis.com/auth/admob.report`
- The AdMob API must be enabled on the Google Cloud project behind the OAuth client, or
  every call returns `403 PERMISSION_DENIED`.

## Steps

1. **Resolve the publisher account** — call `accounts_list` (`GET /v1/accounts`). It returns the
   AdMob publisher account the authorizing user most recently signed into, as
   `account[].name` = `accounts/pub-XXXXXXXXXXXXXXXX`. Never hard-code a publisher id; the
   literal string `accounts/pub-XXXXXXXXXXXXXXXX` in the docs is a placeholder and sending it
   produces `400 INVALID_ARGUMENT: Invalid account information in request url`.

2. **Read the account's reporting context** — call `accounts_get`
   (`GET /v1/accounts/{accountsId}`) and keep two fields: `currencyCode` and
   `reportingTimeZone`. Every money figure in the report is denominated in that currency, and
   every date boundary is in that timezone. Reporting them without the currency is a
   correctness bug, not a formatting one.

3. **Generate the network report** — call `accounts_networkReport_generate`
   (`POST /v1/accounts/{accountsId}/networkReport:generate`) with a `NetworkReportSpec`:
   - `dateRange` — a `startDate`/`endDate` pair of `Date` objects (`year`, `month`, `day`).
   - `dimensions` — e.g. `DATE`, `APP`, `COUNTRY`, `PLATFORM`, `FORMAT`.
   - `metrics` — e.g. `ESTIMATED_EARNINGS`, `IMPRESSIONS`, `CLICKS`, `AD_REQUESTS`,
     `MATCH_RATE`, `IMPRESSION_CTR`.
   - `maxReportRows` — **1 to 100,000**. Anything higher fails with
     `400 INVALID_ARGUMENT: Cannot request more than 100,000 rows in one report request.`
   - `dimensionFilters`, `sortConditions`, `localizationSettings`, `timeZone` as needed.
   Only use dimension/metric combinations the AdMob reference marks compatible — an invalid
   pair returns `400 INVALID_ARGUMENT: Requested metrics and dimensions are incompatible.`

4. **Read the response correctly.** The response is a stream of objects: a `header`
   (`ReportHeader`), then `row` entries (`ReportRow`), then a `footer` (`ReportFooter` with
   `matchingRowCount` and any `warnings`). Each row has `dimensionValues` and `metricValues`.
   **`ReportRowMetricValue` is a oneof**: `integerValue`, `doubleValue`, or `microsValue`.
   `ESTIMATED_EARNINGS` arrives as **`microsValue` — millionths of the account currency**.
   Divide by 1,000,000. Reading `doubleValue` for earnings silently yields nothing.

5. **Check the footer warnings** before presenting numbers. AdMob reports recent days as
   estimates and flags data issues in `footer.warnings[]` (`ReportWarning.type`,
   `.description`). Surface those warnings to the user rather than swallowing them.

## Quota and failure handling

- Reporting quota is **900 read requests per minute per Google Cloud project**; account reads
  are a separate 900/minute pool. See `rate-limits/google-admob-rate-limits.yml`.
- On exhaustion you get `429` with `RESOURCE_EXHAUSTED` and a message naming the quota group
  (e.g. `ReportingReadGroup`). **There is no `Retry-After` header and no `RateLimit-*`
  header** — back off on your own schedule, exponentially, and narrow the date range or drop
  dimensions rather than retrying the same oversized request.
- Errors arrive as `{"error": {"code", "message", "status", "details"}}` — Google's
  `google.rpc.Status`, not RFC 9457 problem+json. Match on `error.status`, not on the message
  text.

## Do not

- Do not page a report with `pageToken`. Reports are not paged; bound them with `dateRange`
  and `maxReportRows`.
- Do not assume the report currency is USD.
- Do not fall back to the v1beta channel for earnings — v1 covers network and mediation
  reporting, and v1beta additionally exposes writes you do not need here.
