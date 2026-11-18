---
name: building-scenes
description: "Builds a Streamloop scene with the Streamloop MCP and changes it while it is on air: a live, designed stream (text, graphics, tickers, scoreboards, lower thirds, live data, cameras) instead of a looping video; the scene tools, the way from an empty scene to published, what the answers mean, changing a scene viewers are watching, and rolling a bad publish back. Use when someone wants, on their Streamloop stream, an overlay, a lower third, a score bug, a ticker, a weather or news corner, \"graphics on my stream\", a layout that changes on its own, \"change the headline on air\", \"what's on air\", \"go to the guest shot\", \"update the score now\", \"undo that\", or anything that isn't only a video playlist, even if they say overlay, graphics or show rather than scene."
---
# Building scenes

A **scene** (`scn_…`) is what a scene stream plays. It is made of **frames**, full pictures of layers at `frame/<id>`, one on air at a time, plus the **sources** it reads (tables of data, feeds, cameras, files, playlists, web pages), reusable **components**, operator **controls**, and a **script** that reacts to events. You edit its **draft**; viewers see nothing until it is published. People with the scene open in the Streamloop studio see each edit as it lands: treat it as shared work.

## The tools
- `scene_get { sceneId, path }` reads anything: `path: "*"` first (the kinds of resource and their definitions, and the order of work), a type (`element/Text`, `source/data`) for what it takes, a resource (`frame/intro`), a mask (`frame/*`). `paths: [...]` reads several at once. `as: "image"` on a frame or a layer answers a picture of it as it goes on air; `as: "json"` the frame as a tree of layers. `state: "published"` or `revision` reads what was published.
- `scene_set { sceneId, path, replace | merge, basedOnVersionId? }` creates or changes one resource in the draft: `replace` the whole resource, `merge` only the fields given (a merge patch; `null` removes a field), or one text edit `{ old, new }` on its code.
- `scene_remove { sceneId, path }`; `scene_publish { sceneId }`; `scene_list_revisions { sceneId }`; `scene_rollback_to_revision { sceneId, revision }`.
- Paths: `element/*`, `frame/<id>`, `frame/<id>/<layer>`, `source/<kind>/<id>`, `component/<Name>`, `control/<id>`, `script/show.ts`, `script/state.ts`, `tokens`.
- Installed with this plugin, a client may show them with a prefix (Claude Code lists `mcp__plugin_streamloop_streamloop__scene_get`, …).

## The way through
1. The scene: `stream_get { id }` answers a scene stream's `sceneId` (null: a video stream, which doesn't become a scene; make a new one). A new one: `stream_create { name, kind: "scene" }` answers the stream and its `sceneId`.
2. Learn the vocabulary once: `scene_get *`, then `scene_get element`, then the props of the elements you need in one call (`paths: ["element/Text", "element/Stack"]`). Props differ: a Text wraps only with `width`; a Stack places its children and ignores their x/y.
3. Data first, if any: a table from a URL, a sheet or a feed (skill shaping-data). Its answer says what arrived and the columns a frame can bind.
4. A frame: `scene_set { sceneId, path: "frame/<id>", replace: { name, description, code } }`, where `code` is one `<Frame background={…}>…</Frame>` element of layers (`Text`, `Number`, `Rect`, `Image`, `Video`, `Ticker`, `Stack`, …); later layers draw on top. The smallest one that works:
   ```jsx
   <Frame background={{ type: "solid", color: "#0B1B3A" }}>
     <Text id="title" value="Starting soon" x={96} y={440} width={1728} style={{ fontSize: 120, fontWeight: 800, color: "#FFFFFF", textAlign: "center" }} />
   </Frame>
   ```
   Bind live values directly: `value={data.weather.rows[0].temp}`, `{controls.showCard ? <…/> : null}` (skill binding-live-data). Layout: composing-frames; a design that repeats: writing-components. A frame is the whole picture viewers get, with nothing under it: an "overlay" on a camera or video is that feed in the same frame, as its background (`{ type: "video", source: "<source id>" }`) or a Video layer under the graphics. A camera or encoder is a source first (skill handling-media).
5. Look, every time: `scene_get { sceneId, path: "frame/<id>", as: "image" }` draws it and lists `drawn` and `issues` (outside title-safe, overlapping, text too small, a bound table with no rows). Fix what it says, then stop: one or two rounds, not ten.
6. `scene_publish { sceneId }` publishes the whole draft as it is now, edits made in the studio meanwhile included. It is refused with `SCENE_INVALID` and the problems to fix.
7. On air: the scene stream plays the published scene. Link a destination, check and start it (skills streaming-to-destinations, running-24-7-loops); a scene stream runs at 1080p or lower. It opens on the first frame `scene_get frame/*` lists; the script or the operator takes the others to air.

## While it is on air
- `scene_publish` changes what viewers see at once, without going off air: the frame on air stays (when the new version still has it), and so do the controls' values. There is no dry run over the MCP: say what will change from the edits you made (or compare `scene_get` of the draft with `state: "published"`), and get a yes first.
- A publish that went wrong: `scene_list_revisions { sceneId }`, then `scene_rollback_to_revision { sceneId, revision }` publishes that earlier revision again as a new one. The draft keeps the latest edits: fix them before the next publish, or it puts them back on air. Throwing the whole draft away is done in the studio; over the MCP, set each changed resource back to what `scene_get { path, state: "published" }` answers.
- What the MCP has no tool for: seeing which frame is on air, taking a frame to air by hand, setting a control's value, replacing a table's rows for one moment, skipping a playlist track. The operator does those in the studio's Live panel. To have them happen on their own, write them into the script (skill scripting-the-show: schedules, reactions to data and feeds); a value that changes often belongs in a table from a URL, not in republished code. `stream_get` shows the stream's state and, when failing, why.
- Encoder stream keys, camera passwords and API keys for tables are typed in the studio, never through the MCP (skill handling-media, shaping-data).

## Why the answers look the way they do
- Every write is built and checked as the studio checks it, and a refusal changes nothing. Read `errors`: each problem with its line or path. One that says "already in the draft" is not yours but must be fixed in the same set.
- `basedOnVersionId` is optional: the `revision` from `scene_get` or from the last `scene_set` answer for that path (for a layer, its own or its frame's). With it, a resource someone changed since is refused (`CONFLICT`) instead of overwritten.
- Code is stored formatted (props alphabetical, one per line). For a text edit copy `old` from what `scene_get` answered; `EDIT_NO_MATCH` quotes the closest line.
- Positions are design pixels (1920×1080 unless the scene says otherwise) from the parent's top-left.

| Code | What it means | Do |
|---|---|---|
| `CHECK_FAILED` / `SCENE_INVALID` | the change or the whole scene would break | fix each entry of `errors`, send again |
| `CONFLICT` | it changed since your `basedOnVersionId` | `scene_get` it, redo the change on what is there |
| `EDIT_NO_MATCH` / `EDIT_AMBIGUOUS` | `old` isn't there / is there twice | copy from the stored code; add context |
| `STALE_PUBLISH` | someone published since this draft was based on it: publishing would take their version off air | tell the user; rebuild on what is published now. The MCP can't publish over it |
| `IN_USE` | something still uses it | change or remove the users first |
| `NOT_FOUND` | wrong id or path | `scene_get frame/*`, or the `sceneId` from `stream_get` |

## More
Layout, colour, type: composing-frames. Motion: animating-frames. Tables and formulas: shaping-data; bindings: binding-live-data. Media, cameras, encoders, playlists: handling-media. Components: writing-components. Script, timers, controls: scripting-the-show. Web pages as layers: coding-web-pages.
