# RH05 #341 proposed recovery addendum: durable home-operation hold

**Status:** proposal for Opus review and explicit owner sign-off. This branch is
not the published contract. The exact candidate frozen diff is in `schema.md`,
`control-api.md` and `openapi.yaml` in this worktree. It builds on the
owner-signed #341 home-claim and mount-binding contract at `cb752156`.

## Failure and scope

`session_swap_app` leaves `sessions.app_id` on the old app until a callback.
The current in-memory target and `state_detail='swapping'` protect the target
only while the session row remains nonterminal. `ReapHostExceptRunning` can
turn a `stopping` row into `failed` on reconnect; heartbeat omission can also
terminalize it. Neither is authenticated proof that the agent stopped using
the target home. The synthetic terminal state erases the durable detail, and
the old app ID does not identify the target. Tombstone or GC can then run.

The addendum stores a target-specific hold on its existing canonical claim.
It changes no agent message, endpoint path, public session state, placement
rule, or home materialization proof. It does not make a callback without an
operation ID prove which swap completed. Claims remain owned by the same host.

## Exact 0090 schema addition

Extend the unpublished 0090 up migration; do not insert a new migration before
0091 or alter any schema that has been deployed. 0090 currently belongs to
#341; 0087, 0088, 0089, 0091, 0092 and 0093 are reserved for their tickets.

```sql
ALTER TABLE managed_home_claims
  ADD COLUMN pending_swap_session_id uuid,
  ADD COLUMN pending_swap_token uuid,
  ADD COLUMN pending_swap_started_at timestamptz,
  ADD CONSTRAINT managed_home_claims_pending_swap_ck CHECK (
    (pending_swap_session_id IS NULL AND pending_swap_token IS NULL
      AND pending_swap_started_at IS NULL)
    OR
    (pending_swap_session_id IS NOT NULL AND pending_swap_token IS NOT NULL
      AND pending_swap_started_at IS NOT NULL)
  );
CREATE INDEX managed_home_claims_pending_swap_session_idx
  ON managed_home_claims (pending_swap_session_id)
  WHERE pending_swap_session_id IS NOT NULL;
```

No FK links the marker to `sessions`: session reaping/deletion cannot erase
it. The token is a random UUID used for internal compare-and-swap, never a
wire field or exported identifier. The hold's historical session ID and token
are absent as hold metadata from API responses, events, traces, logs and
diagnostic exports; existing session APIs keep their own IDs. The
preexisting claim PK bounds one hold
per canonical user/home; multiple target claims may hold the same session.
This addition does not change `state`, `conflict_reason`, `materialized_at`,
the host-delete trigger, or legacy backfill. The existing 0090 down migration
drops the table and therefore the new columns and index.

Add 0090 `BEFORE DELETE` guards on `managed_home_claims`, `users` and canonical
`apps`: if a matching claim has a non-NULL pending token, refuse the delete.
This prevents direct SQL or existing hard-delete endpoints from cascading
away the only recovery marker. The HTTP user/parent-app deletion paths map
this guard to their existing 409 conflict envelope. Deleting a derived tile
alone does not cascade the canonical parent's claim; its existing active-
session guard still applies. Host deletion may proceed under the signed host
trigger; it leaves the marker on a null-host conflict, requiring #347 repair.

## State transitions and evidence

| Event | Claim hold | Session/public outcome | Reason |
| --- | --- | --- | --- |
| Managed-home swap target selected | Set `(session, token, started_at)` in claim transaction before mount resolution/send | `state_detail='swapping'` remains advisory | Protects claim-only and live homes while app ID still names old app. |
| Mount resolution fails before any send | Clear matching token | Swap errors | Agent could not receive target payload. |
| Send proves no delivery or authenticated agent rejects | Clear matching token | Swap rejects | Exact local token and no acceptance proof. |
| Send timeout, lost ack, disconnect or process crash | Retain | Existing response/retry semantics | Acceptance/mount is uncertain. |
| Accepted running, swapping, complete or rolled-back callback | Retain | Existing app ID/detail transitions may occur | Callback has no swap operation ID; duplicate/out-of-order delivery is ambiguous. |
| Operator Stop, reconnect reap, heartbeat omission, synthetic terminal or session deletion | Retain | Existing session outcome | Control-plane transition is not proof of unmount. |
| Authenticated terminal `session_state` from claim owner for exact historical session | CAS-clear matching session/host holds | Existing terminal or late terminal processing | Agent reports the session ended; no target remains mounted by that session. |
| Host deletion or missing agent | Retain | Claim may become `conflict/claim_owner_missing` | No negative mount evidence. |
| #347 audited operator repair | Clear under documented inspection and CAS | Separate workflow | Only recovery when no authenticated terminal proof arrives. |

The terminal callback handler must process a late authenticated terminal for
the marker even if a synthetic reap already made the session row terminal;
it must not reverse or re-notify that public session transition. It checks the
authenticated reporting host against the claim owner, historical session ID
and token under the session row (if present) then claim lock. An untrusted
HTTP caller, another host, heartbeat list, swap callback, stale negative ack,
or detached callback cannot clear a marker. A terminal report for a session
ID that has never been reused is evidence that the reporting agent no longer
mounts any of that session's targets, whether the report was delayed before
or after the swap; a pre-swap terminal means the agent could not subsequently
accept the swap for that same session. A resolved token is never reused.
The pre-dispatch and ack paths compare the exact token so a late failure for
an older request cannot clear a newer hold.

**Conservative cost:** a successful managed-home swap can retain a hold while
the session remains alive; another launch/swap of the same canonical home and
home deletion are refused until authenticated terminal proof or #347 repair.
This is necessary with the current callback wire and is visible to admins.
If the product requires earlier release, a separately reviewed operation ID
echo or authoritative mount inventory is needed; this proposal does not
silently infer completion from an ambiguous callback.

## Interface and compatibility examples

`GET /v1/admin/storage/home-claims` adds one required boolean per item, with
all existing filters/cursors and state meanings unchanged:

```json
{"items":[{"user_id":"00000000-0000-4000-8000-000000000011","username":"test-user","canonical_app_id":"00000000-0000-4000-8000-000000000021","app_name":"Test App","host_id":"00000000-0000-4000-8000-000000000031","host_name":"test-role","state":"reserved","conflict_reason":null,"materialized_at":null,"recorded_host_ids":[],"pending_home_operation":true}],"next_cursor":null}
```

The flag means *may have mounted*, not *currently mounted*. No token, session
ID, provider/ref or mount path appears. A caller attempting to launch a held
canonical home receives the existing `409 home_conflict` fixed message, which
does not reveal the hold. `GET /v1/admin/storage/homes` is unchanged and still
cannot show claim-only uncertainty. An older API client that ignores unknown
JSON object keys can read the item; a strict client must regenerate from the
additive OpenAPI field. Older agents keep their existing swap/terminal wire
behavior; they do not have to understand the marker. If they never report an
authenticated terminal, the hold remains for repair. An old control plane
must not be used against a migrated 0090 DB because it cannot enforce the new
hold; the migration-compatible rollback ref must include this behavior.

## Transaction and refusal rules

Use the signed lock order: session row if present, then target claim, then
`user_homes`. No network call while DB locks are held. The per-user advisory
lock serializes first claims; target hold creation and host/placement checks
commit before dispatch. Every launch, swap and local launch checks the target
claim hold in its reservation transaction. A held target returns generic
`home_conflict` before derived-tile `home_not_provisioned`. A second swap to a
different target may proceed only if its own claim is safe; the earlier hold
continues to protect its target. Tombstone, GC pull and GC confirmation check
the target claim's hold, regardless of the session row's current state or app
ID. A stale GC confirmation cannot bypass the claim lock or delete the hold.

## Acceptance tests for #341 implementation

Use real ephemeral Postgres and operator/session seams; deterministic transport
adapters control ack/callback order. These are required in addition to the
already signed #341 tests:

1. Target claim and hold commit before dispatch; tombstone and GC pull refuse
   during mount resolution and after a lost ack, including claim-only target.
2. A second session launch and same/different-session swap to the held target
   refuse without leaking the first session or host; unrelated canonical home
   still works independently.
3. Stop, restart, reconnect `ReapHostExceptRunning`, heartbeat omission and
   session-row deletion retain hold and admin flag; no GC confirmation release.
4. Duplicate and out-of-order `swap complete`/`rolled back`/ordinary running
   callbacks never clear the hold. A late explicit reject cannot clear a newer
   token. Pre-send failure and explicit matching rejection clear safely.
5. Authenticated matching terminal callback clears exact matching holds once,
   even after synthetic reap. Wrong host/session and duplicate terminal do not
   affect another hold. Terminal public state is not replayed.
6. Host deletion leaves hold on null-host conflict; user/parent deletion fails
   closed; tile-only deletion preserves its parent claim. Direct SQL guards
   are exercised on the isolated DB.
7. Admin pagination and filters return `pending_home_operation` correctly;
   legacy homes endpoint and ordinary `home_conflict` envelope remain stable.
8. A concurrent launch/tombstone/GC race observes the same claim lock order;
   one safe outcome commits with no deadlock or second writer.

No local test proves physical unmount or multi-host absence. Live acceptance
requires the combined migration-capable integration ref (0087+0088+0090),
both GPU test roles, isolated test data, exact source/image/schema identities,
and separate authorization for deployed-stack or backing-home mutation.
