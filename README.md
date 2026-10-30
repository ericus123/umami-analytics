# Shoferi analytics

This repository hosts Umami. Browser tracking lives in the sibling `shoferiui` repository; authoritative reporting uses Sequelize models in `shoferi-api`.

## Agreed scope

- Keep learning metrics on existing progress records, per-user counters and `PlatformDailyStat` daily aggregates. There is no new learning-attempt table, no API endpoint for storing every practice answer, and no per-attempt retention or score-history reporting.
- Keep automatic first-observed acquisition attribution, one row per user.
- Keep historical payment duration/access-expiry facts, one snapshot on each existing payment record.
- Preserve UTC timestamps and the existing UTC daily-quiz reset. Existing Kigali reporting windows are for business reporting; they do not change users' quiz eligibility.

Leaving after passing an exam is expected for this product. Retention is not presented as a success metric requiring indefinite learner history.

## How acquisition is collected

The UI's `AcquisitionTracker` reads selected campaign parameters from the first landing URL. For example:

```
https://shoferi.com/rw?utm_source=instagram&utm_medium=paid_social&utm_campaign=november
```

This records `instagram`, `paid_social`, `november`, the landing pathname `/rw`, and language `rw`. Users fill in no form. Marketing links must include those tags if campaign-level attribution is wanted; the application cannot infer a campaign name from an ordinary untagged link.

Without tags, the tracker uses the referring hostname when the browser supplies one. Some apps, privacy settings and direct visits provide no external referrer, so attribution can be direct or unknown. The code does not claim that every visitor can be attributed.

The first observation is temporarily saved in browser local storage so it survives the signup journey. Once the user has an authenticated profile, the API stores the attribution once against that user's ID. A returning old account in a new browser is not reassigned a new acquisition source. Selected campaign values and the pathname are stored, not the complete URL query string, OTP, email or phone number. Account deletion removes attribution.

The collector is part of the local code changes, not evidence that production is already collecting this data.

## Current access versus purchase history

A user's existing `Subscription` row is found by their authenticated `userId`. It remains the source of truth for current access: the row must exist, be enabled, not be deleted and have an end date later than now. Renewals extend/update this row.

Each `Payment` already links a purchase to `userId` and `planId`, with amount, status and payment time. The added snapshots are:

- `durationInDays`: the number of access days that this purchase bought.
- `accessEndsAt`: the resulting access expiry after that payment was applied, including any access remaining from earlier purchases.

For example, a user with access until November 10 buys another seven days. Their payment snapshot records seven days and November 17; their current subscription row is extended to November 17. On a later renewal, the earlier payment snapshot remains November 17 while the current subscription moves forward.

This supports per-user paid-purchase history and repurchase reporting without keeping multiple active subscription rows. It is not a full audit of every manual grant, revocation or subscription edit.

## Reporting changes

- Signup-to-paid conversion counts unique positive-payment buyers within seven days of signup, divided by fully observed signup users. The overview pools users across cohorts rather than averaging daily percentages.
- Paid-purchase trends include renewals, using successful payment records instead of mutable subscription-row creation.
- Revenue uses actual collected RWF amounts. The former MRR/ARR presentation is labeled collected revenue over 30 days and an annualized run rate, not recurring commitments.
- Historical paid-subscriber counts use existing nightly snapshots; missing snapshots are omitted.
- Learning trends are labeled as newly mastered questions and learners mastering new questions. They do not claim to count every attempt or every active learner.
- The seven-day mock-exam card counts participants; the existing cumulative exam counter and daily platform snapshots remain in place.
- Churn/inactivity list totals count all matching users separately from the displayed page.
- New payment/acquisition reports show current payment outcomes by provider, first-time versus returning buyers, expiry-based repurchases, and acquisition-to-paid conversion.
- Browser events cover authentication, learning interactions, pricing visibility/selection, checkout opening/explicit abandonment, payment initiation, specific-payment confirmation/failure and waiting timeouts. These events are Umami telemetry, not a new learning-history table in the application database.

The repurchase report counts paid access periods with a complete expiry-plus-7/30-day observation window, including early renewals. Historical payments without recorded expiry are excluded rather than assigned invented dates.

## Database changes

Only these additions remain:

1. `user_acquisition`: one attribution row per user.
2. `Payment.durationInDays`: nullable integer.
3. `Payment.accessEndsAt`: nullable timestamp.
4. Payment reporting indexes.

These structures and indexes are declared in the Sequelize models. The API already registers models and conditionally synchronizes them when `ALTER_TABLES=true`. No new migration framework has been introduced.

The standalone SQL migration has been removed. The model declarations, including their field validators and indexes, are the source of truth for the existing Sequelize schema synchronization process. No separate SQL deployment step is required.

No database changes, production backfills or deployments have been executed. Existing timestamps are not converted or rewritten.

## Verification

From `shoferi-api`:

```sh
yarn test --runInBand
yarn build
```

The optional PostgreSQL regression suite requires an explicitly configured disposable database whose name begins with `shoferi_analytics_test`:

```sh
ANALYTICS_TEST_DATABASE_URL='postgres://localhost/shoferi_analytics_test' yarn test --runInBand src/analytics/analytics.postgres.spec.ts
```

It checks Sequelize `sync({ alter: true })` against an existing Payment table, preservation of legacy rows, unique buyers, weighted conversion and payment deadlines, provider outcomes and expiry-based repurchases. Without that variable it is skipped. The local sandbox blocked PostgreSQL shared-memory setup; these tests need a PostgreSQL-capable runner before deployment.

From `shoferiui`:

```sh
node --test scripts/tests/analytics.test.cjs scripts/tests/acquisition.test.cjs
yarn typecheck
```

The browser tests verify automatic campaign/referrer capture, direct-visit fallback, delayed tracker loading, URL query-string removal, duplicate-event suppression and the development gate. Model/service tests cover field lengths, supported languages, path-only landing pages, timezone-qualified observation timestamps, positive purchase durations, valid access expiries, legacy nullable values, and preservation of the purchased duration after plan edits. After staging deployment, check tagged-link signup attribution and a test-provider checkout against the corresponding admin reports.
