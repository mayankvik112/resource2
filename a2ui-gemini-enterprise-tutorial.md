---
title: "A2UI in Gemini Enterprise: Build Agent-Driven UIs with ADK"
date: 2026-10-11
tags: [a2ui, gemini-enterprise, adk, agents, genui, a2a]
sources:
  - https://developers.googleblog.com/introducing-a2ui-an-open-project-for-agent-driven-interfaces/
  - https://a2ui.org/guides/agent-development/
  - https://a2ui.org/quickstart/
  - https://a2ui.org/composer/
  - https://a2ui.org/reference/components/
  - https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/a2ui-component-gallery-reference
  - https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/register-and-manage-an-a2ui-agent
  - https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/tutorial-host-agent-cloud-run
  - https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent
  - https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent
  - https://cloud.google.com/blog/topics/developers-practitioners/guide-to-gemini-enterprise-and-a2ui-integration
  - https://github.com/a2ui-project/a2ui
  - https://github.com/a2ui-project/a2ui/tree/main/samples/agent/adk/custom-components-example/examples/0.9
  - https://medium.com/@visrow/a2ui-components-explained-building-agent-driven-user-interfaces-with-a-declarative-protocol-8f413167c1a9
versions:
  a2ui: v0.8 (legacy), v0.9 (previous stable), v0.9.1 (current)
  a2a: v0.3 (GE streaming)
  adk: v1.2.0+
---

# A2UI in Gemini Enterprise: Build Agent-Driven UIs with ADK

Your agent finds the perfect restaurant. It answers in a wall of text: "We have
availability at 5:00, 5:30, 6:00, 8:30..." — and the user has to type their
answer back. Every option is a turn. Every turn is friction.

**A2UI (Agent-to-UI)** fixes this. Instead of returning text, your agent returns
a JSON description of a UI — a date picker, a list of cards, a multi-select — and
the client renders it natively. Gemini Enterprise ships with a built-in A2UI
renderer, so you build the agent once and it paints rich UI in the GE chat surface
for free.

This tutorial takes you from zero to a working A2UI agent inside Gemini
Enterprise:

1. Understand what A2UI is and where it fits in the stack
2. Learn the component gallery (with real examples)
3. Use the **A2UI Composer** to prototype UI visually
4. Add A2UI to an **ADK** agent
5. **Deploy** to Cloud Run / Agent Runtime
6. **Register** the agent in Gemini Enterprise
7. Troubleshoot the things that actually break

---

## Part 0 — Prerequisites

Before you touch code, line up these.

### Accounts & roles

| Requirement | Why |
| --- | --- |
| Google Cloud project with billing | Cloud Run / Agent Runtime, Discovery Engine |
| Gemini Enterprise **Admin** role | Register agents (also `Discovery Engine Admin`) |
| An existing Gemini Enterprise app | Agents attach to an app |
| A Gemini API key (for Composer only) | Composer's Gemini chat feature |

### APIs to enable

```bash
gcloud services enable \
  aiplatform.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  run.googleapis.com \
  logging.googleapis.com \
  discoveryengine.googleapis.com \
  storage.googleapis.com \
  iam.googleapis.com
```

### Local tools

| Tool | Used for |
| --- | --- |
| `uv` | Running Python ADK agents |
| `gcloud` CLI | Deploy + register |
| Node.js + `corepack` | Running the Lit/Composer renderer clients |
| `git` | Cloning samples |

Install `uv` if you don't have it:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv --version
```

---

## Part 1 — What is A2UI (and where does it sit)?

### The problem

In a multi-agent world, the agent doing the work is often **remote** — a
different server, a different company. It can't touch your DOM. Historically you
"fixed" this by shipping HTML/JS into a sandboxed iframe. That approach is heavy,
visually disjointed, and drags in security headaches (XSS, UI injection).

A2UI's answer: **transmit UI that is safe like data, but expressive like code.**

### The core idea

An agent emits a JSON payload describing:

- a **tree of components** (Card, Text, Button, ChoicePicker, Image, …), and
- a separate **data model** holding the values those components display.

The client owns rendering. It only paints components from a **pre-approved
catalog** it already trusts. No arbitrary code runs. The payload rides inside
whatever transport you already use — here, **A2A JSON-RPC** as `DataPart` objects
with MIME type `application/json+a2ui`.

### Three properties that make it work

| Property | Meaning |
| --- | --- |
| **Declarative, not executable** | It's data. The client renders only catalog components — no code execution, no UI injection. |
| **Streaming-friendly** | A flat list of small JSON messages, so the LLM emits them incrementally and the client paints as they arrive. |
| **Framework-agnostic** | The same payload renders through Lit, Angular, Flutter, or native mobile. The agent doesn't know or care. |

### Where A2UI sits in the stack

Confusion usually comes from conflating four layers. They each own something
different:

| Layer | Owns | Examples |
| --- | --- | --- |
| App experience | Chat shell + conversation state | CopilotKit, AG-UI |
| Pixel drawing | Turning component descriptions into UI | Lit, Flutter, Angular |
| Conversation pipeline | Client ↔ server transport | A2A Protocol |
| **Cargo (data format)** | **The thing describing the UI** | **A2UI** |

**For the Gemini Enterprise path, the stack collapses to two layers you control:**
your A2A endpoint (the agent) and the A2UI cargo it emits. GE *is* the shell, the
renderer, and the transport client. You don't pick a frontend framework to start.

### A2UI vs A2A

- **A2A** = agent ↔ agent communication (tasks, context, tool calls, results).
- **A2UI** = agent ↔ **human** communication (structured UI).

They're complementary. A2UI is the presentation layer for A2A-style systems.

### Version reality check

A2UI evolved fast. Know which revision you're targeting:

- **v0.8** — legacy
- **v0.9** — previous stable
- **v0.9.1** — current

Gemini Enterprise renders the **inline pattern** today (data baked into each
component). See the version diff table in Part 2 for syntax changes.

---

## Part 2 — The A2UI component gallery (with examples)

Gemini Enterprise supports **all standard A2UI components**, rendered in GE's own
styling. There are two catalogs:

- **Material catalog** — Material Design components (richer styling; v0.9 only).
- **Basic catalog** — design-system-agnostic primitives (Row, Column, Card, Text,
  Button, inputs). Works in both v0.9 and v0.8.

> 💡 **Browse them live:** the [A2UI Composer component gallery](https://a2ui-project.github.io/composer/)
> renders every component with a copy-ready JSON snippet. For GE-specific
> components see the [A2UI component gallery reference](https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/a2ui-component-gallery-reference).

### Common properties

Every component shares:

| Property | Required | Description |
| --- | --- | --- |
| `id` | ✅ | Unique identifier within the surface |
| `accessibility` | — | Attributes like labels/roles |
| `weight` | — | Flex-grow when inside a Row/Column |

### Layout components

**Row** — horizontal container:

```json
{
  "id": "toolbar",
  "component": "Row",
  "children": ["btn1", "btn2", "btn3"],
  "justify": "spaceBetween",
  "align": "center"
}
```

**Column** — vertical container:

```json
{
  "id": "content",
  "component": "Column",
  "children": ["header", "body", "footer"],
  "justify": "start",
  "align": "stretch"
}
```

**List** — scrollable list with dynamic templates:

```json
{
  "id": "message-list",
  "component": "List",
  "children": { "componentId": "message-item", "path": "/messages" },
  "direction": "vertical"
}
```

**MaterialCard** (Material catalog) — card container:

```json
[
  { "id": "root", "component": "MaterialCard", "appearance": "outlined", "children": ["card-content"] },
  { "id": "card-content", "component": "MaterialText", "text": "Card content" }
]
```

### Display components

**Text** — with styling variants:

```json
{ "id": "title", "component": "Text", "text": "Welcome to A2UI", "variant": "h1" }
```

Variants: `h1`, `h2`, `h3`, `h4`, `h5`, `caption`, `body`.

**Image**:

```json
{ "id": "hero", "component": "Image", "url": "https://example.com/hero.png", "fit": "cover", "variant": "hero" }
```

**Icon**:

```json
{ "id": "check-icon", "component": "Icon", "name": "check" }
```

**Divider**:

```json
{ "id": "separator", "component": "Divider", "axis": "horizontal" }
```

### Interactive components

**Button** — triggers an agent action:

```json
{
  "id": "submit-btn",
  "component": "Button",
  "child": "submit-text",
  "variant": "primary",
  "action": { "event": { "name": "submit_form" } }
}
```

**TextField**:

```json
{
  "id": "email-input",
  "component": "TextField",
  "label": "Email Address",
  "value": { "path": "/user/email" },
  "textFieldType": "shortText"
}
```

`textFieldType` values: `shortText`, `longText`, `number`, `obscured`, `date`.

**CheckBox**:

```json
{ "id": "terms-checkbox", "component": "CheckBox", "label": "I agree to the terms", "value": { "path": "/form/agreedToTerms" } }
```

**Slider**:

```json
{ "id": "volume", "component": "Slider", "value": { "path": "/settings/volume" }, "minValue": 0, "maxValue": 100 }
```

**DateTimeInput** — the one that ends the restaurant-booking loop:

```json
{ "id": "date-picker", "component": "DateTimeInput", "value": { "path": "/booking/date" }, "enableDate": true, "enableTime": false }
```

**ChoicePicker** (v0.9) / **MultipleChoice** (v0.8) — pick one or many:

```json
{
  "id": "country-select",
  "component": "ChoicePicker",
  "options": [
    { "label": "USA", "value": "us" },
    { "label": "Canada", "value": "ca" }
  ],
  "selections": { "path": "/form/country" },
  "maxAllowedSelections": 1
}
```

### Container components

**Card** (basic catalog):

```json
{ "id": "info-card", "component": "Card", "child": "card-content" }
```

**Modal**, **Tabs**, and Material containers (`MaterialDialog`,
`MaterialExpansionPanel`, `MaterialGridList`) round out the set — see the
gallery reference for full property tables.

### The adjacency list model

Components are a **flat list**; parent/child links are by ID. This is *why* it's
LLM-friendly — the model emits small, independent chunks.

```json
{
  "updateComponents": {
    "surfaceId": "main",
    "components": [
      { "id": "root", "component": "Column", "children": ["title", "content"] },
      { "id": "title", "component": "Text", "text": "Welcome" },
      { "id": "content", "component": "Card", "child": "inner_text" }
    ]
  }
}
```

### Data binding

Bind components to a shared data model using JSON Pointers:

| Form | Example | Meaning |
| --- | --- | --- |
| Static value | `"text": "Hello"` | literal |
| Dynamic binding | `"text": { "path": "/user/name" }` | read from data model |

### v0.8 → v0.9: the syntax that changes

This is the table to keep open. Mixing these up is the #1 source of "why is my
UI blank."

| Aspect | v0.8 | v0.9 |
| --- | --- | --- |
| Component wrapper | `"component": { "Text": { … } }` | `"component": "Text", …props` |
| String values | `{ "literalString": "Hello" }` | `"Hello"` |
| Children | `{ "explicitList": ["a","b"] }` | `["a","b"]` |
| Data binding | `{ "path": "/data" }` | same |
| Text/Image styling | `usageHint` | `variant` |
| Button styling | `"primary": true` | `"variant": "primary"` |
| Action format | `{ "name": "..." }` | `{ "event": { "name": "..." } }` |
| Choice component | `MultipleChoice` | `ChoicePicker` |
| Layout alignment | `distribution`, `alignment` | `justify`, `align` |
| TextField value | `text` | `value` |

### A full message sequence (v0.9.1)

A real render is three messages: create the surface, define the tree, populate data.

```json
{ "version": "v0.9.1",
  "createSurface": {
    "surfaceId": "main",
    "catalogId": "https://a2ui.org/specification/v0_9_1/catalogs/basic/catalog.json"
  }}
```

```json
{ "version": "v0.9.1",
  "updateComponents": {
    "surfaceId": "main",
    "components": [
      {"id": "header", "component": "Text", "text": "# Book Your Table", "variant": "h1"},
      {"id": "date-picker", "component": "DateTimeInput", "label": "Select Date", "value": {"path": "/reservation/date"}, "enableDate": true},
      {"id": "submit-text", "component": "Text", "text": "Confirm Reservation"},
      {"id": "submit-btn", "component": "Button", "child": "submit-text", "variant": "primary", "action": {"event": {"name": "confirm_booking"}}}
    ]
  }}
```

```json
{ "version": "v0.9.1",
  "updateDataModel": {
    "surfaceId": "main",
    "path": "/reservation",
    "value": {"date": "2025-12-15", "time": "19:00", "guests": 2}
  }}
```

> In **v0.8**, `createSurface` was `beginRendering`, components were nested in the
> response, and the data model used an adjacency list instead of flat JSON.

---

## Part 3 — Prototype with the A2UI Composer

Before wiring a real agent, design your UI in the **A2UI Composer**. It's the
fastest feedback loop you'll get.

### What the Composer is

A browser workspace at **[a2ui-project.github.io/composer](https://a2ui-project.github.io/composer/)**
with four panes:

- **Gemini Assistant** — chat that generates/refines layouts in natural language.
  Attach a mock with the paper-clip icon, or screenshot the current UI with the
  camera icon.
- **Rendered A2UI Preview** — the live visual.
- **A2UI JSON Editor** — the raw payload. Edit directly; preview updates instantly.
  Hover for tooltips on each element. Paste JSON from elsewhere and right-click to
  format.
- **Debug tabs** — Data Model, Events, Errors, Raw Messages.

### Step-by-step: build your first widget

1. **Open** <https://a2ui-project.github.io/composer/>.
2. **Set your Gemini API key** — go to Settings (see below). Without it, the
   Gemini chat is disabled, but you can still hand-edit JSON.
3. **Ask Gemini** something like: *"A card listing three restaurants with an
   image, name, rating, and a Book button."*
4. **Watch the preview** update as JSON streams in.
5. **Tweak the JSON** in the editor — change a `variant`, add a `ChoicePicker` —
   and see it re-render.
6. **Copy the JSON** using the copy icon at the top of the Usage panel.

### Getting a Gemini API key

1. Go to <https://aistudio.google.com/api-keys> and sign in.
2. Click **Create API key**; select/create a Google Cloud project.
3. Click **Create key** and save it somewhere safe.

> 🔒 The Composer encrypts your key and stores it locally in your browser's secure
> database using the Web Crypto API. Neither Google nor anyone else can read it.

### Renderer apps

In Settings you can switch the renderer:

- Angular Basic Catalog
- Lit Basic Catalog
- React Basic Catalog
- **Custom** — paste your own renderer app URL (for building your own, see the
  Composer integration guide).

### Reading the Raw Messages tab

This is your debugger. It shows:

- `RENDERER_READY` — renderer bootstrapped
- `A2UI_CATALOG` — full catalog the renderer supports (on request)
- `COMPONENT_USAGES` — data behind the Component Gallery page
- `DATA_MODEL_CHANGE` — the `updateDataModel` message that *would* be sent back to the agent
- `LLM_REQUEST` / `LLM_RESPONSE` — the full prompt and reply (Gemini chat only)

**Why this matters:** the `DATA_MODEL_CHANGE` message is exactly what your agent
will receive when a user taps a widget in production. Design against that contract.

---

## Part 4 — Use A2UI with an ADK agent

Now make an agent emit A2UI. Start simple, then upgrade.

### Step 4.1 — Scaffold an ADK agent

```bash
pip install google-adk
adk create my_agent
```

> If you're working inside the A2UI samples (which already depend on `google-adk`),
> use `uv run adk create my_agent` instead of a global install.

### Step 4.2 — A plain ADK agent (text only)

Edit `my_agent/agent.py`:

```python
import json
from google.adk.agents.llm_agent import Agent
from google.adk.tools.tool_context import ToolContext

def get_restaurants(tool_context: ToolContext) -> str:
    """Call this tool to get a list of restaurants."""
    return json.dumps([
        {
            "name": "Xi'an Famous Foods",
            "detail": "Spicy and savory hand-pulled noodles.",
            "imageUrl": "http://localhost:10002/static/shrimpchowmein.jpeg",
            "rating": "★★★★☆",
            "infoLink": "[More Info](https://www.xianfoods.com/)",
            "address": "81 St Marks Pl, New York, NY 10003"
        },
        {
            "name": "Han Dynasty",
            "detail": "Authentic Szechuan cuisine.",
            "imageUrl": "http://localhost:10002/static/mapotofu.jpeg",
            "rating": "★★★★☆",
            "infoLink": "[More Info](https://www.handynasty.net/)",
            "address": "90 3rd Ave, New York, NY 10003"
        },
    ])

AGENT_INSTRUCTION = """
You are a helpful restaurant finding assistant. Your goal is to help users
find and book restaurants using a rich UI.

1. For finding restaurants:
   a. You MUST call the `get_restaurants` tool. Extract cuisine, location, and
      a specific number (`count`) of restaurants from the query.
   b. After receiving the data, follow the instructions precisely to generate
      the final a2ui UI JSON.
"""

root_agent = Agent(
    model='gemini-2.5-flash',
    name="restaurant_agent",
    description="An agent that finds restaurants and helps book tables.",
    instruction=AGENT_INSTRUCTION,
    tools=[get_restaurants],
)
```

Set your key and run:

```bash
echo 'GOOGLE_API_KEY="YOUR_API_KEY"' > .env
adk web
```

Select `my_agent`, ask about restaurants in New York. You'll get **plain text** —
this is your baseline.

### Step 4.3 — Upgrade to A2UI output

Getting an LLM to emit valid A2UI needs prompt engineering. The SDK gives you
`DirectJsonFormat`, which builds a system prompt including the A2UI schema and
examples from your catalog.

Ensure `a2ui-agent-sdk` is installed (it ships with the samples). Then:

```python
from a2ui.core.basic_catalog import BasicCatalog
from a2ui.inference_formats.direct_json import DirectJsonFormat
from a2ui.schema import VERSION_0_8, VERSION_0_9

ROLE_DESCRIPTION = (
    "You are a helpful restaurant finding assistant. Your final output MUST be a "
    "a2ui UI JSON response."
)

UI_DESCRIPTION = """
- If the query is for a list of restaurants, use the restaurant data you have
  already received from the `get_restaurants` tool to populate the
  `dataModelUpdate.contents` (v0.8) or `updateDataModel.value` (v0.9+) object
  (e.g., for the "items" key).
- If the number of restaurants is 5 or fewer, you MUST use the
  `SINGLE_COLUMN_LIST_EXAMPLE` template.
- If the number of restaurants is more than 5, you MUST use the
  `TWO_COLUMN_LIST_EXAMPLE` template.
- If the query is to book a restaurant (e.g., "USER_WANTS_TO_BOOK..."), you MUST
  use the `BOOKING_FORM_EXAMPLE` template.
- If the query is a booking submission (e.g., "User submitted a booking..."),
  you MUST use the `CONFIRMATION_EXAMPLE` template.
"""

inference_format = DirectJsonFormat(
    [BasicCatalog(VERSION_0_8)],   # use VERSION_0_9 for the newer protocol
    examples_path="examples/0.8",
)

A2UI_AND_AGENT_INSTRUCTION = inference_format.prompt_generator.generate(
    role_description=ROLE_DESCRIPTION,
    ui_description=UI_DESCRIPTION,
    include_schema=True,
    include_examples=True,
    validate_examples=True,
)

root_agent = Agent(
    model='gemini-2.5-flash',
    name="restaurant_agent",
    description="An agent that finds restaurants and helps book tables.",
    instruction=A2UI_AND_AGENT_INSTRUCTION,
    tools=[get_restaurants],
)
```

**Why this works:** `DirectJsonFormat` injects the up-to-date schema plus
validated examples, so the LLM learns what *idiomatic* output looks like for your
catalog.

### Step 4.4 — Understand the output

Your agent now emits **text + a JSON list of A2UI messages**. The standard schema
defines operations like:

- `render` — display a UI
- `update` — change data in an existing UI

Because it's structured JSON, parse and validate before sending to the client:

```python
import json, jsonschema

# 1. Parse
# Warning: parsing LLM output as JSON is fragile and for documentation only.
# LLMs add Markdown fences and make mistakes — rely on frameworks to parse.
parsed_json_data = json.loads(json_string_cleaned)

# 2. Validate against A2UI_SCHEMA — never send malformed UI to a client
jsonschema.validate(instance=parsed_json_data, schema=self.a2ui_schema_object)
```

### Step 4.5 — The three-phase integration (the real pattern)

The full GE integration is three phases. Reference: the
[agent-a2ui-demo](https://github.com/wadave/agent-a2ui-demo) implementation guide.

**Phase A — Define artifacts**

1. **Catalog JSON** — a `catalogId` + `components` map. Each component uses the
   discriminator pattern (a literal `component` const) and references shared
   types:

```json
{
  "catalogId": "https://example.com/my_catalog.json",
  "components": {
    "MyWidget": {
      "type": "object",
      "allOf": [
        { "$ref": "common_types.json#/$defs/ComponentCommon" },
        {
          "type": "object",
          "properties": {
            "component": { "const": "MyWidget" },
            "title": { "$ref": "common_types.json#/$defs/DynamicString" }
          },
          "required": ["component", "title"]
        }
      ],
      "unevaluatedProperties": false
    }
  }
}
```

You only declare **domain-specific** components. The bundled `BasicCatalog`
already provides Row, Column, Card, Text, Button, ChoicePicker, TextField,
Image, Icon, and friends.

2. **Example files** — one JSON file per UI pattern (`list.json`, `detail.json`)
   in an `examples/` directory. Each is an array of A2UI messages showing a
   complete render:

```
createSurface → updateComponents → updateDataModel
```

These get injected into the system prompt — they teach the LLM the shape of a
good response for *your* catalog. **Bad examples produce bad output**, so invest
here.

**Phase B — Wire the agent (schema manager + prompt + toolset)**

```python
from a2ui.schema.manager import A2uiSchemaManager
from a2ui.schema.catalog import CatalogConfig
from a2ui.schema.catalog_provider import FileSystemCatalogProvider
from a2ui.schema.common_modifiers import remove_strict_validation
from a2ui.basic_catalog.provider import BasicCatalog

A2UI_VERSION = settings.a2ui_version  # from env/config — keep it in ONE place

schema_manager = A2uiSchemaManager(
    version=A2UI_VERSION,
    catalogs=[
        CatalogConfig(
            name="my_catalog",
            provider=FileSystemCatalogProvider("catalog_schemas/my_catalog_definition.json"),
            examples_path="examples/my_catalog",
        ),
        BasicCatalog.get_config(version=A2UI_VERSION),
    ],
    accepts_inline_catalogs=True,
    schema_modifiers=[remove_strict_validation],
)
```

Generate the system prompt:

```python
instruction = schema_manager.generate_system_prompt(
    role_description="You are a helpful assistant...",
    workflow_description="1. Analyze the request...",
    ui_description="Use Card for detail views...",
    include_schema=False,
    include_examples=False,   # examples loaded dynamically via session
    validate_examples=False,
)
```

Add one extra tool alongside your domain tools:

```python
from a2ui.adk.send_a2ui_to_client_toolset import SendA2uiToClientToolset

LlmAgent(
    model=model,
    instruction=instruction,
    tools=[
        SendA2uiToClientToolset(
            a2ui_enabled=lambda ctx: ctx.state.get("system:a2ui_enabled", False),
            a2ui_catalog=lambda ctx: ctx.state.get("system:a2ui_catalog"),
            a2ui_examples=lambda ctx: ctx.state.get("system:a2ui_examples"),
        ),
        # ... your domain tools
    ],
)
```

**Why examples load via session state:** the same agent can serve different
catalogs (and message patterns) without rebuilding the prompt.

**Phase C — Runtime + render (executor)**

The executor does four things per request, *before* handing control to the agent:

1. **Detect** which A2UI message pattern the client supports via its
   `X-A2A-Extensions` header.
2. **Fall back** to a sensible default when the header is missing — **Gemini
   Enterprise doesn't send one and expects the inline pattern.**
3. **Select** the matching catalog and load its examples.
4. **Persist** into session state so the toolset lambdas find them next turn.

```python
# app/agent_executor.py — reference implementation
# Uses: try_activate_a2ui_extension, schema_manager.get_selected_catalog,
#       schema_manager.load_examples
# Writes result into ADK session state via EventActions(state_delta=...)
```

> ⚠️ **Catalog gotcha (important):** the catalog must be **self-contained** for
> validation. When the schema manager picks a catalog at startup, it returns the
> first supported catalog standalone — the bundled `BasicCatalog` is **not**
> auto-merged. If your custom catalog only declares its own components but your
> examples reference `Text`/`Column`, validation fails with
> `'/$defs/anyComponent' does not exist`. The reference repo fixes this with a
> small provider (`_MergedBasicCatalogProvider`) that loads `basic_catalog.json`
> and injects custom components into both `components` and
> `$defs.anyComponent.oneOf` before returning.

---

## Part 5 — Deploy the agent

Gemini Enterprise needs a **publicly accessible** A2A endpoint. Two supported
hosts:

| Host | Best for |
| --- | --- |
| **Agent Runtime** (Gemini Enterprise Agent Platform) | Fully managed; ADK-native; single-command GE registration |
| **Cloud Run** | Serverless containers; any framework; most samples use this |

The official sample lives at
`samples/community/agent/adk/gemini_enterprise/v0_9` in the
[A2UI repo](https://github.com/a2ui-project/a2ui). Its structure:

| File/Dir | Purpose |
| --- | --- |
| `agent.py` | Defines the agent, skills, behavior |
| `agent_executor.py` | Execution flow + tool interactions |
| `main.py` | FastAPI app entry point |
| `deploy.sh` | Build + deploy to Cloud Run |
| `examples/` | Component template examples |
| `gemini_enterprise_composite_catalog.json` | Material + custom GE components |
| `tools.py` | Tools the agent can use |
| `prompt_builder.py` | Model prompt construction |

### Step 5.1 — Clone and prepare

```bash
git clone https://github.com/a2ui-project/a2ui.git
cd a2ui/samples/community/agent/adk/gemini_enterprise/v0_9
```

### Step 5.2 — Grant the Discovery Engine service agent Cloud Run Invoker

Only needed if your agent runs on Cloud Run behind a default `.run.app` URL with
authentication:

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-discoveryengine.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

Replace:

- `PROJECT_ID` — your project ID
- `PROJECT_NUMBER` — your project number

### Step 5.3 — Deploy to Cloud Run

```bash
chmod +x deploy.sh
./deploy.sh PROJECT_ID a2ui-demo-agent MODEL_NAME
```

The script builds a container, pushes to Artifact Registry, deploys to Cloud Run,
and sets env vars (including `MODEL` and the public `AGENT_URL`).

Replace:

- `PROJECT_ID` — your project ID
- `MODEL_NAME` — *(optional)* `gemini-2.5-pro` or `gemini-2.5-flash`
  (defaults to `gemini-2.5-flash`)

The script prints the **service URL** — you need it next.

### Step 5.4 (alternative) Deploy to Agent Runtime

If you prefer the fully managed path (recommended for production):

```bash
# Standard deployment: Cloud Console + ADK CLI, step-by-step
#   https://adk.dev/deploy/agent-runtime/deploy/

# Accelerated path with CI/CD + infra-as-code:
adk deploy ...   # see https://adk.dev/deploy/agent-runtime/agents-cli/
```

After deploying, list your agent to get its resource path
(`projects/PROJECT_ID/locations/LOCATION/reasoningEngines/RESOURCE_ID`) — you'll
paste that when registering (Part 6).

> Agent Runtime is a **paid service** beyond the no-cost tier.

---

## Part 6 — Link the agent into Gemini Enterprise

Two registration paths depending on your host.

### Path A — Register an A2A agent (Cloud Run)

**Console method:**

1. Go to the [Gemini Enterprise console](https://console.cloud.google.com/gemini-enterprise/).
2. Click your app → **Agents** → **Add Agents**.
3. Under *Choose an agent type*, click **Add** for **Custom agent via A2A**.
4. Paste the agent card JSON into **Agent card JSON**.
5. Click **Preview agent details → Next**.
6. If the agent needs Google Cloud access on the user's behalf (or your endpoint
   requires an `Authorization` header), enter Client ID / secret / Auth URI /
   Token URI + scopes. Otherwise click **Skip & Finish**.

**REST method:**

```bash
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  https://discoveryengine.googleapis.com/v1alpha/projects/PROJECT_NUMBER/locations/LOCATION/collections/default_collection/engines/ENGINE_ID/assistants/default_assistant/agents \
  -d '{
    "name": "a2ui-demo-agent",
    "displayName": "A2UI v0.9 Demo Agent",
    "description": "A demo agent that showcases A2UI v0.9 UI templates.",
    "a2aAgentDefinition": {
      "jsonAgentCard": "{\"protocolVersion\": \"0.3.0\", \"name\": \"A2UI v0.9 Demo\", \"description\": \"A demo agent that showcases A2UI v0.9 UIs...\", \"url\": \"AGENT_URL\", \"version\": \"1.0.0\", \"capabilities\": {\"streaming\": true, \"preferredTransport\": \"JSONRPC\", \"extensions\": [{\"uri\": \"https://a2ui.org/a2a-extension/a2ui/v0.9\", \"description\": \"Ability to render A2UI v0.9\", \"required\": false, \"params\": {\"supportedCatalogIds\": [\"https://www.gstatic.com/vertexaisearch/a2ui/v0_9/gemini_enterprise_composite_catalog.json\"]}}]}, \"skills\": [{\"id\": \"a2ui_demo\", \"name\": \"A2UI v0.9 Component Demo\", \"description\": \"Demonstrates A2UI v0.9 UIs...\"}], \"defaultInputModes\": [\"text/plain\"], \"defaultOutputModes\": [\"text/plain\"]}"
    }
  }'
```

Replace:

- `PROJECT_NUMBER` — your project number
- `LOCATION` — multi-region of your data store: `global`, `us`, or `eu`
- `ENGINE_ID` — the ID of the app to register with
- `AGENT_URL` — the service URL your deploy step printed

**Note the A2A extension declaration** in `capabilities.extensions`: the `uri` is
`https://a2ui.org/a2a-extension/a2ui/v0.9` and `supportedCatalogIds` points at the
GE composite catalog. This is how GE knows your agent can render A2UI.

**Unregister (optional):**

```bash
curl -X DELETE \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  https://discoveryengine.googleapis.com/v1alpha/projects/PROJECT_NUMBER/locations/LOCATION/collections/default_collection/engines/ENGINE_ID/assistants/default_assistant/agents/AGENT_ID
```

### Path B — Register an ADK agent (Agent Runtime)

ADK agents on Agent Runtime register in **one CLI command** via
[Agents CLI in Agent Platform](https://google.github.io/agents-cli).

**Console method:**

1. Gemini Enterprise console → your app → **Agents** → **Add agent**.
2. Click **Add** for **Custom agent via Agent Runtime**.
3. *(Optional)* Add authorizations for resources the agent accesses on the user's
   behalf (Client ID, secret, Token URI, Authorization URI; Done).
4. Enter an **Agent name** and a **description** (the LLM uses this to decide
   whether to invoke your agent).
5. Enter the **Agent Runtime resource path**:
   `projects/PROJECT_ID/locations/LOCATION/reasoningEngines/RESOURCE_ID`
6. Click **Create**.

**REST method:**

```bash
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: PROJECT_ID" \
  "https://ENDPOINT_LOCATION-discoveryengine.googleapis.com/v1alpha/projects/PROJECT_ID/locations/global/collections/default_collection/engines/APP_ID/assistants/default_assistant/agents" \
  -d '{
    "displayName": "DISPLAY_NAME",
    "description": "DESCRIPTION",
    "icon": { "uri": "ICON_URI" },
    "adkAgentDefinition": {
      "provisionedReasoningEngine": {
        "reasoningEngine": "projects/PROJECT_ID/locations/RESOURCE_LOCATION/reasoningEngines/RESOURCE_ID"
      }
    },
    "authorizationConfig": {
      "toolAuthorizations": ["projects/PROJECT_NUMBER/locations/global/authorizations/AUTH_ID"]
    }
  }'
```

Replace: `ENDPOINT_LOCATION` (`us`/`eu`/`global`), `PROJECT_ID`,
`PROJECT_NUMBER`, `APP_ID`, `DISPLAY_NAME`, `DESCRIPTION`, `ICON_URI`,
`RESOURCE_ID`, `RESOURCE_LOCATION`, `AUTH_ID`.

### Step 6.3 — Use the agent in the web app

1. Gemini Enterprise console → your app → **Integrations** → copy
   **The link to your web app** and share it.
2. Open the web app URL → navigation menu → **Agents**.
3. Under **From your organization**, click your agent.
4. Ask something like *"Show me the recent orders table"* — the agent renders the
   list using custom UI components in chat.

### Authentication headers GE sends

| Header | Purpose |
| --- | --- |
| `X-Serverless-Authorization` | Service-to-service IAM auth (Cloud Run only; sent only for default `.run.app` URLs) |
| `Authorization` | End-user OAuth 2.0 token (only if you configure OAuth) |

> GE does **not** use your agent card's `securitySchemes`/`security` fields, and
> doesn't support `apiKey` or static custom headers. Secure with OAuth or a
> gateway (e.g. Apigee) instead.

---

## Part 7 — Troubleshooting & common pitfalls

### Validation

- **`'/$defs/anyComponent' does not exist`** → your catalog isn't self-contained.
  Merge `BasicCatalog` into your custom catalog (components **and**
  `$defs.anyComponent.oneOf`). See the catalog gotcha in Part 4.5.
- **Blank UI** → check the **Errors** tab in Composer; usually v0.8/v0.9 syntax
  mixed (see the diff table).
- **Malformed UI never reached the client** → good — that means validation
  worked. LLM output frequently has Markdown fences; parse defensively.

### Local dev

- **Port 5173 in use** → the dev server auto-picks the next port; check terminal output.
- **`ERR_CONNECTION_REFUSED` on startup** → known race: the web client starts
  before the Python agent. Wait a few seconds and refresh.
- **Missing API key** → `echo $GEMINI_API_KEY` to verify; re-export if needed.
- **`uv: command not found`** → install `uv` (see Part 0).

### Gemini Enterprise runtime

- **GE shows nothing** → you're likely emitting the decoupled pattern; GE expects
  the **inline** pattern today. Confirm your `X-A2A-Extensions` fallback.
- **Auth fails on Cloud Run** → the Discovery Engine service agent needs
  `roles/run.invoker` (Part 5.2). This only applies to default `.run.app` URLs.
- **Custom components render, basics don't** → your frontend `customCatalog` must
  *also* include the basic components. The backend stamps the merged catalog with
  the custom `catalogId`, so the frontend must mirror that merge.

### Version discipline

- Keep the A2UI version string in **one** config place. Swapping revisions should
  be a one-line change.
- GE supports **A2A v0.3 streaming**. On v1.0.0+, use the SDK compatibility packages
  (`a2a.compat.v0_3` in Python, `a2acompat/a2av0` in Go).

---

## Part 8 — What's next

- **Spec & component reference:** <https://a2ui.org/>
- **Composer:** <https://a2ui-project.github.io/composer/>
- **Samples repo:** <https://github.com/a2ui-project/a2ui>
- **Custom components example (v0.9):** `samples/agent/adk/custom-components-example/examples/0.9`
  (contains `contact_card.json`, `contact_list.json`, `org_chart.json`,
  `floor_plan.json`, `multi_surface.json`, `chart_node_click.json`,
  `action_confirmation.json`)
- **GE + A2UI integration blog:** [Guide to Gemini Enterprise and A2UI integration](https://cloud.google.com/blog/topics/developers-practitioners/guide-to-gemini-enterprise-and-a2ui-integration)
- **Cloud Run host tutorial:** [Host an A2UI agent with Cloud Run](https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/tutorial-host-agent-cloud-run)
- **Reference implementation:** <https://github.com/wadave/agent-a2ui-demo>

**The takeaway:** in Gemini Enterprise you only build the agent and the A2UI it
emits. GE owns the shell, the renderer, and the transport. The next time a user
asks for "a table for two tomorrow at 7pm," the answer can be a date picker
instead of another question.

---

## Sources

1. Introducing A2UI — <https://developers.googleblog.com/introducing-a2ui-an-open-project-for-agent-driven-interfaces/>
2. Agent Development — <https://a2ui.org/guides/agent-development/>
3. Quickstart — <https://a2ui.org/quickstart/>
4. A2UI Composer — <https://a2ui.org/composer/>
5. Component Gallery — <https://a2ui.org/reference/components/>
6. GE Component Gallery Reference — <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/a2ui-component-gallery-reference>
7. Register A2UI + A2A agent — <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/register-and-manage-an-a2ui-agent>
8. Host an A2UI agent with Cloud Run — <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/tutorial-host-agent-cloud-run>
9. Register A2A agents — <https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent>
10. Register ADK agents on Agent Runtime — <https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent>
11. GE + A2UI integration guide — <https://cloud.google.com/blog/topics/developers-practitioners/guide-to-gemini-enterprise-and-a2ui-integration>
12. A2UI repo — <https://github.com/a2ui-project/a2ui>
13. Custom components example (v0.9) — <https://github.com/a2ui-project/a2ui/tree/main/samples/agent/adk/custom-components-example/examples/0.9>
14. A2UI Components Explained — <https://medium.com/@visrow/a2ui-components-explained-building-agent-driven-user-interfaces-with-a-declarative-protocol-8f413167c1a9>
