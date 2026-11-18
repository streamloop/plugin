---
name: shaping-data
description: "Creates and changes the tables of a Streamloop scene through the Streamloop MCP — typed-in rows, CSV/JSON/XLSX files and URLs, Google Sheets, elements picked from a web page — and formulas that filter, sort, page or join them. Use for \"add a table of…\", \"import this sheet\", \"show the price from this API\", \"only the top 5\", \"three at a time\" on a Streamloop scene."
---
# Shaping data

> With the Streamloop MCP: a scene is reached by its id, `sceneId` (`scn_…`): `stream_get` answers it for a scene stream, and `stream_create { name, kind: "scene" }` makes a new one. Every `scene_*` tool takes it. A call written `scene_get('frame/intro', { as: 'image' })` is `scene_get { sceneId, path: "frame/intro", as: "image" }`, and `scene_set('frame/intro', { replace: … })` is `scene_set { sceneId, path: "frame/intro", replace: … }`: `replace` is the whole resource, `merge` only the fields to change (a merge patch, or one text edit `{ old, new }`). To have a change refused when someone edited the resource meanwhile, add `basedOnVersionId`: the `revision` you read. Every set changes the draft only; `scene_publish { sceneId }` puts the whole draft on air at once on every stream that plays the scene, so tell the user what will change first. Starting one from nothing: skill building-scenes.

A table is what the show reads at `data.<id>.rows` (media sources are at `sources.<id>`). Give every row an `id`. Rows keep their nesting: `row.meta.price` and `data.quote.rows[0].regularMarketPrice` bind and type-check like any column.

## A typed-in table
```
scene_set('source/data/guests', { replace: { name: "Guests", description: "Tonight's guests, for the lower thirds", connector: "inline",
  rows: [{ id: "g1", name: "Maria Lopez", title: "Climate correspondent" }, { id: "g2", name: "…", title: "…" }] } })
```
Change rows later with `merge: { rows: [...] }` — a merge patch replaces the whole list.

## From a URL or a web page
1. There is no probe over the Streamloop MCP: the set below is how you look, and nothing reaches viewers before a publish. For `json`, set the table without `pick` first and its answer shows the document's shape; then `merge: { pick }` (or `connector`, `sheet`) and read the answer again, until the rows are right.
2. `scene_set('source/data/<id>', { replace: { name, description, connector, url, pick, refreshSeconds } })`. The answer says what arrived — the status, how many rows and their columns (nesting shown: enough to bind) — or why nothing did. If it says the fetch is still running, read `source/data/<id>/status`; the row values themselves are `scene_get('source/data/<id>/rows')`.
- `connector`: `json`, `csv` or `xlsx` with `url`; `sheets` with `url` (and `sheet`, the tab); `html` with `url` and `pick`; `file` with `file` (its path in the show folder).
- `sheets`: a sheet shared with anyone with the link just works; a private one is read by the studio's service account, and when Google refuses the error names the account — tell the operator to share the sheet with it (Viewer), then set again.
- `pick` on `json`: a path into the document — an array there is the rows, an object one row: `pick: "chart.result[0].meta"`. Without it the document itself must be an array of rows or one object.
- `pick` on `html`: a CSS selector — each matching element is a row `{ id, text, href?, src? }`.
- `refreshSeconds` re-fetches (5 or more; 0 = once; 60 suits a sheet, 10–30 a feed that moves). Write a formula against the types the set's answer shows for each column (a sheet's TRUE arrives as a boolean, a number as a number, a date as the text the cell holds). Fetches go out from Streamloop's servers, as on air: a feed that needs no key works, CORS doesn't matter.
- `headers` sends request headers with every fetch, e.g. `{ Accept: "application/json" }`. A key (Authorization, X-Api-Key, …) is typed by the operator in the table's settings in the Streamloop studio (Headers), saved as a secret, and the header reads `"$secret:<name>"` — ask them to; you never see or write a key. Header values other than secret names read as hidden, and so does a key in the URL's query (`?api_key=(hidden)`): the operator types those in the table's settings too, and a URL you set keeps them as they are.
- Keys are never guessed: no `apikey=demo`, no made-up or borrowed tokens, in a URL or a header — set refuses them. When a feed needs a key, ask the operator to type it in the table's settings in the studio, and stop there. Two failures on one idea: stop, remove the source, ask.

A price card from a public quote feed:
```
scene_set('source/data/nvda', { replace: { name: "NVDA quote", description: "NVIDIA's price for the market card", connector: "json",
  url: "https://query1.finance.yahoo.com/v8/finance/chart/NVDA?interval=1d&range=1d", pick: "chart.result[0].meta", refreshSeconds: 15 } })
<Text id="price" value={number(data.nvda.rows[0].regularMarketPrice, "0,0.00")} … />
```

## An MCP server
A table can be one tool of an MCP server, called on every refresh.
1. The server's tools can't be listed over the Streamloop MCP (that is the studio's probe). Use a tool name and arguments the user or the server's documentation gives — never guess them; set checks the args against the tool's input schema. Otherwise ask the operator to add the table in the Streamloop studio (Sources → Add → MCP server), where its tools are listed, and take it from there with get and `merge`.
2. `scene_set('source/data/<id>', { replace: { name, description, connector: "mcp", url, tool, args, pick, refreshSeconds } })`.
- `pick`: a path into the tool's result, as for `json` (`data` for a Stripe list).
- A result that is text (no structured content from the server) becomes rows `{ id, text }`, one per text part — unless that text is JSON, which reads like a feed.
- Arguments from the show: an arg's value may be bound like a prop, `{ bind: "controls.repo" }` or `{ template: "weather in {{ state.city }}" }`. The table reads again when a bound value changes, and waits (status `waiting`) while a bind has no value. E.g. a `repo` text control (`control/repo`) and `args: { repoName: { bind: "controls.repo" } }`: the operator types a repo, the table follows. `set` checks the args against the tool's own input schema and names the mismatch (`args: limit: expected number`).
- Writes: a tool that changes something (create, update, delete, send, …) is never a table: set refuses it. Running one once is for the studio's assistant, with the operator approving each call.
- Sign-in: `headers` as for URLs. An API key is a secret the operator types in the table's settings. A server that wants OAuth needs the operator to press Connect in the table's settings in the studio: ask them, and stop. After it the header reads `Authorization: "$secret:<table>-authorization"`; you never see or write a token.

DeepWiki, no sign-in:
```
scene_set('source/data/reactWiki', { replace: { name: "React wiki", description: "The React wiki's pages, for the topics card", connector: "mcp",
  url: "https://mcp.deepwiki.com/mcp", tool: "read_wiki_structure", args: { repoName: "facebook/react" }, refreshSeconds: 0 } })
```
Stripe, with a sign-in: ask the operator to add it (Sources → Add → MCP server, `https://mcp.stripe.com`) and press Connect (or paste an API key in Headers), and stop. Then get the table they made: its `headers` carry the `$secret:` reference, and `merge` changes its `args` and `pick`. Stripe's server has no per-resource tools: its data comes through `stripe_api_read` (`args: { stripe_api_operation_id, parameters }`, any GET of the Stripe API: `GetCharges`, `GetPaymentIntents`, `GetInvoices`, `GetSubscriptions`, `GetCustomers`; a list answers `{ data: [...] }`, so `pick: "data"`) and `stripe_analytics` (`args: { intent, params }`, revenue and metrics questions); `stripe_api_search` finds an operation id by intent.
```
scene_set('source/data/revenue', { replace: { name: "Stripe charges", description: "The last ten charges, for the revenue ticker", connector: "mcp", url: "https://mcp.stripe.com",
  headers: { Authorization: "$secret:revenue-authorization" }, tool: "stripe_api_read", args: { stripe_api_operation_id: "GetCharges", parameters: { limit: 10 } }, pick: "data", refreshSeconds: 60 } })
<Text value={number(data.revenue.rows[0].amount / 100, "0,0.00")} … />   // amounts are in the currency's minor unit
```

## A formula
A pure TypeScript function of declared inputs; it re-runs when one changes. It is also how a fetched table takes a new shape — typed from the inferred columns, nested ones too.
```
scene_set('source/derived/topStories', { replace: { name: "Top stories", description: "The first three headlines, for the ticker",
  formula: "export default derive({ from: ['stories'] }, ({ stories }) => stories.rows.slice(0, 3))" } })
scene_set('source/derived/nvdaCard', { replace: { name: "NVDA card", description: "Price and change since the previous close",
  formula: "export default derive({ from: ['nvda'] }, ({ nvda }) => { const q = nvda.rows[0]; return [{ id: 'nvda', price: q.regularMarketPrice, change: q.regularMarketPrice - q.chartPreviousClose }] })" } })
```
- Inputs: `from` (table ids), `state`, `controls`, `show`, `clock` (dotted paths). Only declared inputs are visible, as the second argument: `derive({ from: ['stories'], state: ['page'] }, ({ stories }, { state }) => stories.rows.slice(state.page * 3, state.page * 3 + 3))`.
- Return rows, or `{ rows, …fields }`. No network, timers or randomness.
- Frames read it like any table: `data.topStories.rows`.
- Only for a new shape: filtering, sorting, paging, joining, combining. A column the rows already have binds as it is (`data.odds.rows[0].lastTradePrice`, formatted in the binding) — a formula that renames or converts fields is a step the show doesn't need.
