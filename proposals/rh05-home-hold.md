# RH05 #341 recovery addendum: durable managed-home hold

**Status:** local proposal for fresh Opus review and explicit owner sign-off.
This branch is not the published contract. The exact candidate frozen diff is
in `schema.md`, `control-api.md`, `agent-api.md` and `openapi.yaml` on top of the
owner-signed #341 pin `cb752156`. No shared database has applied 0090. The
candidate extends #341's reserved 0090 migration, not 0091 (#342), and does
not rewrite a deployed migration.

**Compatibility decision for review:** use Opus N1 option (a), selected by the
initiative lead under the owner's invariant that older agents retain supported
behavior. New holds apply only to dispatch on a cleanup-capable authenticated
connection epoch. Older agents keep the signed legacy path and its known
synthetic-reap risk, which the admin view must disclose. This decision requires
the same Opus review and owner sign-off as the rest of the frozen addendum.

## Failure and scope

A swap target lives only in memory while `sessions.app_id` still names the old
app. A reconnect or heartbeat reaper can make the session terminal and erase
`state_detail='swapping'` without agent teardown proof, exposing that target
to tombstone and GC. The original assigned managed home has the same gap once
the session is synthetically terminal. Existing agent `failed` reports also
cannot be accepted as unmount proof: fatal swap currently reports failure
before teardown, and an abandoned runner may report failure while its
container remains. This addendum protects both original and swap-target
canonical claims, and makes cleanup evidence an explicit agent capability.

## Exact 0090 schema addition

Extend `managed_home_claims` in the unpublished 0090 up migration:

```sql
ALTER TABLE managed_home_claims
  ADD COLUMN legacy_unprotected_dispatch boolean NOT NULL DEFAULT false,
  ADD COLUMN pending_home_session_id uuid,
  ADD COLUMN pending_home_token uuid,
  ADD COLUMN pending_home_started_at timestamptz,
  ADD CONSTRAINT managed_home_claims_pending_home_ck CHECK (
    (pending_home_session_id IS NULL AND pending_home_token IS NULL
      AND pending_home_started_at IS NULL)
    OR
    (pending_home_session_id IS NOT NULL AND pending_home_token IS NOT NULL
      AND pending_home_started_at IS NOT NULL)
  );
CREATE INDEX managed_home_claims_pending_home_session_idx
  ON managed_home_claims (pending_home_session_id)
  WHERE pending_home_session_id IS NOT NULL;
```

The three columns can instead be part of the original `CREATE TABLE`; the
resulting schema is identical. No FK links the marker to `sessions`, so a
synthetic terminal state or session deletion cannot erase it. The claim's
existing `host_id` remains the owner; there is no second host field. The
random token is internal CAS identity, not sent on the wire or exported.
The hold's historical session relationship is absent from API responses,
events, traces, logs and diagnostics; existing session resources retain
normal session IDs. Claim state, conflict reason and `materialized_at` retain
their signed meanings. The partial index supports qualified late terminal
reconciliation when the session row is gone.
Backfilled legacy `user_homes` claims set `legacy_unprotected_dispatch=true`;
the flag is also set on an unsupported-agent dispatch and remains true through
later upgrades until audited repair. It records coverage history, never
physical presence or absence.
The first accepted assigned-to-running transition may still materialize the
original claim under its signed mount-binding proof; the hold remains until
qualified cleanup proof.

0090 adds three `BEFORE DELETE` triggers and `_fn` functions:

| Table | Trigger | Guard |
| --- | --- | --- |
| `managed_home_claims` | `rh05_guard_managed_home_claim_delete` | Refuse direct or cascading deletion of a held claim. |
| `users` | `rh05_guard_user_delete_home_hold` | Refuse deletion while any of that user's claims is held. |
| `apps` | `rh05_guard_parent_app_delete_home_hold` | Refuse canonical parent deletion while any claim for it is held; a tile-only deletion leaves the parent claim. |

Each function locks affected claims in ascending
`(user_id,canonical_app_id)` order and, if any has a non-NULL token, uses
`RAISE EXCEPTION USING ERRCODE='QH001', MESSAGE='managed home operation pending'`.
It returns `OLD` for an allowed DELETE; it **never returns NULL** to skip a
foreign-key cascade. HTTP handlers map only `QH001` to their existing
`409 conflict` envelope. The 0090 down migration drops all three triggers
and functions, then the partial index and table, after the sessions binding
columns and signed host-delete trigger. It never deletes `user_homes` or
backing data. 0087, 0088, 0089 and 0091–0093 remain owned by other tickets.

## Agent proof capability

An agent may set optional `register.terminal_home_cleanup_v1=true` only if it
removes and verifies absence of every original and swapped source container
**before** terminal `session_state{stopped|failed}` and holds home refs until
cleanup succeeds. An `ack{ok:false}` for assign/swap must also certify that
**that command** caused no home side effect; rejecting a swap does not tear
down the prior live source. Fatal swap tears down before reporting failure.
Abandoned or panicked runners verify/remediate containers before reporting
failure or freeing refs; if cleanup cannot be proved, they withhold terminal
proof and keep protection. A capable agent durably records accepted session
IDs before creating containers, reconciles orphan containers/refs and queued
pre-capability events on startup, then emits qualified terminal for every
cleaned ID after registration. `session_stop` for a session it held emits
terminal after cleanup even when CP already reaped it. Terminal IDs are
durably retired and later assign/swap commands for them are rejected without
side effects, across restart. Agent tests cover fatal swap, abandoned runner,
startup recovery, stop-after-reap and retired-ID rejection.

The control plane binds the capability to the authenticated connection epoch,
not to a version string, stored host field, or previous registration. Missing
or false means unsupported and every reconnect replaces the value. Older
agents keep signed legacy managed-home launch/relaunch behavior: their
dispatches set **no new RH05 hold** and their terminal reports provide no
RH05 cleanup proof. This retains the known synthetic-reap risk on those
agents. The admin claim read exposes current connection capability and a
sticky legacy-unprotected-dispatch warning; no API reports unheld work as
physically safe. A previously created capable hold still needs qualified
terminal proof or #347 repair if the agent is later downgraded.

## State and evidence matrix

| Event | Hold outcome | Proof boundary |
| --- | --- | --- |
| Original managed-home assignment on capable command epoch | Set with `sessions.managed_home_id` and digest in one dispatch transaction before `session_assign`. | Commit precedes send; no physical-use claim. |
| Managed-home swap target on capable command epoch | Set under session/claim/home locks before mount resolution or `session_swap_app`. | Covers claim-only targets and old `sessions.app_id`. |
| Assignment or swap on unsupported command epoch | No new hold; set sticky `legacy_unprotected_dispatch=true`. | Signed legacy behavior continues with known synthetic-reap risk. |
| Same session targets an already-held canonical home | Reuse existing token/time; never replace or clear on this later command's failure. | Allows tile-to-tile and A→T→A→T without another writer. |
| Other session targets a held canonical home | Refuse generic `home_conflict`; unrelated canonical homes proceed. | Single writer despite synthetic terminal session. |
| Local failure before send; no authenticated **command connection epoch** at send time; failure before frame handed to socket | CAS-clear only newly created matching token. | No delivery possible. |
| Socket write error after handoff, timeout, lost ack, disconnect or crash | Retain. | Delivery/acceptance uncertain. |
| Negative ack on the same cleanup-capable **command connection epoch** and exact in-memory command→token correlation | CAS-clear only newly created matching token; original claim reservation remains. | Rejection promises no home side effect from that command. |
| Negative ack after restart or for reused old token | Retain. | No durable correlation; token never reused. |
| Running, swap complete or rollback callback | Retain. | Callback lacks operation ID, so repeated/out-of-order result is ambiguous. |
| Stop, reconnect/offline/heartbeat reap, restart or session deletion | Retain. | Control-plane terminal is not unmount proof. |
| Authenticated terminal from cleanup-capable connection, reporting host equals non-NULL claim host and existing session host | CAS-clear all that historical session's holds in ordered claim locks. | Agent certifies all its source containers are gone. |
| Late qualified terminal after synthetic terminal/deleted row | Change matching hold columns only. | No session rewrite/event/reservation/materialization. |
| Unsupported/unknown agent terminal, wrong-host terminal, NULL claim host, or failed cleanup for an existing hold | Retain for #347. | No qualified proof; old-agent dispatch itself created no new hold. |
| Host deletion | Tombstone its homes, set affected claims `conflict/claim_owner_missing` with NULL owner, retain holds. | Null-host janitor may remove row, never claim. |

`session_state` has no swap operation ID, so successful/rolled-back swap
callbacks cannot clear a target hold. The same session can carry holds on
several canonical claims after multiple swaps. A cleanup-capable terminal
report clears them together; this is safe because session IDs are never
reused. A terminal event generated before a swap means that same agent could
not subsequently accept a swap for that stopped session because capable
agents durably retire terminal IDs. No terminal report from a pre-capability
agent counts, even if its message is later replayed.

## Transaction, API and deletion rules

Session rows if present (ascending ID) precede claim rows in ascending
`(user_id,canonical_app_id)` order, then `user_homes`. If the session row is
gone, the partial index locates its held claims before ordered locks. This
order covers terminal cleanup, #347 repair, launch, tombstone and GC. The
HTTP user/app delete transaction first locks affected sessions in ascending
ID, then claims in ascending key order, checks holds, then tombstones homes
and DELETEs the owner in that same transaction. A refusal rolls back every
tombstone. Direct SQL deletion can deadlock against a callback; PostgreSQL
aborts one transaction, leaving no partial tombstone. No network operation
runs while DB locks are held.
Tombstone, GC pull and GC confirmation refuse a held canonical home even if
its session is terminal/missing or still names an old app. The signed admin
host-delete path is an explicit exception: it can tombstone while preserving
the claim and hold as a NULL-host conflict. Null-host janitor cleanup never
releases that claim. User and canonical-parent hard deletion refuse while
held; unheld claims retain signed cascade behavior. Tile-only deletion keeps
the parent claim and existing active-session guard.

`GET /v1/admin/storage/home-claims` adds required boolean
`pending_home_operation`. It means *may have mounted*, not *currently mounted*.
`home_cleanup_capability` is `supported|unsupported|unknown` from the current
authenticated owner connection; `unknown` covers offline or null-host claims.
It is capability, not historical cleanup proof. Required boolean
`legacy_unprotected_dispatch` is sticky for backfilled legacy homes or a
managed-home dispatch on an unsupported connection; false does not prove
physical absence. These fields reveal no hold session, token, source path,
mount or provider. Filters, order, cursor, state and conflict reason are
unchanged. The #341 admin console SHOULD show these diagnostics with the
`design_handoff_v3/` visual verification path;
the API is authoritative if UI work is deferred. Legacy
`GET /v1/admin/storage/homes` remains unchanged. Ordinary launch refusal
uses the signed fixed `409 home_conflict` message, with no hold details.

`DELETE /v1/users/{id}` and `DELETE /v1/apps/{id}` use HTTP 409 with
`ErrorEnvelope.error.code="conflict"` and fixed message
`Managed home cleanup is pending` for a held user or canonical parent. The expired ephemeral-user
reaper prefilters held users and handles a racing `QH001` per row in its own
transaction or savepoint, so other users continue; their tokens
expire as before, but identity removal waits for proof or #347 repair. This
amends the prior unconditional-reap wording. Strict older API clients need
regeneration for the new required admin fields; clients that ignore unknown
JSON item fields continue reading. Older agents require no new message
shape, keep signed relaunch behavior without new holds, and show the sticky
unprotected warning. A capable hold remains if that agent is later downgraded.

Example admin item:

```json
{"items":[{"user_id":"00000000-0000-4000-8000-000000000011","username":"test-user","canonical_app_id":"00000000-0000-4000-8000-000000000021","app_name":"Test App","host_id":"00000000-0000-4000-8000-000000000031","host_name":"test-role","state":"reserved","conflict_reason":null,"materialized_at":null,"recorded_host_ids":[],"pending_home_operation":true,"home_cleanup_capability":"supported","legacy_unprotected_dispatch":false}],"next_cursor":null}
```

On an older connected agent, the same item reads
`"pending_home_operation":false,"home_cleanup_capability":"unsupported","legacy_unprotected_dispatch":true`.
The user can relaunch under signed legacy behavior; none of those fields
claims that an old container is absent. After the agent disconnects or is
upgraded, the sticky warning remains true while current capability changes
to `unknown` or `supported`.

An old control plane must never be deployed against a database migrated to
this 0090: it cannot enforce holds. A rollback ref must embed this migration
and protection. No live host, image or user data is modified by this draft.

## Acceptance tests for #341 implementation

Use real ephemeral Postgres for lock/concurrency and a deterministic transport
adapter for delivery order; run agent tests in the sanctioned Rust container:

1. Capable original assignment writes binding+hold before dispatch. Synthetic
   reconnect/heartbeat/offline reap and row deletion retain the hold; launch,
   tombstone and GC pull/confirm refuse until qualified proof.
2. Capable swap target commits hold before mount resolution; claim-only and live homes
   stay protected during delay, lost ack, Stop and synthetic terminal.
3. Concurrent second launch/swap into a held target refuses without leaking
   host/session detail; unrelated canonical home proceeds. Same-session
   tile-to-tile and A→T→A→T reuse the token; reject of later swap leaves it.
4. No command-connection-epoch/before-socket failure clears only a new token.
   Same-epoch capable negative ack clears only a new token without tearing
   down a prior source. Post-handoff write error, timeout, lost ack, reconnect
   epoch mismatch and late negative ack after restart retain.
5. Duplicate/out-of-order swap completion/rollback/ordinary running never
   clear. Qualified matching terminal clears exact session holds once, even
   after synthetic reap. Wrong host, NULL owner, missing capability and old
   queued terminal do not; late proof changes no public session side effect.
6. Agent fatal-swap test verifies source teardown and ref retention before
   terminal report. Panicked/abandoned runner test verifies container absence
   before terminal and no ref release/report when cleanup is uncertain.
   Startup orphan cleanup and stop-after-CP-reap emit qualified terminal for
   recorded sessions; terminal IDs are durably non-reusable and later
   assign/swap is rejected without side effects.
7. Direct SQL held-claim/user/parent delete raises `QH001` (never silently
   skips cascade); tile-only delete retains parent claim. Ephemeral reaper
   prefilters held user and catches a racing `QH001` per row while continuing
   to delete independent expired users; refused HTTP delete leaves no
   partially tombstoned home.
8. Host deletion tombstones held home, leaves NULL-host conflict and hold;
   null-host janitor can remove its row but never claim. GC/host-delete race
   preserves the hold.
9. Admin pagination/filter read shows all three diagnostics without private
   identifiers; legacy homes read and generic refusal stay stable. Old agent
   dispatch creates no hold, preserves relaunch, sets sticky unprotected
   warning and shows unsupported/unknown current capability; later upgrade
   does not erase warning. Capable dispatch creates hold and qualified
   terminal clears it; downgrade retains an existing hold.

Local tests do not prove physical unmount or multi-host absence. Live acceptance
requires combined 0087+0088+0090 migration-capable integration ref, both GPU
test roles, isolated data, exact source/image/schema identities, and separate
authorization before stack or backing-home mutation.
