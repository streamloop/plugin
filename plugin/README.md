# Streamloop

Run 24/7 live streams on [Streamloop](https://streamloop.app) from a conversation: loop videos to YouTube, Twitch, Facebook or any RTMP destination, and build live scenes with overlays, live inputs and live data.

## What it contains

- The Streamloop MCP server, `https://mcp.streamloop.app/mcp`. You sign in to your own Streamloop account with OAuth; the plugin holds no keys.
- Skills that teach the assistant how Streamloop works. `streamloop` is the base: which kind of stream fits, the order of setup, and which skill comes next. The others cover video loops and playlists, destinations and multistream, and building live scenes and changing them on air. A last one, `calling-the-streamloop-api`, is for coding agents that write scripts against Streamloop's HTTP API.

## Install

In Claude Code: `/plugin marketplace add streamloop/skills`, then `/plugin install streamloop@streamloop`. In Claude and ChatGPT, add it from the directory.

## Use it

Connect the Streamloop connector, then ask in plain words:

- "Which of my streams are live right now?"
- "Create a 24/7 stream from this video URL and send it to my YouTube channel."
- "Add a lower third with today's headline to my scene, show me a picture, then publish it."

Starting a stream uses credits from your Streamloop workspace. The assistant cannot buy credits or change your plan.

## Data

Requests go to your Streamloop account through `mcp.streamloop.app`, with the access you grant at sign-in. Media you ask it to import is fetched by Streamloop from the URL you give. The plugin runs no code on your machine and sends nothing anywhere else. Privacy policy: https://streamloop.app/privacy

## Support

https://streamloop.app/contact
