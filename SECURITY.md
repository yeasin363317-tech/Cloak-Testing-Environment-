# Cloak — Security

## Threat model, in one sentence

Cloak protects **message content** from Supabase itself and from casual observers, and protects **room/account integrity** from unauthorized actors — it does not attempt to provide cryptographic proof of identity, since there are no accounts.

## What's protected, and how

| Concern | Mechanism |
|---|---|
| Message content confidentiality | Client-side AES-GCM, key never leaves the browser (see `ARCHITECTURE.md` §7) |
| Room passwords | bcrypt hash only, `pgcrypto`, never plaintext, never returned by any RPC |
| Admin passphrase | bcrypt hash only, in an RLS-locked table with zero policies (no direct API access exists at all) |
| Ban enforcement | Server-side check in `join_room`, backed by an RLS-locked `room_bans` table |
| Privilege escalation via search_path | Every function pins `search_path = public` |
| Direct table tampering | RLS enabled on every table; sensitive tables have zero policies (SECURITY DEFINER functions are the only way in) |
| Message injection by non-members | `messages` INSERT policy requires `EXISTS (... room_members ...)` proof of membership |

## Fixed during development (kept here as a record, not a current risk)

- `admin_config` and `room_bans` briefly had RLS **disabled** (an oversight when the tables were created) — anyone with the public API key could have read the admin password hash directly, or deleted their own ban record to bypass it. Caught via Supabase's own automated security advisor, fixed same-day.
- `messages` INSERT previously allowed any `session_id` to post into any `room_id` with no membership check at all.
- A dead, unused `push_subscriptions` table (from an earlier development pass, never wired into the app) had fully open INSERT/UPDATE/DELETE policies. Removed entirely rather than patched, since nothing depended on it.

## Known, accepted limitations (not bugs — documented trade-offs)

1. **`messages` and `room_members` SELECT policies are broad.** Anyone with the public API key can read message *metadata* (sender name, timestamp, which rooms exist) and full member lists, across all rooms, without knowing a room code. Message *content* stays protected regardless (encrypted). Properly closing this would require moving reads off direct table access onto RPCs and switching Realtime delivery from `postgres_changes` (which needs this policy to function) to `broadcast` — a real architecture change, deliberately deferred. Acceptable for a casual anonymous chat app; would need revisiting for anything higher-stakes.

2. **Password changes mid-session relay the new plaintext password once, via Realtime broadcast**, to whichever members are currently connected, so their clients can re-derive the key without leaving and rejoining. That plaintext is not persisted or logged, but it does transit Supabase's infrastructure for a moment — a deliberate usability trade-off over strict E2EE purity.

3. **No rate-limiting or anti-abuse system.** Room creation, joining, and messaging have no throttling. Someone could script mass room creation or message spam. Deferred as a "Phase 2" item; acceptable risk at current scale.

4. **Identity is a client-supplied `session_id`, not a cryptographic credential.** Anyone who obtained a valid member's `session_id` (e.g. by reading their `localStorage` on a shared device) could act as them. There is no account system to provide a stronger guarantee — this is inherent to an anonymous, no-signup product, not an oversight. See `ROADMAP.md` for how a future account system would change this.

## What is *not* a vulnerability (common misconception, worth documenting)

The Supabase `anon` key embedded in the client is **public by design** — every Supabase (and Firebase) app ships it this way. It carries no elevated privilege by itself; the actual dangerous credential is the `service_role` key, which bypasses RLS entirely and has never been placed in any client file (verified by decoding the embedded key's JWT payload — it reads `"role":"anon"`, never `"role":"service_role"`).

## Verification method

Claims in this document were checked live against the database (via SQL, including `SET LOCAL ROLE anon` to actually simulate the public API's permissions, not the privileged connection used to administer the project), not inferred from code alone. Rows created for testing were cleaned up afterward.
