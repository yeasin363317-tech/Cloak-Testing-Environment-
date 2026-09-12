# Cloak — talk, then vanish

Cloak is an anonymous, ephemeral chat application. No accounts, no persistent identity, no message history that outlives the room. Create a room, share a 6-character code, talk — and when the room closes, every trace of it (messages, membership, room record) is permanently deleted.

**Live philosophy:** if it doesn't need to exist after the conversation ends, it shouldn't exist in the database at all.

---

## What Cloak actually is

- A single-file static web app (`index.html` — HTML/CSS/JS, no build step, no framework)
- Backed entirely by [Supabase](https://supabase.com) — Postgres, Realtime, and PostgREST as the only "backend"
- No traditional server process. The static file can be hosted anywhere (GitHub Pages, InfinityFree, Netlify, etc.)
- A companion `CloakAdmin.html` — a separate, unlinked operator panel for monitoring and moderation, backed by the same Supabase project

## Feature summary

- Create or join a room via a 6-character code
- Optional password protection (bcrypt-hashed server-side, never stored in plaintext)
- Mandatory session duration per room (5 min – 7 days) — rooms cannot exist forever
- End-to-end encrypted messages (AES-GCM, key derived from room code + password — Supabase itself only ever sees ciphertext)
- Host controls: adjustable member limit, ban/remove a member, extend or end the session, change or remove the password mid-conversation
- 1-on-1 WebRTC audio calling (no server relay for the audio itself — only signaling passes through Supabase Realtime)
- Reply-to-message, typing indicators, light/dark/system theming, PWA install support

## Project structure

```
index.html        ← the entire app (this is what you deploy)
CloakAdmin.html    ← separate operator/moderation panel (deploy at an unlinked, obscure path — never link to it from index.html)
sw.js              ← service worker (bump its cache version string on every deploy, or users will keep loading a stale cached copy)
manifest.json, icon-192.png, icon-512.png  ← PWA assets
```

## Deploying it

1. Upload `index.html` (rename it to `index.html` if it isn't already), `sw.js`, `manifest.json`, and the icon files to your host's web root.
2. **HTTPS is not optional.** Message encryption and audio calling both depend on browser APIs (`crypto.subtle`, `getUserMedia`) that only work on a secure origin. Over plain HTTP, the app still runs, but silently falls back to unencrypted messages and calling won't work at all.
3. Deploy `CloakAdmin.html` separately, at a path you don't link from anywhere. It has its own passphrase gate — see `ARCHITECTURE.md` for how that's secured.
4. Every time you re-deploy `index.html`, bump the cache name inside `sw.js`. If you forget, returning visitors' browsers will keep serving the old cached version even after you've uploaded the new one.

## Using it (as an end user)

- **Create Room**: pick a name, a member limit, a session duration, and optionally a password. You get a 6-character code to share.
- **Join Room**: enter the code (and password, if the room has one). You'll see a preview (host's name, how many people are already in) before you commit.
- **In a room**: swipe a message to reply to it, tap the people icon to see who's there (and, if you're the host, to remove someone or start a call), tap the code pill for room info, use the kebab menu for settings, password, member limit, or to close/leave.
- Everything disappears the moment the room closes — by the host, by an admin, or automatically when the session timer runs out.

## Backend (Supabase) at a glance

Nothing in the client talks to the database directly except two tables (`messages`, `room_members`) that need direct Realtime access to work — everything else (creating rooms, joining, passwords, bans, admin actions) goes through `SECURITY DEFINER` Postgres functions, each enforcing its own authorization checks. See `ARCHITECTURE.md` for the full breakdown and `SECURITY.md` for the threat model.

## Further reading

- **`ARCHITECTURE.md`** — how every part of the system actually works
- **`SECURITY.md`** — what's protected, what's a known accepted trade-off, and why
- **`SCALABILITY.md`** — what happens as usage grows, and what breaks first
- **`ROADMAP.md`** — what's planned next, including a future account system
