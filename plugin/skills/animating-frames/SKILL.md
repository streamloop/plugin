---
name: animating-frames
description: "Adds motion to the frames of a Streamloop scene through the Streamloop MCP — how layers enter and leave, how bound position and size changes tween, how changing values roll or swap (count up, flash), and how a frame transitions in. Use when someone working on a Streamloop scene says \"animate\", \"slide in\", \"fade\", \"count up\", \"stagger\", \"transition\", or wants something on their stream to move instead of cut."
---
# Animating frames

> With the Streamloop MCP: a scene is reached by its id, `sceneId` (`scn_…`): `stream_get` answers it for a scene stream, and `stream_create { name, kind: "scene" }` makes a new one. Every `scene_*` tool takes it. A call written `scene_get('frame/intro', { as: 'image' })` is `scene_get { sceneId, path: "frame/intro", as: "image" }`, and `scene_set('frame/intro', { replace: … })` is `scene_set { sceneId, path: "frame/intro", replace: … }`: `replace` is the whole resource, `merge` only the fields to change (a merge patch, or one text edit `{ old, new }`). To have a change refused when someone edited the resource meanwhile, add `basedOnVersionId`: the `revision` you read. Every set changes the draft only; `scene_publish { sceneId }` puts the whole draft on air at once on every stream that plays the scene, so tell the user what will change first. Starting one from nothing: skill building-scenes.

Motion is a layer's `motion` prop plus the frame's transition. Each prop is described at scene_get('element/<Type>').

```jsx
<Stack id="strap" x={96} y={836} padding={[20, 32]} motion={{ enter: { kind: "rise", ms: 420 }, exit: { kind: "fade", ms: 200 } }} style={{ background: "#0B0F1AE6" }}>
  <Text id="strapName" value={controls.guestName} style={{ fontSize: 56, fontWeight: 800, color: "#FFFFFF" }} />
</Stack>
<Number id="homeGoals" x={96} y={120} value={state.score.home} motion={{ change: "countUp" }} style={{ fontSize: 72, fontWeight: 800, color: "#FFFFFF" }} />
<Rect id="lead" x={96} y={240} w={state.score.home * 200} h={12} motion={{ transition: { ms: 600, ease: "out" } }} style={{ background: "#E63946" }} />
```

- `enter` plays when the layer mounts: its frame goes on air, its condition turns true, its row is added. `exit` plays when it goes. Kinds: fade · rise (up from below) · drop (down from above) · slide (in from the left) · wipe · scale; `ms`, `delay`. A kind has one direction; there is no `from`.
- `motion.transition` tweens a layer's position and size changes (a bound x or w, a Stack re-flowing) instead of jumping. It is not the frame's `transition` (last point).
- `change` on a bound `value`: `swap` lifts the old text out and the new one in (480 ms), `countUp` rolls a Number to its new value (800 ms, formatted as it rolls; on a Text it does nothing — make the score a Number), `flash` pulses a highlight on change (300 ms); leave it out to cut. All three animate on air as in the preview.
- A Ticker moves on its own: `speed` in px per second (80–160 on air, 0 holds it still); its items scroll in a loop.
- Rows of a repeat each enter on their own; a frame can stagger its layers' entrances (the operator sets it in the frame's inspector).
- Frame transitions: how the whole frame comes on air is its own `transition` field, next to `name` and `code` (`scene_set('frame/<id>', { merge: { transition: { kind: "fade", ms: 450 } } })`) — cut · fade · slide · wipe · dip. A take from the live panel or the script can override it for one cut.

Timing on air: straps and badges 300–500 ms, bars and wipes 400–700 ms, exits shorter than entrances (150–250 ms). Keep motion for what changes; a whole frame of movement reads as noise.

A picture over the MCP is one still (`scene_get('frame/<id>', { as: 'image' })`): the motion itself shows in the studio's preview.
