---
title: Credits and ledger — schema design
doc_status: unreviewed
tags:
  - workspaces
last_reviewed:
reviewed_by:
review_notes: "Migrated from the standalone accounting and billing repository; not yet reviewed in the guide."
---
# Credits and ledger — schema design

This note defines the database schema for the credits, ledger, and budget features in `accounting-service`. It implements the decisions in [Credits ledger design decisions](credits-ledger-design-decisions.md), which are referenced below as D1 to D12.

## Conventions inherited from the existing schema

New tables follow the patterns already in `accounting_service/models.py`:

- Tables are SQLModel classes, not SQLAlchemy declarative ones. There is no `Base`. A table is `class Thing(ThingBase, table=True)` with `__tablename__` set explicitly, and fields are declared as annotations with `Field` imported as `SQLModelField`. Primary keys are `uuid: UUID = SQLModelField(default_factory=uuid4, primary_key=True)`, and foreign keys are `item_id: UUID = SQLModelField(foreign_key="billing_item.uuid")`, naming the column as a string.
- Relationships are SQLModel's: `item: BillingItem = Relationship()`, with the join inferred from the foreign key beside it. Do not use `Mapped[...]` with a string forward reference. During the migration a `Mapped["BillingItem"]` failed to resolve once the target class left the old registry, and the failure surfaced only when a mapper was configured rather than when the DDL was built.
- Timestamps are declared with the `aware_timestamp()` factory in `models.py`, which sets `sa_type=TIMESTAMP(timezone=True)`. Never a bare `datetime` annotation: SQLModel maps that to TIMESTAMP WITHOUT TIME ZONE, which discards the offset on write. An `Annotated[datetime, ...]` type is also wrong, because both `| None` and an explicit `Field(...)` on the attribute drop the annotation's metadata and revert the column silently. Each timestamp that takes part in a comparison also needs a `*_utc` property returning `as_utc(self.field)`. PostgreSQL returns an aware datetime in the connection's timezone, which is not necessarily UTC. Use the `as_utc` helper rather than `astimezone` directly: on a naive value, which is what an object built in Python holds before any round trip, `astimezone` assumes local time and shifts by the local offset. That defect made a test fail during British Summer Time and pass in winter.
- Money-shaped and credit-shaped values are `Decimal`. Measured quantities stay `float`, matching `BillingEvent.quantity`.
- `workspace` is a plain string with **no foreign key**, following the reasoning in `BillingEvent`'s docstring: messages from the workspace service may arrive late or never, and losing a transaction is worse than holding a dangling reference.
- There is no driver branching. The service and its tests are PostgreSQL only, so DDL and SQL are written for PostgreSQL and nothing else. `models.py` used to carry five `is_sqlite()` branches, including two entirely separate implementations of the day and month aggregation; they are gone.
- Constrained value sets are PostgreSQL enums, reached through the same `sa_type=` route the timestamp factory uses: `SQLModelField(sa_type=sa.Enum(TransactionType))`. This covers `transaction_type`, `correction_batch.kind` and `budget_breach_notification.level`. The service is committed to PostgreSQL in production, and an enum rejects an invalid value at write time rather than at read time.
- Constraint names come from the naming convention on `SQLModel.metadata`, which `models.py` replaces at import, so new tables need no explicit names for indexes, foreign keys or primary keys. Check constraints are the exception and must be named — see *Constraint naming* below. Constraints and indexes go in `__table_args__` on the table class.
- `models.py` re-exports that metadata as a module-level `metadata`, and `alembic/env.py` imports it from there. This is deliberate and load-bearing: defining the table classes is what populates the metadata, so `target_metadata = SQLModel.metadata` in `env.py` left the import of `models` unused, an automated import cleanup removed it, and autogenerate then reported every table as removed. Do not replace that import with a direct reference to `SQLModel.metadata`.
- A table class and its API response share a base only where the two shapes are identical: `ThingBase(SQLModel)` holds the fields, `Thing(ThingBase, table=True)` is the table, and `ThingAPIResult(ThingBase)` is the response. Where a response deliberately differs from what is stored, it has no shared base and maps the difference itself. Two traps came out of doing this: a field the response redeclares must be redeclared without the base's `default_factory`, or it becomes optional in the OpenAPI schema, and a docstring on the response class is published as the endpoint's description.

One cost of SQLModel is worth knowing before you write a query. Pyright cannot check SQLModel query expressions: fields are plain annotations, not SQLAlchemy's `Mapped[...]`, so at class level the checker sees the Python value type instead of a SQL expression. `where(cls.valid_from <= at)` is reported as a bool where a `ColumnElement` is wanted, and constructing a row with `item=obj` looks like a missing `item_id`. All of it works and none of it is checkable. `models.py` carries a file-level suppression for exactly these rules, with the reasoning in its header. Put suppressions in the file that needs them rather than in `[tool.pyright]`, so a rule is never turned off for code nobody has looked at.

Three consequences of the enum choice affect the migrations rather than the schema:

- `sa.Enum(...)` emits a native `ENUM`, and the tests exercise that native type rather than a `VARCHAR` substitute, so an invalid value fails in a test the same way it would in production.
- Alembic does not autogenerate enum changes. A revision must call `sa.Enum(...).create(bind)` explicitly, and `drop()` on downgrade, or the type outlives the table and a downgrade-then-upgrade cycle fails.
- Adding a value is straightforward from PostgreSQL 12 on, though the new value cannot be used in the transaction that adds it. Removing or renaming a value needs a replacement type and a swap of every column that uses it, so the initial value sets are worth choosing deliberately.

## API response conventions

The conventions above govern what is stored. These govern what goes out, and both were
established by changes already made to the existing endpoints rather than by the credits work.

**Money and credits are exact decimal strings, not JSON numbers.** `Decimal` in the database
became `float` on the way out, which loses exactness on precisely the values where it matters.
`price` on `GET /accounting/prices` is now a string, produced by the `ExactDecimal` type in
`app/models.py`. New endpoints returning credits or money should use it.

Two details are deliberate. The serialisation never uses scientific notation: Pydantic's own
`Decimal` output gives `"4.12E-7"` for a small rate, which is correct and unhelpful, so it
goes through `format(value, "f")` and gives `"0.000000412"`. And it preserves the stored
scale, so a price of `0.10` stays `"0.10"` rather than collapsing to `"0.1"`, which matters
for anything displaying currency.

Measured quantities stay JSON numbers. `quantity` is a `float` in the database and a number in
the response, matching the storage convention.

**Timestamps are ISO-8601 in UTC, with sub-second precision where the value has it.** The
`UtcTimestamp` type carries an `AfterValidator` calling `as_utc`, so the value is guaranteed
UTC-aware before serialisation rather than being labelled `Z` on the way past. A naive value
would otherwise go out with no offset at all, which is a silently different format.

This replaced a hand-written serialiser that truncated to whole seconds. Billing event
timestamps carry microseconds, having come from `datetime.fromisoformat` on a Pulsar message,
so the truncation was discarding real precision.

## The configuration document

Items and their prices enter the system through one YAML document, which the ingester reads from `/etc/eodh/accounting.conf` on every pod start. `accounting_service/configuration.py` validates it whole before any of it is applied (T2). The policy tables in T3 extend the same document, and T4's loader reads it on the same schedule, so its rules bind the credits work too, not only the tables that exist now.

Four rules, each closing a way the old loader failed quietly:

- **An entry describes its subject completely.** All three fields of an `items` entry are required. Updating one field of a stored item is a thing the admin CLI does: `update-item` reads the stored row and fills in what the operator omitted, then sends a complete document. The loader used to accept a partial entry and update only the keys it found, which is what kept item entries from being validated at all. `BillingItem` is a SQLModel table class, and `table=True` turns Pydantic validation off, so `BillingItem(**item)` spread unchecked YAML into a row.
- **Unknown keys are rejected.** The production config lives inside a Kubernetes ConfigMap, under `data."accounting.conf"`. Mounting that file whole rather than its inner document gives a mapping with no `items` key, and the load used to succeed and configure nothing.
- **A subject appears once.** A repeated `items` SKU means the first entry is written and then overwritten. A repeated `prices` entry is worse, because applying the first makes the second an amendment of it, so the document means whichever entry was written last.
- **A bad document stops the ingester.** It loads the file before it starts consuming, so continuing would price events against a configuration the operator has already got wrong.

Validation cannot cover whether a price names a SKU that exists, because that needs a query. `upsert_configured_price` raises on an unknown SKU, and a document may legitimately introduce an item and its first price together.

`ConfiguredPrice` lives in `accounting_service/pricing.py`, away from the other document models, because the question a price entry raises — whether it amends, supersedes or appends — is a pricing rule. A policy entry will raise the same kind of question, and T4 answers it in the same place.

## Pricing policy

A policy is one calibration pass covering every rate at once (D3). Rows are immutable once written.

**`pricing_policy`**

| Column | Type | Notes |
|---|---|---|
| `uuid` | UUID | Primary key |
| `version` | int | Human-usable identifier. Monotonic, unique |
| `valid_from` | timestamptz | When this policy starts applying to usage |
| `valid_until` | timestamptz, null | Null on every policy the loader writes. See *The loader never closes a policy* below |
| `configured_at` | timestamptz | Defaults to `func.now()`. Decision time, as distinct from validity time |
| `corrects_id` | UUID, null | Self-reference. Set when this policy corrects an earlier one (D8) |
| `default_category` | str | Applied to workspaces with no assignment yet (D6) |
| `reason` | str, null | Why this calibration happened. Feeds the audit log |

**`pricing_policy_rate`** — one row per SKU per policy: `uuid`, `policy_id` → `pricing_policy`, `item_id` → `billing_item`, `credits_per_unit` (Decimal). Unique on `(policy_id, item_id)`.

**`pricing_policy_category_multiplier`** — one row per category per policy: `uuid`, `policy_id`, `category` (str), `multiplier` (Decimal). Unique on `(policy_id, category)`.

### The loader never closes a policy

T4 appends and nothing else. A stored policy keeps `valid_until` null for good, so every
validity range is open, and `PricingPolicy.resolve` reads the most recently configured policy
whose `valid_from` is at or before the usage time (T5). This is distinct from
`PricingPolicy.current`, which ignores validity and answers "what did we configure last" for
the loader: a policy dated next month is what we configured last, and prices nothing today. Ties on `configured_at` break
on `version` descending, because `configured_at` defaults to `func.now()` and that is the
transaction timestamp: two policies minted in one transaction carry the same value.

This is what makes a correction cheap. A backdated policy configured later wins over the
policy it corrects without rewriting a range it did not create, which is the append-only
property D8 needs. `valid_until` keeps a narrower meaning: a policy deliberately ended with
no replacement, which nothing does yet.

### Mint or match

The `pricing_policy` section of the configuration document holds one calibration pass:

```yaml
pricing_policy:
  valid_from: "2025-01-01T00:00:00Z"
  default_category: standard
  reason: "initial calibration"
  rates:
    - sku: cpu-seconds
      credits_per_unit: 0.5
  category_multipliers:
    - category: standard
      multiplier: 1
```

The loader runs on every ingester pod start, so leaving the stored policy alone is the common
case. `PolicyFingerprint` in `accounting_service/pricing.py` decides: it holds `valid_from`,
the default category, the rates and the multipliers, with rates and
multipliers sorted so document order does not matter and amounts compared as `Decimal` so
rewriting `0.5` as `0.50` is not a calibration. `reason` is deliberately outside it, so
re-wording the note explaining a policy mints nothing. `valid_from` is deliberately inside
it, so re-dating a calibration is recordable rather than a silent no-op.

`default_category` must have an entry in `category_multipliers`, or every workspace with no
category assignment is unpriceable (D6). Rejected at load rather than defaulted to 1, so the
number in force is a number somebody wrote down.

Two replicas starting together both see the same current policy and both mint the same
version. The unique constraint on `version` refuses the second; the write happens inside a
savepoint so the caller's transaction survives, and the loser re-reads. If the winner minted
what this document describes there is nothing left to do, which is the whole recovery.

`pricing_policy.rates` is the only place a rate is configured. A `prices:` section briefly
coexisted with it, feeding `billing_item_price` in pounds, and both are gone: credits are the
unit of account and there was no second number worth keeping in step.

The pair `(valid_from, configured_at)` makes the policy bi-temporal, which is what allows a corrected policy to be added for a period already charged without destroying the record of what was charged at the time. `BillingItemPrice` already describes this pattern in its docstring, so the concept is not new to this codebase — see *What this replaces*.

Resolving the policy for a given usage time selects the policy whose validity range contains that time, ordered by `configured_at` descending so a correcting policy wins over the policy it corrects. This mirrors the lookup `BillingItemPrice` documents. Usage predating every policy resolves to the earliest `valid_from` rather than failing (D10).

## Category assignment

**`workspace_category`** — `workspace` (str, primary key), `category` (str), `updated_at` (timestamptz), `updated_by` (UUID, null).

Populated from the `workspace-settings` Pulsar topic, which `accounting-service` already consumes (`ingester/__main__.py:37`). This is a separate table from `workspace_account` rather than a column on it, because `WorkspaceAccount.record_mapping` is insert-only by design and silently ignores changes — correct for the account mapping, wrong for a category that D6 requires to be changeable.

No history is kept here. History lives on the ledger, which records the category resolved at pricing time.

## Ledger

```python
class CreditLedgerTransaction(Base):
    """
    Append-only. Rows are never updated or deleted; a correction is a new row.

    A workspace's balance is SUM(credits), optionally filtered by item. Because
    rows are only ever inserted, concurrent debits never contend for a row and no
    locking is needed.
    """

    __tablename__ = "credit_ledger_transaction"

    uuid: Mapped[UUID] = mapped_column(Uuid, primary_key=True, default=uuid4)
    workspace: Mapped[str]
    user: Mapped[UUID | None]  # Denormalised from the billing event. Null for grants.
    transaction_type: Mapped[TransactionType]  # 'debit', 'grant' or 'reversal'
    credits: Mapped[Decimal]  # Signed: debits negative, grants positive.

    # Set for usage debits. Null for pool-wide grants.
    billing_event_id: Mapped[UUID | None] = mapped_column(ForeignKey(BillingEvent.uuid))
    item_id: Mapped[UUID | None] = mapped_column(ForeignKey(BillingItem.uuid))
    quantity: Mapped[float | None]  # Metered units, for explainability.

    # How this was priced: a reference, not a computed cost, so it can be replayed.
    # Both nullable as built - a grant is not priced. See 'A grant has no policy' below.
    policy_id: Mapped[UUID | None] = mapped_column(ForeignKey(PricingPolicy.uuid))
    category: Mapped[str | None]  # Resolved at pricing time; survives recategorisation.

    occurred_at: Mapped[datetime] = mapped_column(TIMESTAMP(timezone=True))
    recorded_at: Mapped[datetime] = mapped_column(
        TIMESTAMP(timezone=True), default=func.now()
    )

    # Corrections.
    reverses_id: Mapped[UUID | None] = mapped_column(
        ForeignKey("credit_ledger_transaction.uuid")
    )
    correction_batch_id: Mapped[UUID | None] = mapped_column(
        ForeignKey(CorrectionBatch.uuid)
    )
    created_by: Mapped[UUID | None]  # The hub_admin, for grants and reversals.
    reason: Mapped[str | None]
```

Four properties of this table carry most of the design.

**Signed credits.** `transaction_type` is metadata; the sign carries the arithmetic, so a balance is a `SUM` and a reversal is the negation of the row it reverses. Nothing needs to know the type to compute a balance correctly.

**`Decimal` credits against `float` quantity.** Quantity is a measurement and stays float, matching the existing column. Credits are exact. Pricing converts quantity to `Decimal` before multiplying by the rate and multiplier; multiplying a float by a `Decimal` directly raises in Python.

The conversion goes through `str`, not `Decimal(quantity)`. The latter converts the float's binary value exactly, so a measured 0.1 arrives as 0.1000000000000000055511151231257827021181583404541015625 and every charge derived from it carries that tail. `str` gives the shortest decimal that round trips to the same float, which is the number the collector reported. `exact_decimal` in `pricing.py` is that boundary.

### A grant has no policy

`policy_id` and `category` were specified NOT NULL and are nullable as built. A grant is not
priced: no policy applied to it and no category was resolved for it, so a NOT NULL column
would have to be filled with a policy that did not produce it, and every reader of an audit
trail would then have to know to disbelieve the field.

The invariant that does hold is a check constraint:

```sql
CONSTRAINT ck_credit_ledger_transaction_debit_is_priced CHECK (
    transaction_type <> 'debit' OR (policy_id IS NOT NULL AND category IS NOT NULL)
)
```

Stated as a rule about debits rather than as an equivalence between "is a grant" and "has no
policy". A reversal of a debit carries the original's policy version (D7), and a reversal of a
grant would carry none, so an equivalence would refuse the second.

The API shape follows from this rather than working around it: the single-transaction endpoint
nests the arithmetic in a `pricing` object which is null for a grant, so the response says
which fields are meaningful instead of leaving a reader to infer it from the type.

### Where rounding happens

Nowhere in pricing. `credits` is stored as the product comes out, and the read paths round for display — `ExactDecimal` in `app/models.py` already serialises a `Decimal` without losing scale or falling into exponent notation.

Two reasons, and the second is the one that decided it. Rounding per event rounds every event separately, so a great many small charges each lose their tail and the total drifts away from the quantities that produced it. And a stored charge is meant to be reproducible from the quantity, policy and category beside it (D8); if pricing rounds, a replay reproduces the rounding rule in force when the replay ran rather than the policy that was in force at the time.

So the `credits` column is an unconstrained `NUMERIC` with no precision or scale, which is arbitrary precision in PostgreSQL. Fixing a scale on it would be the same decision made in the schema instead of in the code.

**Both `occurred_at` and `recorded_at`.** Period filters want when the usage happened; audit and reconciliation want when the system learned about it. A back-filled event carries an old `occurred_at` and a recent `recorded_at`. Collapsing them into one column makes both queries wrong.

**`user` denormalised rather than joined.** `BillingEvent` already holds the user, so this column duplicates it. The duplication is worth it because per-user budgets and the per-user ledger filter both need it, and because a grant has no billing event to join through — under a join, grants would have no user at all. Debits copy the value at pricing time; grants leave it null, which marks them as pool-wide.

**Policy and category on every row.** Together with quantity, these make each row replayable: the charge can be recomputed from first principles months later, which is what the explainable-pricing endpoint, the audit log, and historical re-pricing all depend on.

### Idempotency

```sql
CREATE UNIQUE INDEX credit_ledger_original_debit_index
    ON credit_ledger_transaction (billing_event_id)
    WHERE correction_batch_id IS NULL AND transaction_type = 'debit';
```

This permits exactly one original debit per billing event while still allowing correction rows against that same event. A plain unique constraint on `billing_event_id` would block re-pricing entirely. The `WHERE` clause is load-bearing: PostgreSQL treats NULLs as distinct in unique constraints, so without it, correction rows would not be constrained at all and original rows would not be protected.

The tests run against PostgreSQL, so this index is exercised as written.

`BillingEvent.insert_from_message` already deduplicates on the incoming message UUID. This index protects a different failure: the same stored billing event being priced twice, for example after a crash between writing the event and writing the debit.

## Corrections

**`correction_batch`** — `uuid`, `created_at` (timestamptz, default now), `created_by` (UUID), `kind` (str: `metering` or `repricing`), `reason` (str), `policy_id` (UUID, null — set for a re-price, naming the corrected policy applied).

A metering correction reverses the affected debits and writes replacements, all sharing one `correction_batch_id`. Without the batch, a bug affecting thousands of rows becomes thousands of unlinked corrections with no way to answer what was fixed or when (D7).

A reversal reuses the **original** policy version, because fixing a wrong quantity is not re-pricing (D7). A re-price is the one case that deliberately applies a different policy, and it is identifiable by its batch `kind`.

Only a `hub_admin` may write to this table.

## Budgets and breach

**`workspace_budget`** — `uuid`, `workspace` (str), `user` (UUID, null), `limit_credits` (Decimal, the permitted negative balance), `warn_at_credits` (Decimal, where the warning fires), `alerts_enabled` (bool), `per_user_budgets_enabled` (bool, meaningful on the workspace row only), `updated_at`, `updated_by`. Unique on `(workspace, user)`.

One budget per workspace, with optional per-user thresholds drawing on the same pool (D5). The row with a null `user` is the workspace-wide limit; rows naming a user are overrides, and the workspace row's `per_user_budgets_enabled` switches the whole per-user mechanism off without deleting the overrides.

There is no `item_id`. Every SKU draws on one pool, so no budget is held per resource type and no grouping of SKUs is needed. Per-SKU reporting is unaffected, because it aggregates the ledger's `sku` directly.

Configured by the workspace owner. Per D11 the check names a minimum tier rather than a boolean. The `admin` tier now exists, courtesy of PR 53 on `eodhp-workspace-services`, so admitting admins here means changing one constant. Whether to do so is an open decision.

**`budget_breach_notification`** — `uuid`, `workspace`, `user` (UUID, null for a workspace-wide breach), `breached_at`, `published_at`, `level` (`warning` or `limit`), `balance_at_breach` (Decimal).

Append-only. It stops the same breach being published on every subsequent billing event, and it gives the audit log a record of what was published and when. Nothing consumes the published message yet (D9).

## Balance

The query above was specified as one join and is two statements as built: read the latest
snapshot, then sum the rows recorded after it.

```sql
-- The opening figure, or nothing.
SELECT balance, as_of FROM credit_balance_snapshot
WHERE workspace = :workspace AND "user" IS NULL
ORDER BY as_of DESC LIMIT 1;

-- The delta. Without a snapshot, the whole ledger, and still correct.
SELECT COALESCE(SUM(credits), 0) FROM credit_ledger_transaction
WHERE workspace = :workspace AND recorded_at > :as_of;
```

The single join does not survive contact with the data. `FROM credit_balance_snapshot` returns
no rows at all for a workspace nobody has snapshotted, so its balance comes back empty rather
than as the ledger sum, and where snapshots have accumulated it returns one row per snapshot
when only the latest is wanted.

**`credit_balance_snapshot`** — `uuid` (surrogate primary key), `workspace`, `user` (UUID, null for the whole-pool total), `as_of` (timestamptz), `balance` (Decimal). Unique on `(workspace, user, as_of)`, as two partial indexes.

The primary key was specified as `(workspace, user, as_of)`, which PostgreSQL will not accept:
a primary key column is NOT NULL, so the whole-pool row could not exist. Hence the surrogate
key and uniqueness declared separately.

Uniqueness has to account for the null user, because PostgreSQL counts two nulls as different
values and would let one workspace and instant be snapshotted twice. `UNIQUE NULLS NOT
DISTINCT` says exactly that in one line, and it is what this table declared until it reached a
deployed database. **It is PostgreSQL 15, and the deployed estate is 14**, where it is a syntax
error rather than a degradation - the whole revision fails to apply.

So uniqueness is two partial unique indexes instead, which say the same thing on either
version: `credit_balance_snapshot_user_unique` on `(workspace, user, as_of)` where the user is
not null, and `credit_balance_snapshot_pool_unique` on `(workspace, as_of)` where it is. The
pair must not over-constrain - a per-user snapshot and the whole-pool total belong at the same
cut - and `tests/integration/test_credit_ledger.py` asserts both halves refuse a duplicate and
that the two kinds of row coexist.

The general lesson outlived the specific one. The test container and `make check-migrations`
were pinned to `postgres:17` while the estate ran 14, so every check passed on a feature
production did not have. Both now default to the oldest deployed version, overridable with
`PG_IMAGE`.

Whatever writes a snapshot must take its `as_of` from the `recorded_at` of the newest row it
included, not from the clock. `recorded_at` defaults to `func.now()`, which is the transaction
timestamp, so rows written together share it and a cut at that instant would drop all of them
from the delta.

Snapshots are keyed the same way budgets are, because budgets are what need a fast balance read. The null-user row serves the workspace balance endpoint; per-user rows serve per-user threshold checks.

The snapshot is an optimisation, not a source of truth: the ledger alone always gives the right answer, and a snapshot can be rebuilt or discarded at any time. Snapshots must be keyed on `recorded_at` rather than `occurred_at`, or a backfilled event with an old `occurred_at` would be excluded from both the snapshot and the delta and vanish from the balance.

This is what makes task 2b unremarkable rather than risky. The original estimate assumed a mutable balance column needing lock discipline under concurrent debits. An append-only ledger read by aggregation has no such row and no such risk.

## Usage reads

The usage and ledger endpoints return net credits (D12). Netting needs no filtering logic, because it falls out of the aggregation:

```sql
SELECT date_trunc('day', occurred_at) AS period,
       "user",
       item_id,
       SUM(credits) AS credits
FROM credit_ledger_transaction
WHERE workspace = :workspace
  AND occurred_at >= :start
  AND occurred_at < :end
GROUP BY 1, 2, 3
HAVING SUM(credits) <> 0;
```

A reversal reduces the sum for the group it belongs to. The `HAVING` clause hides a fully reversed charge, which would otherwise appear as a zero row. The grouping columns vary with the requested period, user filter, and resource-type filter (task 12); the netting behaviour does not.

Individual reversal rows are therefore invisible here. The audit surface (task 9) is the only place a correction appears as an event in its own right.

## What this replaces

`pricing_policy_rate` replaced `billing_item_price`, and that table is gone as of revision `30ac7fce87ae`. Credits are the unit of account (D2) and nothing converts them to money, so there is no second number to keep in step.

`GET /accounting/prices` serves the rates of the policy in force. `eodhp-workspace-ui` consumes it through `InvoicesContext` (`getSKUPrice`, `getSKUUnit`), and the shape did change: `price` became `credits_per_unit`, renamed rather than redefined so a client displaying credits as pounds fails visibly rather than showing a wrong number; `uuid` and `valid_until` are gone, the first because a rate row's identity is an implementation detail and the second because the loader never closes a policy. `valid_from` kept its name but not its meaning - every SKU now reports the date of the calibration that set it, so they all share one date.

The policy loader appends and never updates a row that already exists. `upsert_configured_price` used to contradict that by mutating price rows in place; it went with the table.

## Migration approach

Alembic is in place. It owns the schema for the running application. `create_db_and_tables` and `drop_tables` have been removed from `db.py` altogether: the application never creates or drops a table, and a `drop_all` reachable from application code was a hazard with no caller. The tests build their own schema.

1. **Baseline.** One revision, generated against an empty PostgreSQL database, holding the schema as it stood before the credits work.
2. **New tables.** Later revisions add the tables above. No existing table is altered, which keeps each revision independently reversible.
3. **Single runner.** `accounting-service` runs two API replicas and two ingester replicas (`api-deployment.yaml:6`, `ingester-deployment.yaml:6`). Four pods each calling `alembic upgrade head` at startup will contend. Migrations belong in a dedicated Job — an ArgoCD PreSync hook, following the Job pattern already used elsewhere in the deployment repository — with the application pods failing fast if the schema version is not what they expect. Locally this is a one-shot `migrate` service that both long-running services wait on, so the same ordering is exercised.
4. **Tests build the schema with `create_all`. Nothing tests the migrations.** `uv run pytest` needs only Docker; see *How the tests get a database* below.

   There were three migration tests and they have been removed deliberately: migrations are applied by hand and reviewed before they reach a remote database. Two of the three duplicated what `alembic check` reports. The third did not — see *Two expression indexes are managed by hand* — so verifying those two indexes is now a manual step, and the reasoning sits in a comment on `UNCOMPARED_INDEXES` in `alembic/env.py`.

   An earlier draft of this note said the tests should build their schema by running the migrations instead, and recorded that as unworkable because the expression indexes have no SQLite form. That objection was entirely about SQLite and no longer applies. Doing it would make the tests run against the schema production has, rather than one `create_all` derives from the same models, and would close the gap the removed tests covered. It is available and not done.

### The deployed databases are migrated, not recreated

Settled and done, recorded here because it constrains what later revisions may assume.

The deployed databases keep their accumulated `billing_event` rows rather than being recreated from empty. Their tables were originally created by `create_all`, so they carried the constraint names PostgreSQL invents rather than the ones the naming convention gives, and Alembic matches constraints by name. Revision `7c3d5e9a1f42` renames them. It reads each current name from `pg_constraint` instead of assuming PostgreSQL's default, so it is safe on a legacy database, on one created by the baseline, and on a second run.

Both remote databases have been stamped with the baseline and migrated. Later revisions can therefore assume convention-shaped constraint names everywhere.

### How the tests get a database

`tests/conftest.py` starts a throwaway PostgreSQL container for the test session, using `testcontainers`. `uv run pytest` needs nothing but Docker, and the shared CI workflow (`EO-DataHub/github-actions/.github/workflows/unit-tests-python-uv.yaml`) needs no change, because the session brings its own database rather than expecting one to be provided.

Three properties of this arrangement matter for the credits work.

**PostgreSQL, not SQLite.** The tests used to run on SQLite, which cost five driver branches in `models.py` and meant the day and month aggregation tests exercised SQLite expressions rather than the `date_trunc` ones that run in production. The two expression indexes did not exist under SQLite at all. Both are now covered.

**A container, not a shared instance.** The suite creates the schema, so it must not be able to reach anything real. An earlier version pointed at a shared server and guarded it by looking for `test` in the database name, which is a heuristic. The container removes the question: the tests never learn how to reach a real database.

**Each test runs inside one transaction that is rolled back.** `db_connection` opens a transaction; `db_session` and `db_session_factory` join it with `join_transaction_mode="create_savepoint"`, which is SQLAlchemy's documented recipe for test suites. Code under test can call `commit()` freely and the outer rollback still discards it.

That last point matters more than it looks. The previous fixture called `session.rollback()` at teardown, which does nothing after a commit, and the ingester committed on its own sessions outside the fixture's transaction entirely. Ten tests began by deleting rows their predecessors had left behind. Those cleanups are gone.

New tests that drive the ingester must pass `session_factory=db_session_factory` when constructing a messager. Without it the ingester opens sessions on the process-wide engine, outside the test's transaction, and its writes survive the test.

### Two expression indexes are managed by hand

`billingevent_day_aggregate_index` and `billingevent_month_aggregate_index` are excluded from autogenerate comparison, listed in `UNCOMPARED_INDEXES` in `alembic/env.py`. PostgreSQL normalises their expressions when storing them — `date_trunc('day', event_start AT TIME ZONE 'UTC')` comes back with explicit casts and parentheses — and Alembic compares the two as strings, so they would otherwise appear changed in perpetuity and every generated revision would drop and recreate them.

The exclusion has a sharp edge: `include_object` governs rendering as well as comparison, so these indexes are also left out of generated revisions. `alembic check` therefore reports clean when they are missing. It is not that the check is unreliable in general; it cannot see these two at all, because the hook hides them from it.

That has already happened once. A regenerated baseline dropped both indexes, `alembic check` reported no changes, and it was caught only by reading the revision file and noticing there was no `date_trunc` in it.

So they are written by hand in the baseline. If you regenerate it, re-add them by hand and confirm by reading the SQL rather than by running `alembic check`. Anything added to `UNCOMPARED_INDEXES` inherits the same problem. There is no test covering this: the one that did has been removed, so it is a review step.

### Constraint naming

`SQLModel.metadata` carries a naming convention, so indexes, unique constraints, check constraints, foreign keys and primary keys all get deterministic names. Check constraints must be named in the model, because Alembic matches them by name and an anonymous one can never be matched against the name PostgreSQL invents. Give the bare name only — `name="start_before_end"` — since the convention adds the `ck_<table>_` prefix; including the prefix yourself produces a stuttering `ck_billing_event_ck_billing_event_start_before_end`.

One trap for a multi-column unique constraint. The convention names a unique constraint after its first column only, so `UniqueConstraint("policy_id", "item_id")` becomes `uq_pricing_policy_rate_policy_id`, which reads as though it constrained `policy_id` alone. That is unique within its table and works, and T3 left both policy constraints named this way for consistency. A table needing two multi-column unique constraints that begin with the same column would collide, and would need explicit names.

## Frontend alignment

Reconciled against the frontend endpoint proposal in `pending-backend-endpoints.md`.

What already fits: the proposed `/api/workspaces/:id/accounting/*` paths sit in a namespace `accounting-service` already serves (`app/app.py:217`, `:319`), and the "any member", "owner", and "`hub_admin` only" tiers all map onto `workspace_authz` as it stands. The proposal's fourth tier, workspace admin, is being built in PR 53 on `eodhp-workspace-services` and reaches this service as a JWT claim (D11). `GET /api/accounting/pricing-policy` is D3's bundled policy, and `PUT /api/workspaces/:id/category` restricted to `hub_admin` is D6's write side, landing in `eodhp-workspace-services` and reaching this service over Pulsar.

One divergence as built. The proposal marks `GET /api/accounting/pricing-policy` as auth "Any", and it requires a token - any valid one, with no claim examined. That was settled on 2026-09-15 for every endpoint this service serves, `/accounting/skus` and `/accounting/prices` included: "Any" now means any signed-in user rather than anonymous. Nothing here is anonymously readable, and `test_no_endpoint_is_readable_without_a_token` walks the routes so a new one cannot quietly be.

Three things the schema cannot supply on its own:

**No GPU SKU exists.** `products-prices-config.yaml` defines `cpu-seconds`, `memory-gb-seconds`, `EFS-STORAGE-STD` and four `AWS-S3-*` items. The Credits page prices GPU and the budget page treats it as a cost, so `billing-collector` must emit GPU consumption first. This is upstream of everything here.

**The per-user alerts toggle has no consumer.** The proposal offers users an `alertsEnabled` switch, while D9 leaves notification delivery out of scope and undesigned. The toggle either ships inert or the consumer comes into scope.

**Grants need a reason.** The proposed top-up body is `{ amount }`. Grant rows carry `created_by` and `reason` for the audit log, so the body should be `{ amount, reason }`, with reason required for any `hub_admin` action.

Corrections and re-pricing (D7, D8) have no proposed endpoint. Both are `hub_admin`-only, so a user-facing UI is not the natural home, but the tooling needs one — an internal admin page or a command-line utility.

## Open items

- The storage-billing decision still gates the charging cycle for storage SKUs. The schema does not depend on it: storage debits are ordinary rows whose `occurred_at` spans the charged period.
- Reconciliation and backfill remain undesigned. `recorded_at` and the `occurred_at`/`recorded_at` split exist to support it, and `budget_breach_notification` gives it somewhere to record retrospective breaches, but nothing detects a gap yet.
- Who may read a workspace's balance and usage. The frontend proposal's tables say "any member" for both, but its own open questions still ask whether usage should be restricted to admins. The tier mechanism in D11 makes this a one-constant change per endpoint, so it does not block implementation.
- Whether a caller should see every category multiplier or only the one their own workspace prices under. Tracked in the [scoping note](credits-ledger-scoping.md).
