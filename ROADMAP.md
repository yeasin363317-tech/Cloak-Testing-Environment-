# Cloak — Roadmap

## Planned: optional account system (username + password, no email/phone)

The core anonymous, session-based model stays as the default — this is an **additive** feature, not a replacement.

**Requirement, as specified:** sign-up asks for a display name, a username, and a password — no email, no phone number. On success, the account gets a permanent UID. Returning users log in with either their username *or* UID, plus their password.

### Proposed schema

```sql
create table users (
  uid uuid primary key default gen_random_uuid(),
  username text unique not null,
  display_name text not null,
  password_hash text not null,       -- bcrypt, same pattern as room passwords
  created_at timestamptz default now()
);
```

### Proposed functions

- `register_user(p_username, p_display_name, p_password)` — validates username uniqueness and format, bcrypt-hashes the password, returns the new `uid`.
- `login_user(p_identifier, p_password)` — accepts either a username or a UID as `p_identifier`, looks up the matching row, verifies the password hash, returns the profile (never the hash).

### Open design questions to settle before building this

1. **What does a logged-in identity actually change inside a room?** The room/message model is currently built entirely around ephemeral `session_id`s. Does a logged-in user's `session_id` get *linked* to their `uid` (for something like "my room history"), while the room itself stays just as ephemeral as today? Or does this go further? Worth deciding deliberately rather than by default.
2. **Password recovery.** With no email or phone on file, there is *no* mechanism to recover a lost password — it would be a permanently lost account. Worth deciding upfront whether to accept that trade-off (consistent with the "no personal info, ever" philosophy) or add something like a one-time recovery code shown once at signup, which the user is responsible for saving themselves.
3. **Brute-force protection becomes mandatory, not optional, once this ships.** Session IDs today aren't a meaningful attack target (they're not "logged into" anything). A `username + password` pair is a classic credential-stuffing target the moment it exists. This should ship *with* login rate-limiting/lockout from day one — not bolted on after, the way the rest of the app's abuse protection has been deferred so far (see `SECURITY.md`).
4. **Username enumeration.** Login failures shouldn't reveal *why* they failed ("wrong password" vs "no such user") — a generic "incorrect username/UID or password" avoids leaking which usernames exist.

## Carried over from prior planning (still open)

### Performance / scale (see `SCALABILITY.md` for full detail)
- Consolidate the 3 per-client Realtime channels into 1.
- Make the 3-second poll a true fallback, only active while Realtime is actually disconnected.

### Security (see `SECURITY.md` for full detail)
- Rate limiting on `create_room` / `join_room` / message sending.
- Revisit the broad `messages`/`room_members` SELECT policies if the privacy bar for this app ever needs to rise above "casual anonymous chat" (would require moving reads to RPCs and switching Realtime to `broadcast`-based delivery).

### Calling
- Currently STUN-only, 1-on-1. A TURN relay would substantially reduce connection failures on restrictive mobile networks, at the cost of running/paying for relay infrastructure.
- Group calling would require a different architecture entirely (a media server / SFU), not just "more STUN" — full-mesh WebRTC doesn't scale past a handful of participants.

### UX polish
- Accessibility: `aria-label`s on icon-only buttons.
- Loading-state polish (skeleton placeholders while history loads).

### Admin tooling
- `CloakAdmin.html` was built as a temporary, disposable operator tool — revisit whether it's still needed, or whether Supabase's own dashboard (Logs/Reports) covers what's actually used in practice.
