---
name: saleor-runtime-api-testing
description: Boot Saleor locally and verify runtime/API behavior end-to-end through the GraphQL API (golden-path commerce flows, staff auth/permissions, CRUD on staff-managed entities, introspection vs schema.graphql, log scans). Use when validating a runtime/dependency/Python-version change, or any change that must be proven against a live server rather than unit tests.
---

# Saleor runtime & GraphQL API testing

## Where to run

- **In a git worktree, prefer the worktree's own stack**: `.worktree-container/compose.sh up -d`, then
  `.worktree-container/compose.sh exec saleor <cmd>`. The host's Postgres/Valkey may belong to a
  different worktree, so host-side runs can hit the wrong DB.
- On a plain checkout, host-side runs are fine. `libmagic1` must be installed or plugin imports fail.

## Boot sequence (host)

```sh
(cd .devcontainer && docker compose up -d db cache)   # Postgres :5432, Valkey :6379
.venv/bin/python manage.py migrate                    # tables are NOT there after a fresh DB
.venv/bin/python manage.py populatedb
JWT_TTL_ACCESS="2 hours" .venv/bin/uvicorn saleor.asgi:application \
  --host 127.0.0.1 --port 8000 > /tmp/uvicorn.log 2>&1 &
```

`.env` needs `DATABASE_URL`, `CACHE_URL`, `SECRET_KEY`, `ALLOWED_HOSTS="localhost,127.0.0.1"`,
`DEFAULT_CHANNEL_SLUG=default-channel`, `HTTP_IP_FILTER_ALLOW_LOOPBACK_IPS=True`.

## Traps that cost real time

- **`populatedb` does NOT create a superuser** unless you pass `--createsuperuser`. Otherwise create
  one via the ORM (`admin@example.com` / `admin`).
- **`JWT_TTL_ACCESS` defaults to minutes.** A long GUI-driven session will expire mid-run; export
  `JWT_TTL_ACCESS="2 hours"` *before* starting uvicorn and mint the token after that.
- **Staff need unrestricted channel access.** A staff group with `restricted_access_to_channels=True`
  fails mutations with `"You don't have access to some objects' channel."`. Put the test user in a group
  with `restricted_access_to_channels=False` **and all permissions**.
- **`PositiveDecimal`/money literals must be numeric, not quoted.** `price:"12.50"` fails GraphQL
  validation (`Expected type "PositiveDecimal", found "12.50"`); use `price:12.50`.
- **`addressUpdate` requires `country`** even when you only intend to change one street field;
  omitting it returns `{field:"country", code:"REQUIRED"}` and does nothing.
- **`orderMarkAsPaid` is rejected for orders that already have transactions**
  (`Orders with transactions can not be manually marked as paid.`). To reach `FULLY_CHARGED`, use
  `transactionCreate` then `transactionUpdate(amountCharged: …)`.
- **Debug exception details are on by default**: GraphQL permission/validation errors include a Python
  traceback under `extensions.exception.stacktrace` in the *response body*. This is not a 500 — judge
  server health from the uvicorn log (status codes), not from the presence of a stacktrace in JSON.
  Query-document validation failures return **HTTP 400**, which is expected, not a crash.
- **Negative stock is accepted.** `StockInput.quantity` is a plain `graphene.Int` with no non-negative
  validation, so `productVariantStocksUpdate(quantity:-3)` returns `errors: []` and persists `-3`.
  Pre-existing product behavior — don't report it as a regression of whatever you're testing.

## Driving GraphiQL from the shell (browser-based testing)

GraphiQL is served at `/graphql/`. Paste the token into the **Headers** pane as
`{"Authorization":"Bearer <token>"}`. Helper scripts make this scriptable via xdotool:

```sh
# /tmp/gq.sh — type a single-line query into the query editor and run it
X=/opt/.devin/package/custom_binaries/xdotool; export DISPLAY=:0
$X mousemove 280 300 click 1; sleep 0.3; $X key ctrl+a; $X key Delete
$X type --delay 6 "$1"; sleep 0.5; $X key ctrl+Return; sleep 2.5
```

- The Headers pane sits lower on the page (~`280,589` at 1024x768). **Click it, `ctrl+a`, `Delete`
  to go anonymous; retype the full header JSON to go back to staff.** Always take a screenshot to
  confirm the header actually landed before trusting a "permission denied" or a successful mutation —
  a silently-empty Headers pane makes staff mutations look broken.
- Typing a ~700-char JWT with `xdotool type` takes >10 s; run it as a background shell and wait.

## CRUD test recipe (proves persistence, not just mutation payloads)

For every mutation: assert `errors: []` **and** the exact returned values, then **re-read the object
with a separate query**. Useful entity chain (each step gives you the id for the next):
`productTypeCreate` → `categoryCreate` → `productCreate` → `productVariantCreate` →
`productChannelListingUpdate` → `productVariantChannelListingUpdate` → `productVariantStocksUpdate`
→ `collectionCreate`/`collectionAddProducts`/`collectionRemoveProducts` →
`customerCreate`/`addressCreate`/`addressUpdate`/`addressDelete`/`customerUpdate`/`customerDelete` →
`updateMetadata`/`updatePrivateMetadata`/`deleteMetadata` → `productDelete` (cascades the variant) →
`productBulkDelete` → `categoryDelete`/`productTypeDelete`.

Negative coverage that actually catches things: unpublish via `productChannelListingUpdate` then
re-query **anonymously** (`totalCount` must drop to 0 while staff still sees 1); run writes with the
Authorization header removed and check the object was *not* changed; duplicate slug (`UNIQUE`),
negative price (`PositiveDecimal`), unknown id (`product(id:"UHJvZHVjdDotMQ==")` → `null`).

## Introspection vs committed schema

graphene 2 / graphql-core 2 serve the API, but you can compare with graphql-core 3 in a throwaway
venv: build the type/field/enum-value sets from live introspection and from
`saleor/graphql/schema.graphql`, then diff. Known **pre-existing** differences: ~10 arg/input default
values graphene 2 never reports (`format: ThumbnailFormatEnum = ORIGINAL`,
`PaymentInput.storePaymentMethod`). Reproduce them against `main` before blaming a branch.

## Final log scan

```sh
grep -cE 'Traceback|ImportError|ModuleNotFoundError' /tmp/uvicorn.log
grep -oE '" [0-9]{3} ' /tmp/uvicorn.log | sort | uniq -c     # expect only 200s (+400s for bad queries)
grep -E 'DeprecationWarning|RemovedIn' /tmp/uvicorn.log
```

Known pre-existing warning: `AuthlibDeprecationWarning` from `saleor/core/jwt_manager.py`.

## Devin Secrets Needed

None — everything runs against the local compose stack with the `.env` values above.
