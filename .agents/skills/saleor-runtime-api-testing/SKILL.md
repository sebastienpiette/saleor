---
name: saleor-runtime-api-testing
description: Boot Saleor on the host and verify real runtime/API behaviour (GraphQL playground, staff auth, checkout→order golden path, live introspection vs schema.graphql). Use when a change must be proven end-to-end at runtime — runtime/dependency/Python-version migrations, GraphQL schema regeneration, auth/permission or checkout changes — rather than by unit tests.
---

# Saleor runtime / API testing

## 1. Bring up services and the app

In a **git worktree** (see AGENTS.md), never touch the `.devcontainer` stack or the host `.venv` —
they may belong to another checkout on the same fixed ports. Use the per-worktree stack instead, run
from the worktree root:

```sh
.worktree-container/compose.sh up -d
.worktree-container/compose.sh exec saleor python manage.py migrate
.worktree-container/compose.sh exec saleor python manage.py populatedb
# the saleor service publishes no port, so serve through an explicitly published one:
.worktree-container/compose.sh run --rm -p 8000:8000 -e JWT_TTL_ACCESS="2 hours" saleor \
  uvicorn saleor.asgi:application --host 0.0.0.0 --port 8000
```

Pick a free host port per worktree (`-p 8001:8000`, …) and use it everywhere `8000` appears below.
The stack's other services publish ephemeral host ports, so resolve those before connecting from the
host: `.worktree-container/compose.sh port db 5432`, `... port cache 6379`.

Only in a **single, non-worktree checkout** is the host recipe safe (run it from the repo root):

```sh
(cd .devcontainer && docker compose up -d db cache)   # Postgres 5432, Valkey 6379
.venv/bin/python manage.py migrate
.venv/bin/python manage.py populatedb                 # seeds channels/products; takes a few minutes
JWT_TTL_ACCESS="2 hours" .venv/bin/uvicorn saleor.asgi:application --host 127.0.0.1 --port 8000 \
  > /tmp/uvicorn.log 2>&1 &
```

That recipe's `.env` needs `DATABASE_URL=postgres://saleor:saleor@localhost:5432/saleor`,
`CACHE_URL=redis://localhost:6379/0`, `SECRET_KEY`, `ALLOWED_HOSTS="localhost,127.0.0.1"`,
`DEFAULT_CHANNEL_SLUG=default-channel`, `HTTP_IP_FILTER_ALLOW_LOOPBACK_IPS=True`.
`libmagic1` must be installed or plugin imports fail at startup.

Gotchas learned the hard way:
- `populatedb` does NOT create a superuser unless you pass `--createsuperuser`. Otherwise create one
  via `manage.py shell` (set `is_staff`, `is_superuser`, `is_active`, `is_confirmed`, password).
- Default `JWT_TTL_ACCESS` is **5 minutes** — always start uvicorn with a longer TTL for manual
  browser testing, or tokens expire mid-session.
- A seeded/created admin can still hit `"You don't have access to some objects' channel."` on order
  mutations. Fix by adding the user to a group with `restricted_access_to_channels=False` and all
  permissions.
- The startup telemetry log line reports the interpreter version — good evidence of which Python is
  actually serving.

## 2. Browser GraphQL testing (GraphiQL at /graphql/)

`PLAYGROUND_ENABLED` defaults True, so `GET http://127.0.0.1:8000/graphql/` renders GraphiQL.
Auth: put JSON in the **Headers** pane: `{"Authorization":"Bearer <token>"}`.
Typing a JWT by hand is error-prone — click the editor and type it with
`DISPLAY=:0 xdotool type --delay 25 "$(cat /tmp/token.txt)"` instead. GraphiQL auto-inserts closing
braces/quotes, so prefer single-line queries typed via xdotool and `ctrl+Return` to execute.

Get a token:
```sh
curl -s http://127.0.0.1:8000/graphql/ -H 'Content-Type: application/json' \
  -d '{"query":"mutation{tokenCreate(email:\"admin@example.com\",password:\"admin\"){token errors{message}}}"}'
```

## 3. Checkout → order golden path

`products(first:3, channel:"default-channel", filter:{isPublished:true, stockAvailability:IN_STOCK})`
→ variant id → `checkoutCreate` → `checkoutShippingAddressUpdate` + `checkoutBillingAddressUpdate`
→ `checkout(id){shippingMethods}` → `checkoutDeliveryMethodUpdate` → authorize → `checkoutComplete`.

Payment on 3.2x: `checkoutComplete` requires `authorizeStatus: FULL` unless
`channel.allow_unpaid_orders`. Use staff `transactionCreate(id, transaction:{amountAuthorized:...})`
(needs `HANDLE_PAYMENTS`). To then charge the order use
`transactionUpdate(id, transaction:{amountCharged:...})` — `orderMarkAsPaid` is rejected with
`TRANSACTION_ERROR "Orders with transactions can not be manually marked as paid."`, so pick
`orderNoteAdd` or another mutation if you only need a staff-auth/permission proof.

Always run each mutation once with the Authorization header removed first: expect a permission error
naming the missing permission and `data.<mutation> == null` (no side effect).

## 4. Live introspection vs committed schema

graphene 2 / graphql-core 2.3.2 cannot parse the committed SDL (block-string descriptions) and has no
`get_introspection_query`. Introspect with graphql-core 2's own introspection query, then parse and
compare **structurally** (types/fields/args/enum values/input fields/interfaces/unions) using a
separate graphql-core 3.x venv (`uv venv /tmp/gqlvenv && /tmp/gqlvenv/bin/pip install graphql-core==3.2.11`).
Naive SDL text diffing produces thousands of false positives.

Known pre-existing structural differences that graphene 2 introspection does not report (10 entries:
`format: ThumbnailFormatEnum = ORIGINAL` args and `PaymentInput.storePaymentMethod` defaults) — verify
they reproduce against the base branch's `schema.graphql` too, which proves they are not caused by
your change.

## 5. Post-run log check

```sh
grep -cE 'Traceback|ImportError|ModuleNotFoundError' /tmp/uvicorn.log
grep -E ' 5[0-9][0-9] ' /tmp/uvicorn.log
grep -E 'DeprecationWarning|RemovedIn' /tmp/uvicorn.log
```
An `AuthlibDeprecationWarning` from `saleor/core/jwt_manager.py` is pre-existing, not a regression.

## Devin Secrets Needed
None — all local (Postgres/Valkey via docker compose, local admin user).
