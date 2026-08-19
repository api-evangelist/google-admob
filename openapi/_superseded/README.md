# Superseded scaffold OpenAPIs

These files were retired on 2026-08-13 by the API Evangelist enrichment pipeline.

`google-admob-scaffold-original-openapi.yml` was a hand-written 6-operation scaffold, not a
Google-published contract. The five tag-split files beside it were refined from it. The
scaffold named real AdMob capabilities but got the contract wrong in ways that matter:

- It placed `createAdUnit` on `/v1/{parent}/adUnits`. Ad unit creation exists only on the
  **v1beta** channel; the stable v1 channel is entirely read-only.
- It placed `listMediationGroups` on v1. Mediation groups are v1beta-only.
- It omitted `accounts.get` and `accounts.mediationReport:generate`, which are real v1
  operations.
- It carried 5 schemas against the 28 (v1) / 49 (v1beta) Google actually publishes.

They are superseded by contracts converted from Google's own first-party Discovery Documents,
harvested the same day:

- `../google-admob-api-v1-openapi.yml` — 6 operations, 29 schemas
- `../google-admob-api-v1beta-openapi.yml` — 19 operations, 50 schemas
- `../../discovery/google-admob-api-v1.json` — verbatim source, revision 20260731
- `../../discovery/google-admob-api-v1beta.json` — verbatim source, revision 20260731

Kept for provenance and diffing. Nothing in `apis.yml` points at them.
