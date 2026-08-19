---
name: Audit AdMob apps and ad units
description: Walk a publisher's AdMob inventory — apps, ad units and the ad-unit-to-adapter mappings behind
  mediation — using only read operations, paging correctly and staying inside the tight inventory quota.
api: openapi/google-admob-api-v1beta-openapi.yml
operations:
  - accounts_list
  - accounts_apps_list
  - accounts_adUnits_list
  - accounts_adSources_list
  - accounts_adSources_adapters_list
  - accounts_adUnits_adUnitMappings_list
generated: '2026-08-13'
method: generated
source: openapi/google-admob-api-v1beta-openapi.yml, data-model/google-admob-data-model.yml, conventions/google-admob-conventions.yml,
  rate-limits/google-admob-rate-limits.yml
---

# Audit AdMob apps and ad units

Read-only inventory walk. Uses **v1beta** because ad sources, adapters and ad unit mappings
do not exist on v1. Every operation below is a `GET`; this skill never mutates anything.

## Before you start

- Base URL: `https://admob.googleapis.com`
- Auth: OAuth 2.0 user token, scope `https://www.googleapis.com/auth/admob.readonly`.
- **Inventory quota is the tightest pool on the API: 120 read requests per minute and
  172,800 per day, per Google Cloud project.** Every call in this skill except
  `accounts_list` counts against it. Plan the walk before making it.

## Steps

1. **Resolve the account** — `accounts_list` (`GET /v1beta/accounts`) → `account[].name` =
   `accounts/pub-XXXXXXXXXXXXXXXX`. This is the account-class quota pool (900/min), not the
   inventory pool.

2. **List apps** — `accounts_apps_list` (`GET /v1beta/accounts/{accountsId}/apps`). Each `App`
   carries `appId`, `platform`, `appApprovalState`, and either `linkedAppInfo` (linked to a
   store listing: `appStoreId`, `displayName`) or `manualAppInfo` (`displayName` only).
   `appApprovalState` is the field worth auditing — an unapproved app does not serve.

3. **List ad units** — `accounts_adUnits_list` (`GET /v1beta/accounts/{accountsId}/adUnits`).
   Each `AdUnit` carries `adUnitId`, `adFormat`, `adTypes`, `displayName`, optional
   `rewardSettings`, and **`appId`, which is the join back to the app**. Group ad units by
   `appId` to build the inventory tree; there is no nested list endpoint.

4. **Page every list properly.** All list methods use Google's standard paging: send
   `pageSize` and `pageToken`, read `nextPageToken`, repeat until it is absent. Do not guess a
   large `pageSize` to save calls — a rejected page costs you a quota unit anyway.

5. **Resolve mediation adapters (only if you need the mediation view).**
   - `accounts_adSources_list` (`GET /v1beta/accounts/{accountsId}/adSources`) → the catalog of
     networks (`adSourceId`, `title`).
   - `accounts_adSources_adapters_list`
     (`GET /v1beta/accounts/{accountsId}/adSources/{adSourcesId}/adapters`) → per-network
     `Adapter` records with `adapterId`, `formats`, `platform` and the
     `adapterConfigMetadata` fields that adapter requires.
   This is one call **per ad source**. With a wide mediation setup that is dozens of calls
   against a 120/minute pool — fetch it once and cache it, do not re-walk it per ad unit.

6. **Resolve ad unit mappings** — `accounts_adUnits_adUnitMappings_list`
   (`GET /v1beta/accounts/{accountsId}/adUnits/{adUnitsId}/adUnitMappings`), one call per ad
   unit. Each `AdUnitMapping` binds an AdMob ad unit to one adapter (`adapterId`) with the
   third-party network's own configuration (`adUnitConfigurations`) and a `state`. This is the
   step that will exhaust your quota on a large account — scope it to the ad units you
   actually care about. Both this method and `accounts_mediationGroups_list` accept a `filter`
   query parameter, so filter server-side rather than listing everything and discarding.

## What a good audit reports

- Apps whose `appApprovalState` is not approved.
- Ad units with no `AdUnitMapping` at all — inventory that mediation cannot fill.
- Mappings whose `state` is not active.
- Ad units whose `adFormat`/`adTypes` do not match any adapter `formats` on their mapped
  adapter.
- Apps present in AdMob with zero ad units.

## Failure handling

- `403 PERMISSION_DENIED` "not accessible to the effective user" means the consenting Google
  account lacks the AdMob role for that resource — re-consent as the right user, do not retry.
- `429 RESOURCE_EXHAUSTED` naming the inventory quota group means stop and wait. There is no
  `Retry-After` header; back off client-side and resume from the last `nextPageToken` you
  successfully consumed.
