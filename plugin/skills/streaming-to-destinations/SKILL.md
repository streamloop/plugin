---
name: streaming-to-destinations
description: "Connects a Streamloop stream to where it is watched, with its MCP: a YouTube channel signed in with Google or by stream key, Twitch, Facebook, Kick, X, TikTok, LinkedIn, Telegram or any RTMP server; sends one stream to several at once and turns one off while live; sets the YouTube broadcast's title, description, privacy, category and thumbnail; fixes a destination that stopped working. Use when someone with a Streamloop stream says \"put it on YouTube\", \"also send it to Twitch\", \"stream to two platforms\", \"my YouTube stopped\", \"change the title or thumbnail\", \"how many viewers\", or pastes a stream key."
---
# Streaming to destinations

A **destination** (`dest_…`) is where a stream's output goes. One stream publishes to up to 5 at once, at most one YouTube channel, all from one encode.

## Making one: never guess an address
Start with `destination_connection_hints { platform }` (`youtube`, `twitch`, `facebook`, `x`, `kick`, `tiktok`, `linkedin`, `telegram`, `other`): the exact ingest URL, where the user finds their stream key, and the platform's limits. A wrong host is a stream that never connects.
- **YouTube, signed in with Google** (title, privacy, thumbnail and viewer numbers from here): `destination_create_youtube_oauth {}` answers a Google consent URL and the channels already authorized. The user opens the URL and approves (an assistant can't); call it again to see the channel, then `destination_create_youtube_oauth { channelId, name? }` with the `channelId` from that answer, not a channel URL or handle.
- **A stream key** (Twitch, Facebook, Kick, X, TikTok, Telegram, YouTube by key, any server): `destination_create_rtmp { name, rtmpUrl, rtmpKey }` with `rtmpUrl` from the hints. The key is stored encrypted and never read back. A YouTube key carries video only: nothing about the broadcast can be set through it.
- `destination_list { kind?, name? }` shows the workspace's destinations and their status.

## Putting it on a stream
- `stream_add_output { streamId, destinationId }`, on a stopped stream. Refusals name the reason: `STREAM_LIVE`, `DESTINATION_LIMIT` (5), `YOUTUBE_DESTINATION_LIMIT` (a second channel), `DESTINATION_DUPLICATE`, `DESTINATION_INCOMPATIBLE` (the stream's quality or codec can't reach it).
- While live the only change is taking one of several off: `stream_toggle_output { streamId, destinationId, enabled: false }`; the others stay up. Not the last one (`LAST_DESTINATION`: stop the stream instead). Turning one back on, adding or unlinking (`stream_remove_output`) waits for a stopped stream: `stream_stop`, the change, `stream_start` is about a minute off air, so say so and get a yes first. Nothing here is timed: "YouTube off for an hour" is a disable now and, in an hour, a stop, enable and start.
- `stream_get { id }` lists each output with its `state` and why it is off.

## Keeping destinations working
- A YouTube connection whose permission expired or was withdrawn: `destination_create_youtube_oauth {}` again; the user approves, and the same destination works again.
- The platform rotated the key: `destination_update { id, rtmpKey }` (a live stream uses it after a restart). `destination_update { id, name }` renames; `destination_remove { id }` removes a destination from the workspace.

## The YouTube broadcast
With a channel signed in with Google linked: `stream_update_youtube_broadcast_settings { id, title, description, privacy, madeForKids, category, tags, … }`. `madeForKids` is required by YouTube before a start: ask the user, never assume. A custom thumbnail is an uploaded image (`upload_get_upload_url` with `type: "image"`) as `thumbnailSource: "custom", thumbnailObjectID`. `scheduleOnYoutubeWhenScheduled` shows the broadcast as upcoming on the channel; after reconnecting the channel, `retryMirror: true` copies the schedule to YouTube again. While live, `stream_get` answers the viewer numbers (`youtubeLiveStats`). `stream_update { id, runBackupStream: true }` also publishes to YouTube's backup ingest.

## Cost
Each destination after the first adds a flat fee per 30 days, by resolution, charged while the stream is live; `account_get` shows the balance and burn rate.
