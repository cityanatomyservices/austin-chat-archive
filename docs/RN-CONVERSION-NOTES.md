# austin.chat — React Native conversion notes (planning paused)

_2026-08-12. An RN conversion was scoped by two code-exploration passes, then
paused — the product idea needs more thought first. This doc preserves what
the exploration found so it isn't lost. Some findings are live-site bugs
worth fixing regardless of the app._

## Bugs found in the live site (fix independent of any app)

1. **Anyone can wipe two tables.** RLS on `chat_votes` and `geofence_tags`
   has `DELETE ... using (true)` for anon — an unfiltered
   `DELETE FROM chat_votes` from any browser succeeds. Only client-side
   politeness prevents it. Scope deletes to the caller (needs an identity —
   see below) or drop anon delete entirely.
2. **Silent message loss.** `templates/pubchat/chat.js` `persistCityChatRow`
   (≈lines 486–499) inserts `app: origin.app || null` and
   `hotspot_id: ... || null` into NOT NULL columns (0001 schema). When origin
   is incomplete the insert fails 23502 — and the error handler only
   special-cases `P0001`, so the failure is swallowed: the message broadcasts
   live but never reaches history or archives. Relax the constraints or stamp
   defaults.
3. **Archives read the wrong column.** `templates/archive/archive.js`
   queries `chats?app=eq.<bucket>` but the city-chat path stores the
   *category* in `app` and the bucket in `bucket`. Posts made through the
   current app never appear on the archive pages. Filter on `bucket`.
4. **View RLS footgun.** `poll_results`, `chat_vote_counts`,
   `geofence_tag_counts` lack `WITH (security_invoker = true)` — they bypass
   base-table RLS (the comment in 0001 claiming inheritance is wrong).
   Safe today (aggregates only); adding any raw column would leak past the
   24-hour read window. Add security_invoker before extending.
5. **Deployment unverified** (the parcel_geojson lesson): every client error
   is swallowed, so if migration `0004` (votes/tags/buckets) or the pg_cron
   purge jobs were never applied, the app looks identical. Verify in the SQL
   editor:
   `select to_regclass('public.chat_votes'), to_regclass('public.geofence_tags');`
   `select jobname, active from cron.job;`
   `select min(created_at) from public.chats;` — if the oldest row is >30
   days old, the purge cron is not running and the table grows unbounded.
6. **Copy vs reality.** The app says "nothing is stored"; rows live 30 days
   and are republished on public archive pages ("permanently", per the
   archive footer). Align the copy (or the retention) before any app ships.

## Architecture digest (for the future RN plan)

- The "family of maps" collapsed into ONE app: `/index.html` (~3,900 lines),
  27 categories in 5 buckets, 1,202 hotspots from `data/<slug>/hotspots.json`.
  `/CLAUDE.md` describes the old per-map layout and is stale.
- Chat: ONE city-wide Supabase Realtime channel (`chat:atx`),
  **broadcast-first** (DB insert is best-effort side-write), origin metadata
  stamped per message, filtered client-side. Presence via channel.track.
- Identity: none — random handle+emoji in sessionStorage; rate limits are
  DB triggers keyed on the client-chosen handle (bypassable by renaming).
  RN has no sessionStorage, so identity must be rebuilt anyway → natural
  moment for Supabase anonymous auth (`signInAnonymously()`), real
  ownership RLS, honest rate limits.
- Safety surface: nothing (no report/block/moderation UI; moderation =
  manual SQL editor). Play Store UGC policy will require report + block.
- Supabase project `tqnklodtiithbsxxyycp` — dedicated; no overlap with the
  `parcels` project. Anon key + TomTom key ship in client code (TomTom key
  lightly obfuscated at index.html:3449).
- Dead code (~2,100 lines, don't port): `templates/pubchat/{index.html,
  style.css,ui.js,pubchat-engine.js}`, `overview/`, polls (defined,
  dormant), the v1 per-hotspot chat API in chat.js.
- Mobile data concern: parks 5.3 MB + neighborhoods 2.3 MB polygon JSON,
  floodzone 12 MB — needs tiling/simplification for an app.
- Schedules module (`templates/pubchat/schedule.js`) is pure and ports as-is;
  the geofence scorer (`recomputeEntered`, smallest-ratio-wins) is pure and
  ports as-is.

## When planning resumes

The user will bring a product outline first (the current shape is
deliberately undecided). Open questions parked: harden the chat model vs
port as-is; safety scope; app name; whether the feed/ticker and archives
come along.
