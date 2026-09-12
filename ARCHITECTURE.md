# Cloak — Architecture

## 1. System shape

There is no application server. The system is two layers:

```
Browser (index.html — vanilla JS, no framework, no build step)
        │
        │  HTTPS / WebSocket
        ▼
Supabase project
  ├─ PostgreSQL           (data + all business logic, as SQL functions)
  ├─ PostgREST            (auto-generated REST API over Postgres)
  ├─ Realtime             (Postgres CDC + pub/sub broadcast, over WebSocket)
  └─ pg_cron              (scheduled jobs inside Postgres itself)
```

The client embeds Supabase's public `anon` API key directly in the HTML. This is expected and safe — that key carries no special privilege on its own; it identifies the *project*, not a *user*. Authorization is enforced entirely server-side (§4, §5).

`CloakAdmin.html` is a second, independent client of the same Supabase project, with its own authentication scheme (§6).

## 2. Data model

| Table | Purpose | Notable columns |
|---|---|---|
| `rooms` | One row per active room | `code` (6-char, public), `password_hash` (bcrypt, nullable), `expires_at` (mandatory), `admin_closing` / `last_banned_session_id` (signal columns, see §5) |
| `messages` | Chat messages | `content` / `reply_to_content` are ciphertext (`enc1:<base64>`); `sender_name` is plaintext |
| `room_members` | Who is in which room | `(room_id, session_id)` unique; `is_creator` flag |
| `room_bans` | Ban records | `(room_id, session_id)`, cascades away when the room is deleted |
| `admin_config` | Single-row table | `key_hash` — bcrypt hash of the admin passphrase, never plaintext |

There are no user accounts. Identity is a random UUID (`session_id`) generated on first use and kept in `localStorage`. Every privileged action is checked against `room_members.session_id`, not against any cryptographic proof of identity — see §7 for the implication of this.

## 3. Room lifecycle

Every room has a mandatory `expires_at` — there is no "forever" room, enforced at both the client (no such option exists in the UI) and the database (`create_room` rejects a missing/non-positive duration).

Expiry is enforced two ways simultaneously:
- **Lazy sweep**: `create_room`, `join_room`, `preview_room`, and `resume_room` all call `cleanup_expired_rooms()` at the start, deleting any room past its `expires_at` as a side effect of ordinary traffic.
- **Scheduled sweep**: a `pg_cron` job runs the same cleanup every 2 minutes, so expiry is enforced even with zero active clients.

Closing a room — by its creator, by an admin, or by expiry — is a hard SQL `DELETE`, not a status flag. Messages, members, and ban records cascade away with it.

## 4. Authorization model

All meaningful logic lives in Postgres functions marked `SECURITY DEFINER` (they run with elevated privilege, deliberately bypassing Row-Level Security) — but every one of them re-implements its own authorization check before doing anything:

```sql
select rm.is_creator into v_is_creator from room_members rm
where rm.room_id = p_room_id and rm.session_id = p_session_id;
if v_is_creator is null or v_is_creator = false then
  raise exception 'NOT_CREATOR';
end if;
```

Functions: `create_room`, `join_room`, `preview_room`, `resume_room`, `leave_room`, `close_room`, `update_member_limit`, `set_room_password`, `ban_member`, `extend_room_session`, `cleanup_expired_rooms`, and the `admin_*` family.

Every function explicitly sets `search_path = public`. This isn't cosmetic — an unset search_path is a known Postgres privilege-escalation vector (a malicious schema earlier in the resolution order could shadow a built-in function). Every function here closes that door.

**A bug worth remembering**: any function that `RETURNS TABLE(id uuid, ...)` implicitly creates a PL/pgSQL variable named `id` in scope. An unqualified `WHERE id = ...` inside that function is ambiguous — is it the table column or the return variable? — and the call fails outright. This shipped and broke password-changing and member-limit-changing before being caught. Every function now qualifies these explicitly (`public.rooms.id`).

## 5. Realtime: two mechanisms, chosen deliberately per use case

**`postgres_changes`** (row-level CDC, requires the row to be visible under RLS to the subscriber) drives: live message delivery, member-count/password/expiry updates, "someone joined" notices.

**`broadcast`** (ephemeral pub/sub, no database row involved) drives: typing indicators, WebRTC call signaling (offer/answer/ICE candidates), and the password-sync-on-change mechanism.

**Why this split matters — a real incident**: admin-initiated room closures were originally signaled via `broadcast` sent from the admin panel just before the `DELETE`. This was unreliable — broadcast delivery has no ordering guarantee relative to the subsequent `DELETE`'s CDC event, so the "closed by admin" flag frequently lost the race and the affected client fell back to a generic "the creator closed this" message. **Fix**: the admin panel now flips `rooms.admin_closing = true` (an ordinary `UPDATE`, on the same reliable CDC path that member-count changes already use) and waits ~2.5s before actually deleting the row. Same pattern for ban notifications (`last_banned_session_id` column). Lesson: prefer a durable-row UPDATE over a fire-and-forget broadcast for anything that has to race a subsequent state change.

A 3-second polling fallback (`resume_room` on an interval) exists as a backstop if Realtime itself is unhealthy.

## 6. Admin panel (`CloakAdmin.html`)

A fully separate HTML file, never linked from the main app, served with `<meta name="robots" content="noindex, nofollow">`. It authenticates by passphrase — stored server-side only as a bcrypt hash — and **every** privileged admin RPC re-validates that passphrase on every call (no session token exists to be stolen). It can list rooms, see message metadata and raw ciphertext, and remotely close rooms or ban members — it never derives an encryption key or attempts to decrypt anything, even for password-less rooms where it technically could.

## 7. End-to-end encryption

- Key: `PBKDF2(room_code + "|" + password, salt = "cloak-salt-" + room_code, 100,000 iterations, SHA-256)` → AES-GCM 256-bit key, via `window.crypto.subtle`.
- Each message gets a fresh random 12-byte IV; ciphertext is stored as `enc1:<base64(iv + ciphertext)>`.
- Supabase never sees plaintext message content, by construction.
- **Trade-off, accepted deliberately**: this is a *shared-secret* scheme, not a real per-user key exchange. Whoever knows the room code (and password, if set) can derive the key — there's no forward secrecy and no protection against someone who legitimately has valid room credentials. This is the correct trade-off for "anonymous room chat," not for anything requiring real per-identity cryptographic guarantees.
- **Requires a secure context.** If ever served over plain HTTP, `crypto.subtle` doesn't exist and the app silently falls back to sending plaintext rather than crashing — a deliberate degrade-gracefully choice, but worth auditing for before any production deploy.

## 8. Audio calling

1-on-1 only, WebRTC, signaling multiplexed over the same Realtime channel a room already uses for state updates (`call-offer` / `call-answer` / `call-ice` / `call-decline` / `call-end` broadcast events, addressed by `session_id`). Media itself is peer-to-peer (or fails to connect) — Supabase never touches the audio stream. Currently STUN-only (Google's public servers); no TURN relay exists yet, so calls between two peers on restrictive/symmetric NATs (common on mobile carrier networks) can fail to connect entirely. See `ROADMAP.md`.

## 9. Client-side resilience notes

- `localStorage` access is wrapped in a feature-detected fallback to an in-memory store — some browsers (Safari private mode, storage-locked webviews) throw on *any* storage access, which previously crashed the app at load with a blank screen and no error.
- Every RPC call on a user-initiated action (close, leave, extend) is wrapped so a network failure surfaces a toast instead of leaving the UI silently stuck.
