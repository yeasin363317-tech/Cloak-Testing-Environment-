# Cloak — Scalability

Honest assessment of what happens as concurrent usage grows, in the order things would actually break.

## Current architecture's scaling properties

**What scales well, structurally, without any changes:**
- The frontend is a single static file — hosting it for 10 users or 10 million costs and behaves identically, since it's just file serving (no server-side rendering, no per-request compute).
- WebRTC call *media* never touches Supabase at all — audio flows peer-to-peer (or fails to connect, see `ARCHITECTURE.md` §8). Call volume doesn't add database or bandwidth load on our infrastructure, only on the two participants' own connections.
- Database indexes are in place on the hot paths (`rooms.code` unique, `messages(room_id, created_at)`, `room_members(room_id, session_id)` unique) — query performance shouldn't degrade meaningfully with row count growth, since every real query is scoped to a single room.

**What does not scale automatically — this is where 1,000 concurrent users would actually hurt:**

### 1. Realtime connection count (the first thing to break)

Each connected client currently opens **3 separate Realtime channels** (`room-messages-X`, `room-state-X`, `room-members-X`). At 1,000 concurrent users, that's **3,000 concurrent Realtime connections** — this will exceed Supabase's free-tier and even standard Pro-tier concurrent connection limits well before you hit 1,000 users; you'd need to either consolidate channels (technically straightforward — multiple `.on()` handlers can share one channel object, cutting this to 1 per client) or upgrade to a plan/add-on with a higher Realtime connection ceiling. **This is the single highest-leverage fix and hasn't been done yet.**

### 2. The polling fallback runs unconditionally

Every connected client polls `resume_room` every 3 seconds **regardless of whether Realtime is healthy** — it's meant as a backstop, but currently runs all the time as a parallel path. At 1,000 concurrent users that's roughly **330+ requests/second** sustained against the database, purely from a mechanism that should be idle almost all the time. Making this genuinely conditional (only poll while the Realtime channel's own status reports disconnected) removes this load almost entirely under normal conditions.

### 3. Database connection/request limits by plan tier

Supabase's connection pooling (PgBouncer) and PostgREST both have limits tied to your project's plan. RPC calls and the two directly-queried tables (`messages`, `room_members`) all go through this same pool. 1,000 concurrent *active* users (not just connected, but actively sending messages) would likely require at least a Pro-tier project, possibly with compute add-ons, to avoid connection exhaustion under burst load (e.g., many rooms' messages arriving at once).

### 4. `pg_cron` cleanup job

Runs a simple `DELETE ... WHERE expires_at <= now()` every 2 minutes. This scales fine even with a large number of rooms — it's a single indexed sweep, not per-room work — not a concern at any realistic scale for this app.

## Rough capacity estimate, as-is today

On a Supabase free-tier project, this architecture would likely start showing real strain (dropped Realtime connections, slower message delivery, possible rate-limit errors) somewhere in the **low hundreds of concurrent users**, driven almost entirely by items #1 and #2 above — not by database size or query performance, which are already in good shape.

## Priority order to actually support 1,000 concurrent users

1. **Consolidate the 3 Realtime channels into 1 per client.** Biggest single win, no infrastructure cost, code-only change.
2. **Make the poll a true fallback**, gated on Realtime connection status instead of running unconditionally.
3. **Move to a Supabase plan with sufficient Realtime connection and database compute headroom** for your target concurrency — the exact tier depends on Supabase's current published limits at the time you scale, worth checking directly against their pricing page rather than assuming a fixed number here.
4. **Add basic rate-limiting** on `create_room` / `join_room` / message sending (already flagged in `SECURITY.md` as a deferred item) — at higher scale, a single misbehaving client script has more potential to degrade service for everyone else.

None of this requires a rewrite. Items #1 and #2 are the same "Phase 1" work already identified in the project roadmap, just not yet implemented — they're also, not coincidentally, the correct first step for real scale.
