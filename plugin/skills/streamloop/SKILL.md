---
name: streamloop
description: "What Streamloop is and how a stream gets on air: the two kinds of stream (pre-recorded video, scene) and which fits a channel, the way from nothing to live through the Streamloop MCP, quality and what it costs in credits, which workspace the connection works in, and when the dashboard, the studio or the HTTP API is the place instead. Use when someone first brings up Streamloop or a 24/7 stream and the task is not yet a specific one: \"stream my videos 24/7\", \"what do I need to go live\", \"which quality\", \"how many credits do I have\", \"how long can this run\", \"what did I spend last month\", \"should I use a scene\", \"I can't find my stream\"."
---
# Streamloop

Streamloop runs a live stream from the cloud, around the clock: no OBS, no encoder, no PC left on. The dashboard calls one a **loop**; the MCP calls it a **stream** (`stream_…`). It plays to **destinations** (YouTube, Twitch, Kick, Facebook, X, TikTok, LinkedIn, Telegram or any RTMP server; up to 5 at once, at most one YouTube channel) and spends the workspace's **credits** while it is live. Streamloop watches every stream and restarts it if it stalls.

## Two kinds of stream: choose first

| | Pre-recorded video | Scene |
| --- | --- | --- |
| What plays | uploaded videos on repeat, with an optional audio playlist under them | a designed live show: frames of layers, cut between |
| Quality | up to 4K 60 fps | up to 1080p 60 fps |
| Live inputs (RTMP, SRT, RTSP cameras, HLS) | no | yes |
| Web pages, text, logos, clocks, live data (sheets, CSV, JSON) | no | yes |
| Overlays, lower thirds, tickers, transitions, a fallback when a feed drops | no | yes |
| A script that reacts to time and data; operator controls | no | yes |

Pick **video** when the channel is media on repeat: music, ambience, a show archive. Pick a **scene** when something has to change on air (scores, headlines, weather), come from a live feed, or sit on top of video. Scenes are available to every workspace. `stream_create { name, kind }` decides it (`pre_recorded` or `scene`); a stream doesn't change kind over the MCP, so make a new one.

## The way to live, in order

1. `stream_create { name, kind }`, then `stream_update { id, quality, framerate, streamDuration: 0 }`: `streamDuration` 0 runs until stopped, which a 24/7 stream needs.
2. What it plays: media in a playlist (skill running-24-7-loops), or a scene built and published (skill building-scenes).
3. A destination linked with `stream_add_output` (skill streaming-to-destinations).
4. `stream_preflight_check { streamId }`: every blocking issue at once, each with its fix, credits included. Then `stream_start` now or `stream_schedule` for later. A start is live within about a minute.

## Quality and credits

Resolution `q_720p`, `q_1080p`, `q_1440p` or `q_2160p` (scenes up to 1080p); frame rate `f_24`, `f_25`, `f_30` or `f_60`. 1080p at 30 fps suits most channels; higher costs more per minute.

Credits are spent only while a stream is live, by resolution and frame rate; on top, each destination after the first adds a flat fee while live, a scene adds its rendering, and new videos pay a one-time preparation fee. 1080p 30 fps around the clock is about $10 per 30 days ($1 = 1,000,000 credits). `account_get` answers the workspace's balance (`billing.creditsLeft`), the burn rate now (`spend`), each running stream's cost so far (`running`) and the rate per quality (`rates`, credits per minute). "How long can it run": `creditsLeft` ÷ `currentCreditsPerMinute`; say it is an estimate that leaves out the flat fees. Past usage (minutes streamed and credits spent by month) has no MCP tool: it is on the dashboard's Billing page. When the balance runs short, the user adds credits in the Streamloop dashboard: no tool buys credits or changes a plan.

## The workspace

Streams, media, destinations and credits belong to a **workspace**. A connection works in one, chosen when the user connected the Streamloop connector; `account_get` names it. No tool switches workspace: when something the user is sure exists isn't in `stream_list`, `upload_list` or `destination_list`, it is most likely in another workspace, and the user connects again choosing that one. Members, roles and API keys are managed in the dashboard.

## Which way in

- **The Streamloop MCP** (`https://mcp.streamloop.app/mcp`, OAuth: the user signs in as themself) is the way for an assistant acting for someone. Its tools are named `<resource>_<action>`; ids are prefixed (`stream_`, `pl_`, `obj_`, `dest_`, `oseq_`, `scn_`): read them from a tool, never make one up. Errors carry a code and what to do next.
- **The dashboard** (`streamloop.app/loops`) and, for scenes, the **studio** are the user's own hands: signing in, paying, Google's YouTube consent, and what the MCP has no tool for (encoder keys, secrets).
- **The HTTP APIs** (`https://api.streamloop.app/v1`, REST and Scenes) are for code: scripts, backends, cron, CI. Their reference is at `https://streamloop.app/docs/api-reference/introduction`.

## Then read

| The task | Skill |
| --- | --- |
| uploads, playlists, play orders, schedules, a stream that won't start, a playlist change while live | running-24-7-loops |
| YouTube, Twitch, any RTMP; multistream; one destination off while live; YouTube title and privacy | streaming-to-destinations |
| a scene from nothing to on air, publishing, rolling back | building-scenes, then composing-frames, animating-frames, shaping-data, binding-live-data, scripting-the-show, handling-media |
| a design reused across frames (a lower third, a score bug, a card) | writing-components |
| a layer the scene's elements can't draw (rich CSS, SVG, charts, canvas), or HTML or React asked for | coding-web-pages |
