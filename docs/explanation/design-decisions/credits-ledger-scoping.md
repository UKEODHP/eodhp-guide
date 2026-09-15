---
title: Credits, ledger, and account system — implementation tasks
doc_status: unreviewed
tags:
  - workspaces
last_reviewed:
reviewed_by:
review_notes: "Migrated from the standalone accounting and billing repository; not yet reviewed in the guide."
---
# Credits, ledger, and account system — implementation tasks

This note breaks the credits work into tasks that can be picked up and finished one at a time. It replaces the earlier version of this document, which sized 15 business-level items before the design was settled. Those items are preserved in the [traceability table](#traceability-to-the-original-task-numbers), because the source spreadsheet and the other two design notes refer to them by number.

Read this with [Credits ledger design decisions](credits-ledger-design-decisions.md), which says why the design is shaped this way, and [Credits ledger schema](credits-ledger-schema.md), which defines the tables. Decisions are cited below as D1 to D12.

Scope is a workspace's own platform usage: compute, storage, object-store calls and transfer. The commercial-data purchasing rearchitecture is separate work.

## Findings from the current code

Four things in `accounting-service` change how the work should be sequenced. None was known when the earlier sizing was written.

**`workspace_authz` has two flags, and hub_admin does not override by default.** The function takes `require_owner` and `allow_hub_admin`, and `allow_hub_admin` defaults to `False` (`app/app.py:178`). D11 states that `hub_admin` overrides every tier. Making the override universal changes the behaviour of endpoints that exist now, so it is a decision to take before T1 rather than during it.

**The workspace admin tier is now real.** PR 53 on `eodhp-workspace-services` adds a `workspace_admins` table and lets admins manage members and linked accounts. D11 previously built an empty middle tier on the assumption that no admin concept existed. The tier is now populated, which turns T1 from scaffolding into working authorisation and adds T20. It also makes the question of which credit endpoints accept admins a live decision instead of a deferred one.

**The config loader runs in the ingester only.** `load_config_file` is called at `ingester/__main__.py:32`. Nothing in the API calls it. Two consequences for T4: the loader runs on every ingester pod start, so an unchanged config must not mint a policy version, and two ingester replicas can try to mint the same version at once. The unique constraint on `pricing_policy.version` catches the race, and the loader has to handle the conflict instead of failing the pod. `insert_configuration` now takes a session and does not commit, so T4 has the transaction control that needs.

T2 has since put a validated document model in front of this loader, so T4 compares typed frozen objects rather than dicts. It also removed two ways the loader failed quietly, both of which would have applied to a policy document as well: a misspelled top-level key loaded and configured nothing, and a repeated `prices` entry meant whichever entry was written last.

**An admin CLI already exists.** `dev/billing_admin.py` is a rich-click tool with `ls`, `set-price`, `add-item` and `update-item` commands. The schema note said the hub_admin operations needed "an internal admin page or a command-line utility"; this is that utility, so T15 to T18 have somewhere to land. Its `set-price` help text already says "credits per unit" while `BillingItemPrice.price` is documented as pounds. That ambiguity predates this work and T11 resolves it.

**Test and configuration plumbing has been reworked, and the conventions changed with it.** This was not on the task list and is now done. It matters to every task below, because it changes what a new table or query is written against:

| Was | Now |
|---|---|
| `settings = Settings()` and `engine = create_engine(...)` at import | `get_settings()` and `get_engine()`, cached, resolved on first use |
| Five `is_sqlite()` branches in `models.py` | None. PostgreSQL only, so DDL and SQL are written for it and nothing else |
| Tests on SQLite, migration tests skipped | 160 tests on a throwaway PostgreSQL container, one command. 119 of them need no database and run in under two seconds; the rest are in `tests/integration`, which is where the database fixtures now live, so the fast half cannot reach a container. The three migration tests were removed, and `make check-migrations` applies every revision to an empty container and runs `alembic check` against the result |
| `session.rollback()` at teardown, plus ten manual `delete(...)` cleanups | One transaction per test with savepoints, no cleanups |
| Ingester reaching for a module-global engine | `DBIngester(session_factory=...)` |

Two consequences for the tasks below. New tables need no driver guards. And a test that drives the ingester must pass `session_factory=db_session_factory`, or its writes land outside the test's transaction. See *How the tests get a database* in the [schema note](credits-ledger-schema.md).

## Wave 0 — foundations that unblock the rest

| ID | Task | Est | Status | Notes |
|---|---|---|---|---|
| T1 | Replace `require_owner` with ordered authorisation tiers | 0.5d | Done | A `MinTier` enum ordering member, admin, owner, with `hub_admin` above all three. The admin tier reads the `workspaces-admin` claim from T20 (D11) |
| T2 | Validate the config document with Pydantic | 1d | Done | `accounting_service/configuration.py` holds `ConfiguredItem` and a `Configuration` document model, and `load_configuration` validates the whole document before any of it is applied. Took about half the estimate, because `ConfiguredPrice` had already landed with the pricing rules. See *The configuration document* in the [schema note](credits-ledger-schema.md) |
| T20 | Publish workspace admin status as a JWT claim | 0.5d + review | Blocked | Cross-repo. A client scope in `eodhp-argocd-deployment/apps/keycloak/base/realms.yaml` using the same `oidc-api-claims-protocol-mapper` pattern as `workspaces-owned`, plus an entry in `apps/oauth2-proxy/base/values.yaml:6`. Depends on the inverse endpoint below |

T20 is all that is left in this wave, and it waits on another repository.

T1 came first because around eight new endpoints each name a minimum tier, so building them against the boolean flag would have meant rewriting all of them later.

T20 needs an endpoint that PR 53 does not include. The PR adds `GET /workspaces/{workspace-id}/admins`, which lists the admins of one workspace. A claim describes the user and is minted before any workspace is known, so the mapper needs the inverse — `GET /api/workspaces?admin`, mirroring the existing `?owned` filter and returning the same object shape. That request is with the PR author. Until it lands, T1 reads a claim that is absent and treats it as an empty list, which is the behaviour D11 originally specified, so T1 is not blocked.

## Wave 1 — the pricing policy

Everything downstream stores a `policy_id`, so the policy tables come before the ledger.

| ID | Task | Est | Status | Notes |
|---|---|---|---|---|
| T3 | Add the three policy tables and a migration | 1d | Done | `pricing_policy`, `pricing_policy_rate`, `pricing_policy_category_multiplier`, in `models.py`, with revision `9b12692d3f40`. `corrects_id` is a bare self-referential foreign key with no relationship attribute, because following it is an audit path rather than a read path. `version` is unique; monotonic is the loader's job in T4 |
| T4 | Mint-or-match policy loader | 1.5d | Done | Each load either matches the current policy or mints a new version. Matching compares every rate and every category multiplier together, against `Configuration` objects rather than dicts now that T2 validates the document first. `PolicyFingerprint` in `pricing.py` is the match rule; `PricingPolicy.load_configured_policy` carries it out, recovering from a lost version race inside a savepoint. `valid_until` is never written - see *The loader never closes a policy* in the [schema note](credits-ledger-schema.md) |
| T5 | Resolve the policy for a usage time | 0.5d | Done | Select the policy whose validity range contains the time, ordered by `configured_at` descending then `version` descending, limit 1. If no policy applies, the earliest `valid_from` prices it (D10). `PricingPolicy.resolve`, written when `GET /accounting/prices` needed it: a policy dated in the future must not be served as a current rate |

## Wave 2 — the pricing engine

| ID | Task | Est | Status | Notes |
|---|---|---|---|---|
| T6 | Add `workspace_category` and read the category from `workspace-settings` | 1d | Part done | The table, the read at pricing time and `billing-admin set-category` have landed, so a category can be assigned and a multiplier applied today. What remains is cross-repo: the Go producer does not send the field, so nothing populates the table automatically. The Python side works from the policy's `default_category` until the Go producer sends the field. That is D6's specified behaviour for an uncategorised workspace, so it is not a stopgap. Fix the `member_group`/`Owner` schema drift in the same pass. Adding the field and fixing the drift both change `eodhp-utils` and `eodhp-workspace-manager`, so only the table and the read-if-present can land here alone |
| T7 | The credit pricing function | 0.5d | Done | `price_usage` in `pricing.py`, over a `RateCard` — a policy's numbers projected over values, the counterpart to `PolicyFingerprint`. `PricingPolicy.rate_card()` makes the projection from a stored row. **Nothing is rounded**; see *Where rounding happens* in the [schema note](credits-ledger-schema.md). Resolving the category, including D6's fallback to the default, is part of pricing rather than of looking a rate up, so it lives on the rate card |

## Wave 3 — the ledger

| ID | Task | Est | Status | Notes |
|---|---|---|---|---|
| T8 | Add `credit_ledger_transaction`, the `transaction_type` enum, and the idempotency index | 1.5d | Done | Revision `0b4b174175ad`, with `credit_balance_snapshot` and `workspace_category` in the same revision. Two departures from the [schema note](credits-ledger-schema.md): `policy_id` and `category` are nullable, because a grant is not priced, with `ck_credit_ledger_transaction_debit_is_priced` stating the invariant that does hold; and the snapshot has a surrogate key, because PostgreSQL will not accept a nullable primary key column. Its uniqueness is two partial indexes rather than `UNIQUE NULLS NOT DISTINCT`, which is PostgreSQL 15 and failed on the deployed 14; the test container was pinned to 17 and so never saw it. The enum discipline was needed twice over - `pg_enum()` in `models.py` now passes `values_callable`, without which SQLAlchemy stores member *names*, and the column is declared `create_type=False` so `create_table` does not emit a second CREATE TYPE |
| T9 | Write debits from the ingester | 1d | Done | `AccountingIngesterMessager._charge_event`. The event and its debit commit in one transaction, so usage is never recorded uncharged. Three cases record the event and log at error level rather than failing the message: no policy covers the usage time, the policy holds no rate for the SKU, or the quantity is not a measurement. None is fixed by redelivery, and a stored quantity can be charged later (T18) where a dropped one cannot be recovered |

## Wave 4 — read paths

| ID | Task | Est | Status | Notes |
|---|---|---|---|---|
| T10 | Balance read and the snapshot table | 1.5d | Done | `GET /workspaces/{workspace}/accounting/balance`, any member. `CreditLedgerTransaction.balance` reads the latest snapshot plus every row recorded after it, and is correct with no snapshot at all - nothing writes one yet, so that is the path in use. Two statements rather than this note's single join, which returns no row for a workspace with no snapshot and one row per snapshot where only the latest is wanted. See *A snapshotter cannot cut at "now"* below |
| T11 | Policy read endpoints | 1d | Done | `GET /accounting/prices` serves credit rates from the policy in force, brought forward as part of removing fiat pricing: `price` became `credits_per_unit`, and `uuid` and `valid_until` are gone. `eodhp-workspace-ui` reads this through `InvoicesContext` and needs updating to match. `GET /accounting/pricing-policy` serves the whole rate card in force as one version - every rate, every multiplier and the default category. **Version history is not served**: it is an audit read rather than a product one, and `billing-admin ls <sku>` already covers it. The endpoint requires a token but asks nothing of its claims, through a `require_token` dependency now carried by every endpoint that has no workspace or account in its path |
| T12 | Usage reads with period, user and SKU filters | 1d | | `SUM(credits)` grouped by the requested dimension, with `HAVING SUM(credits) <> 0` so a fully reversed charge disappears instead of showing as a zero row (D12) |
| T13 | Explainable pricing endpoint | 0.5d | Done | `GET /workspaces/{workspace}/accounting/ledger/{transaction}`, any member. The row stores the quantity, the policy and the resolved category but not the rate or the multiplier, so the endpoint projects that policy into a rate card and prices again through `price_usage` - the same function that produced the charge, so the two cannot drift. The recomputed `charge` equalling the stored `credits` is the check on the whole scheme. `pricing` is null for a grant. A transaction in another workspace is a 404, not a 403 |
| T14 | Pre-execution cost estimate | 0.5d | | Runs T7's function against a proposed quantity and writes nothing. Advisory only (D4) |

### A snapshotter cannot cut at "now"

`recorded_at` defaults to `func.now()`, and in PostgreSQL `now()` is the *transaction*
timestamp. Every row written inside one transaction therefore carries the same `recorded_at`,
and no ordering exists between them.

This decides how T16 must write a snapshot. It cannot take its cut from the clock: rows it has
just summed may carry the very instant it would cut at, and the delta is `recorded_at > as_of`,
so those rows would be counted in neither half. It has to cut at the `recorded_at` of the
newest row it included.

The related hazard is not solved by either choice of clock. A transaction that writes a row and
commits later can leave a `recorded_at` earlier than a snapshot taken in between, so the row
falls outside the delta permanently. `clock_timestamp()` makes it worse rather than better,
because it moves the timestamp further from the commit. Whatever writes snapshots needs to
account for it, and until something does, the ledger is read in full and is always right.

### The display scale of a credit is undecided

Nothing is rounded when a charge is priced, which is deliberate (D8), so a charge carries the
scale of its inputs multiplied together: 3600.0 CPU-seconds at 0.001 credits with a multiplier
of 0.5 stores `1.80000`, and a balance takes the largest scale among the rows summed. The
schema note says read paths round for display, but no read path does, because no decision has
been taken about how many decimal places a credit has.

`GET /accounting/balance` therefore returns `"994.60000"` where a reader expects `"994.60"`.
This is cosmetic and it is a product decision rather than a schema one, but it will reach the
Credits page unless it is taken. One quantisation in `ExactDecimal`, or a credit-specific type
beside it, is the whole change.

## Wave 5 — administration and control

These extend `dev/billing_admin.py`. All are restricted to `hub_admin`.

**T21 gates this wave.** Waves 0 to 4 only read data, and a forged token already reads usage data today, so they add no exposure. Wave 5 introduces privileged writes, and those must not ship to production on an unverified token.

| ID | Task | Est | Notes |
|---|---|---|---|
| T21 | Verify JWT signatures | 1.5d | `decode_jwt_token` currently passes `verify_signature: False` and trusts whatever claims arrive (`app/authz.py`). See *Token verification* below |
| T15 | Credit grants | 1d | The request body is `{amount, reason}`. The frontend proposal offers `{amount}` alone, and a grant row carries `created_by` and `reason` for the audit log |
| T16 | Budget model and breach publication | 2d | Budgets keyed `(workspace, user-or-null)` (D5). A breach publishes a Pulsar message and blocks nothing (D4). This makes the service a Pulsar producer for the first time and needs a message schema added to `eodhp-utils`, so it touches a shared contract |
| T17 | Corrections and reversals | 2d | A reversal is a new row referencing the original, sharing a `correction_batch_id`. A reversal reuses the original policy version (D7) |
| T18 | Historical re-pricing runner | 1.5d | Reads quantity and category from each affected row, applies the corrected policy, and writes a reversal and a fresh charge in one batch (D8) |
| T19 | Audit log | 1.5d | Covers ledger writes and policy changes. Reversals are invisible in the usage endpoints (D12), so this is the only place a correction appears as an event |

### Token verification

`accounting-service` decodes bearer tokens without checking their signature, so it trusts the identity and the tier claims in any token it is handed. The assumption is that oauth2-proxy verifies upstream and that the service cannot be reached another way. That assumption has never been written down, and it now decides more than it used to: the same claims that select which workspace's usage you read will select who may set a spending limit (T16) and who may grant credits (T15).

There is a working pattern to copy in `wf-catalogue-service/src/wf_catalogue_service/api/auth/helpers.py`, a FastAPI service on the same platform. What T21 needs:

- `PyJWKClient` against the realm's JWKS endpoint, which matches the token's `kid` so key rotation needs no redeployment. The realm is `eodhp` and the issuer is `https://<platform-domain>/keycloak/realms/eodhp` (`apps/oauth2-proxy/base/values.yaml:5`).
- Verification of `exp` and `iss` as well as the signature, with a few seconds of leeway for clock skew.
- The correct `aud` list. This is the part most likely to cost time — see below.
- A decision on the failure mode. Verification makes Keycloak a dependency of any request arriving on a cold key cache. Failing closed is right for authorisation, but it means a Keycloak outage takes the API down, so the cache lifetime is a deliberate choice.
- A local-development path. The compose stack has no Keycloak, and unsigned tokens are what make local work possible today. `wf-catalogue-service` skips verification when its environment setting reads `local`. That is a switch which disables authentication, so it must default to off and be unreachable from production configuration — a sentinel that deployed config cannot produce, not a boolean.

Three details in that file are worth not copying. It builds `PyJWKClient` inside the decode function, so every request discards the cache and refetches JWKS. It does not check the issuer. And its audience list is `["oauth2-proxy-workspaces", "oauth2-proxy", "account"]`, none of which is a `clientId` in `realms.yaml` — the clients there are `eodh`, `eodh-workspaces`, `jupyter`, `sparkgeo`, `spyrosoft`, `oxidian`, `ades`, `argocd` and `kargo`, and the audience mappers inject `eodh` and `eodh-workspaces`.

So neither source can be trusted for the audience list. Read `aud` off a real token from each path that reaches this service — browser through oauth2-proxy, notebook, workflow. Getting it wrong fails closed, which is safe but presents as blanket 401s on whichever path was missed.

## Effort

Waves 0 to 5 total about 23 days. Done: T1 to T5, T7, T8, T9, T10, T11 and T13, which is 11 days, plus most of T6. This excludes the two tasks below that remain blocked, and matches the earlier estimate closely enough that the [ADR](accounting-billing-backend-adr.md) does not need revising.

Waves 3 and 4 came in close to their estimates. What they did not include is the demonstration surface: `billing-admin` gained `grant`, `set-category` and `ledger` so that the whole path can be driven from a terminal without the front end, and the walkthrough in the service's own `README.md` is the script for it. `grant` is not T15 - it has no endpoint and no authorisation, and T15 is still to do - but it is what makes a balance start above zero, without which a demonstration shows a workspace going into deficit on its first charge.

T21 accounts for 1.5 of those days and is hardening rather than a credit feature. It is counted here because it gates wave 5, but it would be defensible to fund it separately.

T20 is the only item that depends on another team's work. Its half day is small, but it crosses two repositories and a Keycloak realm change, so it wants lead time.

The largest single item is now 2 days. The earlier list had one 20-day item. Its size hid the risk inside it, which is why it was split.

## Work that is still blocked

**Periodic storage charging on a cycle, with proration.** This waits on the storage-billing decision. The mechanism it would extend already exists: `ConsumptionSampleRateIngesterMessager` turns rate samples into `BillingEvent`s in one-hour windows (`ingester/messager.py:83-140`). One caveat on that mechanism: generation is paced by arriving messages rather than by a clock, so it stalls when samples stop.

**Metering of shared and system resources.** It is still unconfirmed whether this reuses T7's engine against a different event source, which is cheap, or needs a flow that splits one cost across several workspaces, which is not. The answer decides the size.

## Excluded deliberately

**GPU metering.** No GPU SKU exists. `products-prices-config.yaml` defines `cpu-seconds`, `memory-gb-seconds`, `EFS-STORAGE-STD` and four `AWS-S3-*` items. The Credits page in the frontend design prices GPU and the budget page treats it as a cost, so `billing-collector` has to emit GPU consumption before anything here can charge for it.

**The breach-message consumer.** T16 publishes a message that nothing reads. D9 puts delivery out of scope, and the consuming subsystem is undesigned. Until it exists, warn-only enforcement has no visible effect on a user, which matters because the warning is the whole control.

**Payment and invoicing.** Credits are the unit of account and nothing converts them to money (D2). Buying credits is out of scope: a user asks a hub admin, who grants them (T15).

## Traceability to the original task numbers

The source spreadsheet and the other two design notes cite the original numbering. This table maps it to the tasks above.

| Original | Task | Now |
|---|---|---|
| 0 | Config and loader: consumption rate | T2, T3, T4 |
| 1 | Metered usage to credits, with category multiplier | T6, T7, T9 |
| 2a | Ledger schema and idempotent debit ingestion | T8, T9 |
| 2b | Balance maintenance and read API | T10 |
| 2c | Credit allocation and top-up | T15 |
| 3 | Pricing-policy version history API | T11 |
| 4 | Explainable-pricing endpoint | T13 |
| 5 | Shared and system-resource metering | Blocked |
| 6 | Periodic storage charging with proration | Blocked |
| 7 | Apply the policy in effect at time of usage | T5 |
| 8 | Store the resolved policy version per transaction | T8 (a column on the ledger table) |
| 9 | Audit log for ledger and policy changes | T19 |
| 10 | Pre-execution cost estimate | T14 |
| 11 | Budget and threshold model | T16 |
| 12 | Usage query filters | T12 |

Task 2b was sized for the risk of a mutable balance column under concurrent writes. An append-only ledger read by aggregation has no such column, so T10 carries none of that risk.

## What happened to the earlier gap findings

The earlier version of this document listed eight gaps. Six are now closed by a decision or a task:

| Gap | Outcome |
|---|---|
| Threshold enforcement had no task | D4 makes the breach message the whole control. T16 |
| No reversal or correction type | D7. T17 |
| No alerting on breach | D4 and D9. Published by T16, consumer out of scope |
| Versioning asymmetry between price and category | D6 pins the resolved category on every ledger row |
| The exchange rate had no consumer | Confirmed, and it never acquired one. Removed 2026-09-08 with `billing_item_price`; see the revision to D2 |
| Access control was not stated per endpoint | D11. T1 |
| `/accounting/skus` and `/accounting/prices` were anonymously readable | Settled 2026-09-15: every endpoint requires a token. See below |

Two remain open:

- **Reconciliation and backfill.** Deduplication stops double-charging but nothing detects a gap from ingester downtime, consumer lag or a dropped message. The `occurred_at` and `recorded_at` split exists to support this, and `budget_breach_notification` gives it somewhere to record a retrospective breach. The unmetered final hour after a resource is deleted (`ingester/messager.py:100-104`) is one instance of the same class.
- **The real-money funding boundary.** D2 defers it. T15 is an admin lever with no payment behind it.

## Open items

- Whether `hub_admin` should override every tier, including on endpoints that exist now. This gates T1. See the first finding above.
- Which credit endpoints admit the workspace admin tier. Budget configuration (T16) is the likeliest candidate, since PR 53 already gives admins linked-account management. One constant per endpoint, so it does not block T1.
- Whether PR 53 will carry the `GET /api/workspaces?admin` filter that T20 needs. If it does not, T20 waits on a follow-up in that repository.
- Whether `accounting-service` can be reached from inside the cluster without passing through oauth2-proxy. If it can, T21 is not hardening but a fix, and its priority changes.
- Which audience values appear in the tokens that reach this service. T21 cannot be finished without them, and they cannot be read reliably from either `realms.yaml` or the existing consumer.
- The storage-billing charging cycle and proration rules. This gates the first blocked task.
- Whether reading a balance and usage is open to any workspace member or to admins alone. Answered as `MinTier.MEMBER` for the balance and the single-transaction endpoints, matching the proposed endpoint table and the usage-data endpoints that predate this work. One constant per endpoint, so it can still be tightened per endpoint without touching the rest.
- Whether a caller should see every category's multiplier or only the one their workspace prices under. `/accounting/pricing-policy` serves them all today. If it narrows, `PricingPolicyAPIResult.of` is where the list is built, and the route's `Vary` has to gain `Authorization` in the same change. With the same stakeholders as the item above.
- How often a calibration pass runs. This affects how much tooling T4 deserves, not whether it is correct.
- **The remote test database still needs migrating, and this is an action rather than a question.** Its schema was created by `create_all` and then stamped at the baseline, so `alembic_version` claimed it was up to date while five columns were still naive. The chain is `20fef2107e45` → `7c3d5e9a1f42` → `9b4e2c81a7d3` → `9b12692d3f40` → `30ac7fce87ae`, and everything from `9b4e2c81a7d3` on is unapplied. Rehearse against a `pg_dump` copy before touching the real one. Note that `30ac7fce87ae` drops `billing_item_price` and discards its rows, which was accepted on the grounds that nothing consumes that data; its downgrade is deliberately empty, so recovery means restoring the dump.
