---
name: Run a mediation A/B experiment
description: Create or adjust an AdMob mediation group and run an A/B experiment on it via v1beta — the
  write surface, on production inventory, with no sandbox and no idempotency key.
api: openapi/google-admob-api-v1beta-openapi.yml
operations:
  - accounts_list
  - accounts_mediationGroups_list
  - accounts_mediationGroups_create
  - accounts_mediationGroups_patch
  - accounts_mediationGroups_mediationAbExperiments_create
  - accounts_mediationGroups_mediationAbExperiments_stop
  - accounts_mediationReport_generate
generated: '2026-08-13'
method: generated
source: openapi/google-admob-api-v1beta-openapi.yml, conventions/google-admob-conventions.yml, sandbox/google-admob-sandbox.yml,
  errors/google-admob-problem-types.yml
---

# Run a mediation A/B experiment

**This skill writes to production.** AdMob has no sandbox: there is no test mode, no test
credentials and no test/live separation on `admob.googleapis.com`. Every create below makes a
real mediation group on a real publisher account that affects real ad serving and real
revenue.

**There is no idempotency key.** No `Idempotency-Key` header, no client request id, nothing in
the Discovery Document. A timed-out or retried create can and will produce a duplicate
mediation group or a duplicate experiment. Confirm with the human before every write, and
after any ambiguous failure **re-read the collection before retrying**.

## Before you start

- Base URL: `https://admob.googleapis.com`, channel **v1beta** — mediation groups and A/B
  experiments do not exist on v1.
- Auth: OAuth 2.0 user token. The scopes AdMob publishes are `admob.readonly` and
  `admob.report`; the consenting user must additionally hold an AdMob role permitting
  mediation changes, or writes return `403 PERMISSION_DENIED`.
- v1beta is a **beta** channel with no published deprecation policy and no Sunset header
  support. Pin nothing to it that you cannot change quickly.

## Steps

1. **Resolve the account** — `accounts_list` → `accounts/pub-XXXXXXXXXXXXXXXX`.

2. **List existing mediation groups first** — `accounts_mediationGroups_list`
   (`GET /v1beta/accounts/{accountsId}/mediationGroups`), using the `filter` parameter to
   narrow. Do this even when you intend to create: it is how you avoid creating a duplicate of
   something that already exists, and it is the only protection available.

3. **Create a mediation group** (only if one does not already exist) —
   `accounts_mediationGroups_create` (`POST /v1beta/accounts/{accountsId}/mediationGroups`)
   with a `MediationGroup`:
   - `displayName`
   - `targeting` (`MediationGroupTargeting`): `platform`, `format`, `adUnitIds`,
     `targetedRegionCodes` / `excludedRegionCodes`, `idfaTargeting`
   - `mediationGroupLines`: each `MediationGroupMediationGroupLine` names an `adSourceId`, a
     `cpmMode` and `cpmMicros` (**micros — millionths of the account currency**), its
     `adUnitMappings`, and a `state`
   Capture the returned `mediationGroupId` before doing anything else. If the call times out,
   re-run step 2 and look for the group by `displayName` rather than retrying the create.

4. **Or adjust an existing group** — `accounts_mediationGroups_patch`
   (`PATCH /v1beta/accounts/{accountsId}/mediationGroups/{mediationGroupsId}`). **`updateMask`
   is required in practice**: it names exactly which fields to change. Omitting it is the
   single most common way to overwrite `mediationGroupLines` or `targeting` you did not intend
   to touch. Patch is the one write here that is naturally idempotent — prefer it over
   create-then-delete patterns.

5. **Start the experiment** — `accounts_mediationGroups_mediationAbExperiments_create`
   (`POST /v1beta/accounts/{accountsId}/mediationGroups/{mediationGroupsId}/mediationAbExperiments`)
   with a `MediationAbExperiment`: `displayName`, `treatmentTrafficPercentage`,
   `controlMediationLines` and `treatmentMediationLines`. Record the returned `experimentId`.
   Check `MediationGroup.mediationAbExperimentState` before creating — a group already running
   an experiment should not get a second one.

6. **Measure it** — `accounts_mediationReport_generate`
   (`POST /v1beta/accounts/{accountsId}/mediationReport:generate`) with a
   `MediationReportSpec`. Metric values arrive as `ReportRowMetricValue`; revenue is
   `microsValue`, so divide by 1,000,000. Cap `maxReportRows` at 100,000. Reporting quota is a
   separate 900/minute pool from the inventory quota.

7. **Stop the experiment** — `accounts_mediationGroups_mediationAbExperiments_stop`
   (`POST .../mediationAbExperiments:stop`) with a `StopMediationAbExperimentRequest` carrying
   `variantChoice` — which arm to keep. Read `MediationAbExperiment.variantLeader` first and
   present it to the human; **choosing the losing arm is not reversible by re-running stop.**

## Hard rules

- Never create without listing first.
- Never retry a failed create blindly — re-read, then decide.
- Never patch without `updateMask`.
- Never call `:stop` without human confirmation of `variantChoice`.
- All CPM and revenue figures are micros. Treating them as currency units is off by 10^6.
