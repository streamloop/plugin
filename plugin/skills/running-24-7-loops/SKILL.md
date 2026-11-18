---
name: running-24-7-loops
description: "Sets up and runs a Streamloop video stream with its MCP: importing or uploading videos and music, filling and reordering its playlists, shuffle and custom play orders, scheduling (once, daily, weekly, restart every N hours), starting and stopping, changing what a live stream plays without going off air, and finding out why a stream won't start or stopped. Use when someone wants a lofi or ambience channel, an always-on stream, \"loop these videos to YouTube\", \"add this clip to my stream\", \"stream every weekday at 9\", \"never two songs from the same artist in a row\", or asks why their stream isn't live."
---
# Running 24/7 loops

A **stream** (`stream_…`) loops its **video playlist**, with an optional **audio playlist** under it, to its **destinations**, now or on a schedule. It spends the workspace's credits while it runs.

## Setting one up, in order
1. `stream_create { name, kind: "pre_recorded" }` answers its `id`, `videoPlaylistId` and `audioPlaylistId` (both empty). Then `stream_update { id, quality: "q_1080p", framerate: "f_30", streamDuration: 0 }`. `streamDuration` is seconds after which a run stops by itself; `0` runs until stopped, which a 24/7 stream needs (`stream_get` shows the current value).
2. Media in, one of two ways. Reuse what the workspace has first: `upload_list { name?, type? }`.
   - **A link** (a YouTube, Vimeo or Twitch page, or a direct file link): `upload_import_from_url { url }` (`maxHeight` to cap the quality, near the stream's; `startSeconds`/`endSeconds` to import part of a video page). It answers the media `id` (`obj_…`) at once and imports in the background: `upload_get_import_job_state { id }` until `ready` is true, or `error` says why not.
   - **A file on the user's machine**: `upload_get_upload_url { name, type: "video" | "audio" | "image", mimeType, fileSize }` answers `id` and `uploadUrl`. PUT the whole file to `uploadUrl` with `Content-Type` equal to `mimeType` (do it yourself if you can make HTTP requests; otherwise give the user the URL and the header). There is no completion call: processing starts once the file has arrived, and `upload_get_import_job_state` shows `awaiting_upload` until then. Over 5 GB: `multipart: true` answers `partSize` and one URL per part; PUT each part, then `upload_get_import_job_state` assembles the file once every part is there.
3. Fill the playlist: `playlist_patch { playlistId, patch }`, a JSON Patch over the document `playlist_get { id }` answers (`version`, `items: [{ id, objectId, name, duration }]`). One operation per change:
   - add: `{ op: "add", path: "/items/-", value: { objectId: "obj_…" } }` (end) or `/items/<index>`;
   - remove: `{ op: "remove", path: "/items/<index>" }`;
   - reorder: `{ op: "move", from: "/items/<index>", path: "/items/<index>" }`;
   - swap media in place: `{ op: "replace", path: "/items/<index>", value: { objectId } }`;
   - guard: `{ op: "test", path: "/version", value: <version from playlist_get> }` first refuses the patch if someone edited the playlist since you read it.
   Indexes are the playlist as it was read at the start of the call; all operations apply or none. Music under the videos goes in the audio playlist the same way.
4. A destination: skill streaming-to-destinations, then `stream_add_output { streamId, destinationId }`.
5. `stream_preflight_check { streamId }` lists every blocking issue with its fix (no destination, empty playlist, media still being analyzed, missing YouTube settings, not enough credits). Then `stream_start { streamId }`, or a schedule. A refused start is `STREAM_NOT_READY` with the same issues. An accepted one is live within about a minute: `stream_get` shows `state` (`preparing`, `activating`, then `active`).

## Schedules (the rules that bite)
`stream_schedule { streamId, startAt, endAt? | durationSeconds?, repeat?, days?, restartGapSeconds? }` replaces any schedule the stream had.
- Every time carries the user's UTC offset: `2026-09-22T21:00:00-04:00`, never a bare local time (it is refused). The answer restates the window in UTC (`scheduledUTC`): read it back to the user in their own time.
- `repeat`: `none`; `daily` / `weekly` (the same session again, which needs `endAt` under 24 h after `startAt`; weekly `days` are the UTC weekdays of the start, so a Monday 9 pm Eastern start is a Tuesday in UTC); `restart` (run `durationSeconds`, at least 600, stop, wait `restartGapSeconds`, start again until stopped: "restart every 12 hours with a 30 s break" is `durationSeconds: 43200, restartGapSeconds: 30`).
- The offset is fixed, not a time zone: when the user's clocks change for daylight saving, the window moves an hour in their day. Say so.
- A schedule is refused like a start when the stream isn't ready. `stream_cancel_schedule { streamId }` removes it; a running session keeps going.

## Changing a live stream
A running stream keeps the playlist it started with. `playlist_patch` with `live: true` puts the edited playlist on air without stopping (new media may be prepared first). If the answer is `PARTIAL_FAILURE` (the edit was saved, the switch failed), don't send the patch again: send only `[{ op: "test", path: "/version", value: <its version> }]` with `live: true`. Stream settings and outputs change on a stopped stream.

## Play order
- `playlist_set_playing_order { playlistId, mode: "sequential" | "shuffle" }`.
- Anything else ("start with the intro, never two songs in a row", "a jingle every 5 songs"): `custom_order_create { name, intent, playlistId }` writes rules from the user's words (always pass the `playlistId`, so rules can name its items). Read `status`: `READY`; `INVALID` (see `diagnostics`); `MESSAGE` (a question back: ask the user, answer with `custom_order_refine { id, message }`); `PENDING` (still being written: `custom_order_list { id }` later). Creating doesn't apply it: `playlist_set_playing_order { playlistId, mode: "custom", customOrderId }` does, and answers the first items it will play. `custom_order_list` shows the orders already written.

## Finding, stopping, reusing
- `stream_list { status?, name? }` (`active`, `scheduled`, `inactive`), `stream_get { id }`.
- `stream_stop { streamId }` ends a run; nothing is deleted. A repeating schedule starts it again at its next window unless `cancelSchedule: true`.
- `stream_create_copy { id, name? }` copies settings, playlists, order, destinations and YouTube settings (not the schedule). `playlist_copy` fills an empty playlist from another.
- `stream_remove { id }` deletes a stopped stream with its playlists (media stays); `upload_remove { id }` deletes media no playlist uses.

## When it won't start, or stopped
`stream_preflight_check` before a start; afterwards `stream_get`: a failing or failed stream has `problem` with the `reason` and what to do. Short of credits: `account_get` shows the balance, and the user adds credits in the Streamloop dashboard. Errors carry a code (`STREAM_LIVE`, `IN_USE`, `CONFLICT`): act on it.

## Before acting
`stream_start`, `stream_schedule`, `stream_stop` and `playlist_patch` with `live: true` change what the public sees; `stream_remove` and `upload_remove` can't be undone. Say what will happen and get the user's yes first.
