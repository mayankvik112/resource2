---
title: "A2UI in Gemini Enterprise: Build Agent-Driven UIs with ADK (Grounded)"
date: 2026-10-11
tags: [a2ui, gemini-enterprise, adk, agents, genui, a2a]
grounded_with: "Google Developer Knowledge MCP server (search_documents / get_documents / answer_query)"
sources:
  - https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/register-and-manage-an-a2ui-agent
  - https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/a2ui-component-gallery-reference
  - https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/tutorial-host-agent-cloud-run
  - https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent
  - https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent
  - https://a2ui.org/
  - https://a2ui.org/a2a-extension/a2ui/v0.9
  - https://a2ui-composer.ag-ui.com/basic-catalog
  - https://github.com/a2ui-project/a2ui
versions:
  a2ui: "GE supports v0.9 and v0.8 (A2UI itself is in Early Stage Public Preview)"
  a2a: "0.3.x (GE streaming; agent card protocolVersion 0.3.0)"
  adk: "google-adk 1.x (Python 3.10+)"
  gemini-enterprise: "Agent Runtime on Gemini Enterprise Agent Platform, or Cloud Run"
---

# A2UI in Gemini Enterprise: Build Agent-Driven UIs with ADK

Your agent finds the perfect restaurant. It answers in a wall of text: "We have
availability at 5:00, 5:30, 6:00, 8:30..." — and the user has to type their
answer back. Every option is a turn. Every turn is friction.

**A2UI (Agent-to-UI)** fixes this. Instead of returning text, your agent returns
a JSON description of a UI — a date picker, a list of cards, a multi-select — and
the client renders it natively. **Gemini Enterprise supports all the standard
A2UI components, rendered using Gemini Enterprise's built-in styling**, so you
build the agent once and it paints rich UI in the GE chat surface for free.

> 📚 **Grounded revision.** Every Google-product claim below is grounded against
> official Google documentation (Gemini Enterprise docs on `docs.cloud.google.com`,
> ADK docs on `adk.dev`, and the A2UI project), retrieved via the Google Developer
> Knowledge MCP server. Where the source tutorial leaned on non-Google sources or
> stale version numbers, this copy follows the GCP docs and flags the change with
> **🔄 Grounded**.

This tutorial takes you from zero to a working A2UI agent inside Gemini
Enterprise:

1. Understand what A2UI is and where it fits in the stack
2. Learn the component gallery (with real, doc-accurate examples)
3. Prototype UI visually
4. Add A2UI to an **ADK** agent
5. **Deploy** to Cloud Run / Agent Runtime
6. **Register** the agent in Gemini Enterprise
7. Troubleshoot the things that actually break

> ⚠️ **Launch stage.** Per Google, the development and use of an Agent2UI (A2UI)
> agent is in **Early Stage Public Preview**; the A2A agent registration feature is
> likewise marked **Pre-GA**. Pre-GA features are "as is" and may have limited
> support. Sources: Google Workspace Chat A2UI quickstart; Gemini Enterprise
> "Register and manage A2A agents".

---

## Part 0 — Prerequisites

Before you touch code, line up these.

### Accounts & roles

| Requirement | Why |
| --- | --- |
| Google Cloud project with billing | Cloud Run / Agent Runtime, Discovery Engine |
| **Discovery Engine Admin** role (`roles/discoveryengine.admin`) | Register agents in Gemini Enterprise |
| An existing Gemini Enterprise app | Agents attach to an app |
| A Gemini API key (Composer's Gemini chat only, if you use it) | Composer design assistant |

> 🔄 **Grounded — the exact role.** The GE A2UI Cloud Run tutorial lists the
> **Discovery Engine Admin** role as the "Before you begin" requirement for
> registering the agent. Source: "Host an A2UI agent with Cloud Run".

### APIs to enable

Google's own A2UI Cloud Run tutorial enables exactly these eight APIs — this list
matches verbatim:

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

> 🔄 **Grounded — API list.** Source: "Host an A2UI agent with Cloud Run" (the
> docs label `aiplatform.googleapis.com` as the "Vertex AI API"; the platform is
> now branded the Gemini Enterprise Agent Platform).

### Local tools

| Tool | Used for |
| --- | --- |
| `uv` (optional) | Running Python ADK agents / samples |
| `gcloud` CLI | Deploy + register |
| Node.js + `corepack` | Running the Lit/Angular gallery & renderer clients |
| `git` | Cloning samples |

Install `uv` if you don't have it:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv --version
```

> 🔄 **Grounded — Yarn for the repo samples.** The GE component gallery notes the
> in-repo sample apps require **Yarn** (`yarn start gallery`) via Corepack
> workspaces; for your *own* standalone projects, use any package manager
> (`npm`, `pnpm`, …).

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
whatever transport you already use — here, **A2A** as `DataPart` objects with MIME
type `application/json+a2ui`.

### Three properties that make it work

| Property | Meaning |
| --- | --- |
| **Declarative, not executable** | It's data. The client renders only catalog components — no code execution, no UI injection. |
| **Streaming-friendly** | A flat list of small JSON messages, so the LLM emits them incrementally and the client paints as they arrive. |
| **Framework-agnostic / transport-agnostic** | The same payload renders through Lit, Angular, Flutter, or native mobile. ADK's A2UI docs describe it as "transport-agnostic." The agent doesn't know or care. |

### Where A2UI sits in the stack

Confusion usually comes from conflating four layers. They each own something
different:

| Layer | Owns | Examples |
| --- | --- | --- |
| App experience | Chat shell + conversation state | Gemini Enterprise, CopilotKit, AG-UI |
| Pixel drawing | Turning component descriptions into UI | Lit, Flutter, Angular, Jetpack Compose |
| Conversation pipeline | Client ↔ server transport | A2A Protocol (also AG-UI, MCP, REST, WebSockets) |
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

- **v0.8** — earlier revision. In v0.8 the *basic* catalog is called the
  **"standard catalog"**.
- **v0.9** — current revision supported by Gemini Enterprise.

> 🔄 **Grounded — GE supports v0.9 and v0.8.** Both the GE "Register and manage
> agents using A2UI and A2A" page and the GE "Host an A2UI agent with Cloud Run"
> tutorial state plainly: **"Gemini Enterprise supports A2UI versions v0.9 and
> v0.8."** The original drafted a `v0.9.1` "current" version; treat v0.9 as the
> version to target for GE, and check `https://a2ui.org/#specification-versions`
> for the latest spec revision. Note also that the canonical basic-catalog id was
> realigned from `.../v0_9_1/...` back to `.../v0_9/...` "to align with the canonical
> A2UI protocol specification."

Gemini Enterprise renders the **inline pattern** today (data baked into each
component). See the version diff table in Part 2 for syntax changes.

---

## Part 2 — The A2UI component gallery (with examples)

Gemini Enterprise supports **all standard A2UI components**, rendered in GE's own
styling. There are two catalogs:

- **Material catalog** — Material Design components (`MaterialCard`,
  `MaterialText`, `MaterialButton`, `MaterialTabs`, `MaterialTable`,
  `MaterialDialog`, …). **Supported in v0.9 only.**
- **Basic catalog** — design-system-agnostic primitives (Row, Column, Card, Text,
  Button, inputs). **Supported in both v0.9 and v0.8.** (Called the *standard
  catalog* in v0.8.)

> 💡 **Browse them live:** the GE docs link the A2UI Composer catalogs at
> <https://a2ui-composer.ag-ui.com/basic-catalog> and
> <https://a2ui-composer.ag-ui.com/custom-catalog>. The authoritative per-component
> property tables are the [A2UI component gallery reference](https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/a2ui-component-gallery-reference).

> 🔄 **Grounded — catalog facts.** Source: GE "A2UI component gallery reference".
> The original used `a2ui-project.github.io/composer/`; the GE docs link
> `a2ui-composer.ag-ui.com/...`, so this copy uses the docs' links.

### Common properties

Every **basic-catalog** component shares:

| Property | Required | Description |
| --- | --- | --- |
| `id` | ✅ Yes | Unique identifier within the current surface |
| `accessibility` | — No | Attributes to improve accessibility (labels, roles) |
| `weight` | — No | In `Row`/`Column` layouts, how much space the component takes relative to siblings (like `flex-grow`) |

> 🔄 **Grounded — exact wording.** Source: GE "A2UI component gallery reference"
> (basic catalog common properties).

### Layout components (basic catalog, v0.9)

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

> 🔄 **Grounded — layout props.** In v0.9 the distribution/alignment props are
> `justify` / `align`; in v0.8 they are `distribution` / `alignment`. `children` in
> v0.9 is an array of component IDs or a template definition; in v0.8 it is an
> `explicitList` or a `template`. Source: GE component gallery reference.

### Display components

**Text** — with styling variants:

```json
{ "id": "title", "component": "Text", "text": "Welcome to A2UI", "variant": "h1" }
```

Basic-catalog variants: `h1`, `h2`, `h3`, `h4`, `h5`, `caption`, `body`.
*(In v0.8 the property is `usageHint`; in the **Material** catalog the equivalent
hint is `usageHint` with values like `h1`, `subtitle1`, `body1`, `caption`.)*

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

### Interactive components (basic catalog)

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

> 🔄 **Grounded — component examples are doc-accurate.** Every JSON blob above is
> taken directly from the GE "A2UI component gallery reference" (basic catalog
> sections), which gives both v0.9 and v0.8 forms for each. Source:
> <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/a2ui-component-gallery-reference>

### Container components

**Card** (basic catalog):

```json
{ "id": "info-card", "component": "Card", "child": "card-content" }
```

**Modal** (basic catalog):

```json
{
  "id": "confirmation-modal",
  "component": "Modal",
  "entryPointChild": "open-modal-btn",
  "contentChild": "modal-content"
}
```

**Tabs** (basic catalog):

```json
{
  "id": "settings-tabs",
  "component": "Tabs",
  "tabItems": [
    { "title": "General", "child": "general-tab" },
    { "title": "Privacy", "child": "privacy-tab" }
  ]
}
```

The **Material catalog** adds `MaterialCard`, `MaterialColumn`, `MaterialRow`,
`MaterialDialog`, `MaterialExpansionPanel`, `MaterialGridList`, `MaterialBadge`,
`MaterialProgressBar`, `MaterialTable`, `MaterialButton`, `MaterialTabs`,
`MaterialDatepicker`, `MaterialSelect`, and more — see the gallery reference for
full property tables.

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
| Children | `{ "explicitList": ["a","b"] }` (or `template`) | `["a","b"]` (or template definition) |
| Data binding | `{ "path": "/data" }` | same |
| Text/Image styling | `usageHint` | `variant` |
| Button styling | `"primary": true` | `"variant": "primary"` |
| Action format | `{ "name": "..." }` | `{ "event": { "name": "..." } }` |
| Choice component | `MultipleChoice` | `ChoicePicker` |
| Layout alignment | `distribution`, `alignment` | `justify`, `align` |
| TextField value | `text` | `value` |
| Basic catalog name | "standard catalog" | "basic catalog" |

> 🔄 **Grounded — the diff table.** Every row is confirmed by the paired v0.9/v0.8
> JSON in the GE component gallery reference, except the catalog-name row, which the
> reference states explicitly ("The basic catalog is referred to as the 'standard
> catalog' in v0.8").

### A full message sequence

A real render is three messages: create the surface, define the tree, populate data.

```json
{ "version": "v0.9",
  "createSurface": {
    "surfaceId": "main",
    "catalogId": "https://a2ui.org/specification/v0_9/catalogs/basic/catalog.json"
  }}
```

```json
{ "version": "v0.9",
  "updateComponents": {
    "surfaceId": "main",
    "components": [
      {"id": "header", "component": "Text", "text": "Book Your Table", "variant": "h1"},
      {"id": "date-picker", "component": "DateTimeInput", "label": "Select Date", "value": {"path": "/reservation/date"}, "enableDate": true},
      {"id": "submit-text", "component": "Text", "text": "Confirm Reservation"},
      {"id": "submit-btn", "component": "Button", "child": "submit-text", "variant": "primary", "action": {"event": {"name": "confirm_booking"}}}
    ]
  }}
```

```json
{ "version": "v0.9",
  "updateDataModel": {
    "surfaceId": "main",
    "path": "/reservation",
    "value": {"date": "2025-12-15", "time": "19:00", "guests": 2}
  }}
```

> 🔄 **Grounded — spec version strings.** Google's A2UI docs point at the v0.9
> basic-catalog schema (`specification/v0_9/catalogs/basic/catalog.json`) and the
> canonical id was realigned from `v0_9_1` to `v0_9`. So this copy writes `"v0.9"`
> and the `v0_9` catalog path; confirm the current string at
> <https://a2ui.org/#specification-versions>. The *message shapes*
> (`createSurface` / `updateComponents` / `updateDataModel`) follow the A2UI
> protocol; the exact `version` tokens should be read from the spec you target.

> In **v0.8**, `createSurface` was `beginRendering`, components were nested in the
> response, and the data model used an adjacency list instead of flat JSON.

---

## Part 3 — Prototype with a visual composer

Before wiring a real agent, design your UI in the composer. It's the fastest
feedback loop you'll get.

> ⚠️ **Grounding note.** The A2UI *Composer* tool itself (its panes, its Gemini API
> key handling, its Settings/renderer switcher, and its `Raw Messages` tabs) is
> **not documented in the Google documentation corpus**, so those specifics can't
> be independently verified here and should be treated as tool-specific. What *is*
> documented by Google is that GE links the A2UI Composer catalogs at
> <https://a2ui-composer.ag-ui.com/basic-catalog> and
> <https://a2ui-composer.ag-ui.com/custom-catalog>. For the composer's own UI, see
> <https://a2ui.org/> and the A2UI repo.

### What you can verify from Google docs

- GE supports **all standard A2UI components**, rendered in GE's built-in styling.
- The **basic catalog** examples and the **custom catalog** are browsable via the
  composer links above.
- The Material catalog's components are documented in the GE component gallery
  reference.

### Renderer apps

The GE docs indicate you can see components live via the in-repo sample clients
(the gallery runs with `cd samples/client/angular && yarn start gallery`, using
Yarn/Corepack workspaces).

> 🔒 For any composer-embedded API key handling, follow the tool's own
> documentation; Google's docs don't specify it.

---

## Part 4 — Use A2UI with an ADK agent

Now make an agent emit A2UI. Start simple, then upgrade.

### Step 4.1 — Scaffold an ADK agent

```bash
pip install google-adk            # Python 3.10+ required
adk create my_agent
```

> 🔄 **Grounded — Python version.** ADK's Python quickstart requires **Python 3.10
> or later**. Source: "Python Quickstart for ADK".

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

> 🔄 **Grounded — model ids + auth.** Google's ADK deployment samples use
> `gemini-2.5-flash` / `gemini-2.5-pro`. For a Google Cloud (Agent Platform) path
> you'd authenticate with ADC / a service account instead of `GOOGLE_API_KEY`; for
> the Gemini Developer API a key is fine. Source: GE "Host an A2UI agent with Cloud
> Run" (model values), ADK "Connect to Google Cloud and Agent Platform".

### Step 4.3 — Upgrade to A2UI output

Getting an LLM to emit valid A2UI needs prompt engineering: build a system prompt
that includes the A2UI schema and validated examples from your catalog. The A2UI
Python SDK's `DirectJsonFormat` does this.

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
    [BasicCatalog(VERSION_0_9)],   # GE targets v0.9 (or v0.8)
    examples_path="examples/0.9",
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

> ⚠️ **Grounding note.** `a2ui-agent-sdk` internals such as `DirectJsonFormat`,
> `A2uiSchemaManager`, and `SendA2uiToClientToolset` are **not described in the GCP
> documentation corpus**. Google's ADK A2UI page does document the *schema-manager*
> flow (`A2uiSchemaManager`, `BasicCatalog`, `generate_system_prompt(...)`), so
> treat the SDK class names above as consistent with, but not literally quoted from,
> GCP docs. Prefer the ADK-documented form in Part 4.5 when in doubt.

**Why this works:** injecting the up-to-date schema plus validated examples teaches
the LLM what *idiomatic* output looks like for your catalog — which is exactly the
mechanism Google's ADK A2UI docs describe for `A2uiSchemaManager.generate_system_prompt`.

### Step 4.4 — Understand the output

Your agent now emits **text + a JSON list of A2UI messages**. The schema defines
operations such as `createSurface` / `updateComponents` / `updateDataModel`.
Because it's structured JSON, parse and validate before sending to the client:

```python
import json, jsonschema

# 1. Parse
# Warning: parsing LLM output as JSON is fragile and for documentation only.
# LLMs add Markdown fences and make mistakes — rely on frameworks to parse.
parsed_json_data = json.loads(json_string_cleaned)

# 2. Validate against the catalog schema — never send malformed UI to a client
jsonschema.validate(instance=parsed_json_data, schema=self.a2ui_schema_object)
```

> 🔄 **Grounded — always validate.** ADK's A2UI docs: *"Always validate the LLM's
> JSON output before sending it to the client,"* using the active catalog's
> validator. Source: <https://adk.dev/integrations/a2ui/>.

### Step 4.5 — The schema-manager pattern (the documented integration)

The ADK-documented way to wire an A2UI-capable agent uses a **schema manager** that
loads catalogs, generates the system prompt, and hands you a validator:

```python
from a2ui.core.schema.manager import A2uiSchemaManager
from a2ui.basic_catalog.provider import BasicCatalog

schema_manager = A2uiSchemaManager(
    catalogs=[BasicCatalog.get_config(examples_path="examples")],
)

instruction = schema_manager.generate_system_prompt(
    role_description="You are a helpful restaurant assistant.",
    workflow_description="Analyze the request and return structured UI when appropriate.",
    ui_description="Use Card for detail views; List for results.",
    include_schema=True,
    include_examples=True,
    allowed_components=["Heading", "Text", "Card", "Button", "Table"],
)
```

Custom catalogs (mix the Basic Catalog with a domain catalog from a JSON path):

```python
from a2ui.core.schema.manager import CatalogConfig

schema_manager = A2uiSchemaManager(
    catalogs=[
        BasicCatalog.get_config(),
        CatalogConfig.from_path(
            name="my_catalog",
            catalog_path="catalog_schemas/my_catalog_definition.json",
            examples_path="examples/my_catalog",
        ),
    ],
)
```

Dynamic catalogs — resolve at runtime from client capabilities and stash in session
state, then let a toolset read them:

```python
async def _prepare_session(self, context, run_request, runner):
    session = await super()._prepare_session(context, run_request, runner)
    capabilities = context.message.metadata.get("a2ui_client_capabilities")
    a2ui_catalog = self.schema_manager.get_selected_catalog(
        client_ui_capabilities=capabilities
    )
    examples = self.schema_manager.load_examples(a2ui_catalog, validate=True)
    await runner.session_service.append_event(
        session,
        Event(actions=EventActions(state_delta={
            "system:a2ui_enabled": True,
            "system:a2ui_catalog": a2ui_catalog,
            "system:a2ui_examples": examples,
        })),
    )
    return session
```

> 🔄 **Grounded — schema-manager + dynamic-catalog pattern.** Both snippets are
> adapted from ADK's A2UI integration page (which documents `A2uiSchemaManager`,
> `BasicCatalog.get_config`, `CatalogConfig.from_path`, `get_selected_catalog`, and
> `load_examples`). Source: <https://adk.dev/integrations/a2ui/>.

> ⚠️ **Catalog must be self-contained (important).** When the schema manager
> selects a catalog, it returns *that* catalog standalone — the bundled
> `BasicCatalog` is not auto-merged. If your custom catalog declares only its own
> components but your examples reference `Text`/`Column`, validation can fail with
> `'/$defs/anyComponent' does not exist`. Fix it by merging the Basic Catalog's
> components into your custom catalog's `components` **and**
> `$defs.anyComponent.oneOf`. (This is a community implementation detail from the
> reference repo, not a GCP-documented API.)

---

## Part 5 — Deploy the agent

Gemini Enterprise needs a **publicly accessible** A2A endpoint. The GE docs name
two supported hosts:

| Host | Best for |
| --- | --- |
| **Agent Runtime on Gemini Enterprise Agent Platform** | Fully managed; ADK-native; designed for deploying and scaling AI agents |
| **Cloud Run** | Serverless containers; any framework; the doc tutorial uses this |

> 🔄 **Grounded — hosting options.** Source: GE "Register and manage agents using
> A2UI and A2A" (Host your agent).

The official GE tutorial sample lives at
`samples/community/agent/adk/gemini_enterprise/v0_9` in the
[A2UI repo](https://github.com/a2ui-project/a2ui). Documented structure:

| File/Dir | Purpose |
| --- | --- |
| `__init__.py` | Marks the directory as a Python package |
| `__main__.py` | Entry point to run the agent locally |
| `agent.py` | Defines the agent, its skills, and behavior |
| `agent_executor.py` | Manages execution flow + tool interactions |
| `deploy.sh` | Build + deploy to Cloud Run |
| `examples/` | Component template examples |
| `examples/0.9/material_table_orders.json` | Sample UI template for the recent-orders demo |
| `gemini_enterprise_composite_catalog.json` | Material + custom GE component catalog |
| `main.py` | FastAPI app entry point |
| `prompt_builder.py` | Model prompt construction helper |
| `pyproject.toml` | Project configuration + dependencies |
| `tools.py` | Tools the agent can use |

> 🔄 **Grounded — the file table.** The original omitted `__init__.py`,
> `__main__.py`, and `pyproject.toml` and had no `examples/0.9/...` row; this copy
> matches the documented structure. Source: GE "Host an A2UI agent with Cloud Run".

### Step 5.1 — Clone and prepare

```bash
git clone https://github.com/a2ui-project/a2ui.git
cd a2ui/samples/community/agent/adk/gemini_enterprise/v0_9
```

### Step 5.2 — Grant the Discovery Engine service agent Cloud Run Invoker

Only needed if your agent runs on Cloud Run behind a **default `.run.app` URL** with
authentication:

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-discoveryengine.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

Replace:

- `PROJECT_ID` — your project ID
- `PROJECT_NUMBER` — your project number

> 🔄 **Grounded — scope of this grant.** Google is explicit: this does **not apply
> when Cloud Run is behind Apigee, a load balancer, or a custom domain.** Source: GE
> "Host an A2UI agent with Cloud Run"; "Register and manage A2A agents".

### Step 5.3 — Deploy to Cloud Run

```bash
chmod +x deploy.sh
./deploy.sh PROJECT_ID a2ui-demo-agent MODEL_NAME
```

The script **builds a container image → pushes to Artifact Registry → deploys to
Cloud Run → sets env vars** (including `MODEL` and the public `AGENT_URL` of the
service itself).

Replace:

- `PROJECT_ID` — your project ID
- `MODEL_NAME` — *(optional)* supported values are `gemini-2.5-pro` or
  `gemini-2.5-flash`; defaults to `gemini-2.5-flash`

The script prints the **service URL** — you need it next.

> 🔄 **Grounded — deploy.sh contract.** Exact supported `MODEL_NAME` values and the
> default (`gemini-2.5-flash`) are per the GE Cloud Run tutorial.

### Step 5.4 (alternative) Deploy to Agent Runtime

If you prefer the fully managed path:

```bash
# Standard deployment: Cloud Console + ADK CLI, step-by-step
#   https://adk.dev/deploy/agent-runtime/deploy/

# Accelerated path with CI/CD + infra-as-code:
adk deploy ...   # see https://adk.dev/deploy/agent-runtime/agents-cli/
```

After deploying, get the Reasoning Engine resource path
(`projects/PROJECT_ID/locations/LOCATION/reasoningEngines/RESOURCE_ID`) — you'll
paste that when registering (Part 6).

> ⚠️ Agent Runtime is a **paid service** beyond the no-cost tier.

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
      "jsonAgentCard": "{\"protocolVersion\": \"0.3.0\", \"name\": \"A2UI v0.9 Demo\", \"description\": \"A demo agent that showcases A2UI v0.9 UIs built from the Material component catalog and Gemini Enterprise custom components (Canvas, Iframe). Ask it what can you do? to see the available demos.\", \"url\": \"AGENT_URL\", \"version\": \"1.0.0\", \"capabilities\": {\"streaming\": true, \"preferredTransport\": \"JSONRPC\", \"extensions\": [{\"uri\": \"https://a2ui.org/a2a-extension/a2ui/v0.9\", \"description\": \"Ability to render A2UI v0.9\", \"required\": false, \"params\": {\"supportedCatalogIds\": [\"https://www.gstatic.com/vertexaisearch/a2ui/v0_9/gemini_enterprise_composite_catalog.json\"]}}]}, \"skills\": [{\"id\": \"a2ui_demo\", \"name\": \"A2UI v0.9 Component Demo\", \"description\": \"Demonstrates A2UI v0.9 UIs built from the Material catalog and Gemini Enterprise custom components: cards, forms & inputs, tabs, tables, progress indicators, dialogs & menus, the Canvas side panel, and the Iframe (IFrameSrcdoc / IFrameUrl) components.\"}], \"defaultInputModes\": [\"text/plain\"], \"defaultOutputModes\": [\"text/plain\"]}"
    }
  }'
```

Replace:

- `PROJECT_NUMBER` — your project number
- `LOCATION` — multi-region of your data store: `global`, `us`, or `eu`
- `ENGINE_ID` — the ID of the app to register with
- `AGENT_URL` — the service URL your deploy step printed

> 🔄 **Grounded — the registration payload.** The `name`/`displayName`/`description`/
> `a2aAgentDefinition.jsonAgentCard` shape, the extension `uri`
> `https://a2ui.org/a2a-extension/a2ui/v0.9`, and `supportedCatalogIds` pointing at
> `https://www.gstatic.com/vertexaisearch/a2ui/v0_9/gemini_enterprise_composite_catalog.json`
> are copied from the GE Cloud Run tutorial verbatim. Note the catalog id uses
> `v0_9` (not `v0_9_1`).

**Unregister (optional):**

```bash
curl -X DELETE \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  https://discoveryengine.googleapis.com/v1alpha/projects/PROJECT_NUMBER/locations/LOCATION/collections/default_collection/engines/ENGINE_ID/assistants/default_assistant/agents/AGENT_ID
```

> ℹ️ **Agent Gateway note.** Google states that traffic to A2A agents added through
> this method **does not go through Agent Gateway**, and Agent Gateway policies do
> not apply. Source: GE "Register and manage A2A agents".

### Path B — Register an ADK agent (Agent Runtime)

Console method: Gemini Enterprise console → your app → **Agents** → **Add agent** →
**Custom agent via Agent Runtime**; optionally add authorizations; enter an **Agent
name**, a **description** (the LLM uses it to decide whether to invoke your agent),
and the **Agent Runtime resource path**
(`projects/PROJECT_ID/locations/LOCATION/reasoningEngines/RESOURCE_ID`) → **Create**.

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

Replace: `ENDPOINT_LOCATION` (`us`/`eu`/`global`), `PROJECT_ID`, `PROJECT_NUMBER`,
`APP_ID`, `DISPLAY_NAME`, `DESCRIPTION`, `ICON_URI`, `RESOURCE_ID`,
`RESOURCE_LOCATION`, `AUTH_ID`.

> 🔄 **Grounded — the ADK registration shape.** Confirmed against GE "Register and
> manage ADK agents hosted on Agent Runtime": the endpoint uses
> `locations/global`, the body nests
> `adkAgentDefinition.provisionedReasoningEngine.reasoningEngine`, and it sends
> `X-Goog-User-Project`. (The original's phrasing "register in one CLI command via
> Agents CLI" is not what the GE doc describes — the doc gives Console + REST. Use
> Agents CLI only if your own tooling documents it.)

### Step 6.3 — Use the agent in the web app

1. Gemini Enterprise console → your app → **Integrations** → copy
   **The link to your web app** and share it.
2. Open the web app URL → navigation menu → **Agents**.
3. Under **From your organization**, click your agent.
4. Ask something like *"Show me the recent orders table"* — the agent renders the
   list using custom UI components in chat.

> 🔄 **Grounded — usage steps + demo prompt.** Source: GE "Host an A2UI agent with
> Cloud Run" ("Use the agent on the Gemini Enterprise web app").

### Authentication headers GE sends

| Header | Purpose |
| --- | --- |
| `X-Serverless-Authorization` | Service-to-service IAM auth. Sent **only for a default `.run.app` URL**. GE generates a Google-signed OIDC ID token for the Discovery Engine service agent (`service-PROJECT_NUMBER@gcp-sa-discoveryengine.iam.gserviceaccount.com`) with your agent URL as audience. |
| `Authorization` | End-user OAuth 2.0 token (only if you configure OAuth 2.0). Standard `Bearer USER_OAUTH_TOKEN`. |

> 🔄 **Grounded — auth behavior.** Google: GE does **not** use your agent card's
> `securitySchemes` / `security` fields; it does **not** support the `apiKey`
> security scheme; and it provides **no way to add static API keys or custom header
> values**. Because Cloud Run consumes `X-Serverless-Authorization` for service-level
> IAM while leaving `Authorization` intact for your code, you can keep the service
> private *and* receive user OAuth tokens without collisions. If you front the agent
> with Apigee, GE sends the user token in `Authorization` but **not**
> `X-Serverless-Authorization` — validate the token at the proxy. Source: GE
> "Register and manage A2A agents".

---

## Part 7 — Troubleshooting & common pitfalls

### Validation

- **`'/$defs/anyComponent' does not exist`** → your catalog isn't self-contained.
  Merge `BasicCatalog` into your custom catalog (components **and**
  `$defs.anyComponent.oneOf`). See Part 4.5.
- **Blank UI** → check the renderer's errors; usually v0.8/v0.9 syntax mixed (see
  the diff table). Remember v0.8 nests `"component": {"Text": {...}}` and uses
  `literalString`/`explicitList`/`usageHint`, while v0.9 flattens all of it.
- **Malformed UI never reached the client** → good — that means validation worked.
  LLM output frequently has Markdown fences; parse defensively and gate with the
  catalog validator.

### Local dev

- **Port 5173 in use** → the dev server auto-picks the next port; check terminal output.
- **`ERR_CONNECTION_REFUSED` on startup** → known race: the web client starts
  before the Python agent. Wait a few seconds and refresh.
- **Missing API key** → `echo $GOOGLE_API_KEY` to verify; re-export if needed.
- **`uv: command not found`** → install `uv` (see Part 0).

### Gemini Enterprise runtime

- **GE shows nothing** → you're likely emitting the wrong pattern for the client.
  GE expects the **inline** pattern today. Confirm your extension/fallback logic.
- **Auth fails on Cloud Run** → the Discovery Engine service agent needs
  `roles/run.invoker` (Part 5.2) — **only** for default `.run.app` URLs.
- **Custom components render, basics don't** → your frontend `customCatalog` must
  *also* include the basic components. The backend stamps the merged catalog with
  the custom `catalogId`, so the frontend must mirror that merge.

### Version discipline

- Keep the A2UI version string in **one** config place. Swapping revisions should
  be a one-line change.
- GE supports **A2A v0.3 streaming** (the sample agent card declares
  `"protocolVersion": "0.3.0"` and `"preferredTransport": "JSONRPC"`). On newer A2A
  versions, use the SDK compatibility packages.
- **Watch the catalog id:** Google realigned the canonical basic-catalog id from
  `.../v0_9_1/...` to `.../v0_9/...` "to align with the canonical A2UI protocol
  specification." Copy ids from the docs you're targeting, not from old blog posts.

---

## Part 8 — What's next

- **Spec & component reference:** <https://a2ui.org/> (see "Specification Versions")
- **GE component gallery reference:** <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/a2ui-component-gallery-reference>
- **GE: register & manage A2UI/A2A agents:** <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/register-and-manage-an-a2ui-agent>
- **GE: host an A2UI agent with Cloud Run:** <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/tutorial-host-agent-cloud-run>
- **GE: register & manage A2A agents:** <https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent>
- **GE: register & manage ADK agents (Agent Runtime):** <https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent>
- **ADK × A2UI:** <https://adk.dev/integrations/a2ui/>
- **Samples repo:** <https://github.com/a2ui-project/a2ui>
- **Composer catalogs (linked by GE docs):** <https://a2ui-composer.ag-ui.com/basic-catalog> · <https://a2ui-composer.ag-ui.com/custom-catalog>

**The takeaway:** in Gemini Enterprise you only build the agent and the A2UI it
emits. GE owns the shell, the renderer, and the transport. The next time a user
asks for "a table for two tomorrow at 7pm," the answer can be a date picker
instead of another question.

---

## Sources (retrieved/validated via the Google Developer Knowledge MCP server)

1. Register and manage agents using A2UI and A2A — <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/register-and-manage-an-a2ui-agent>
2. A2UI component gallery reference — <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/a2ui-component-gallery-reference>
3. Host an A2UI agent with Cloud Run — <https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/tutorial-host-agent-cloud-run>
4. Register and manage A2A agents — <https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent>
5. Register and manage ADK agents hosted on Agent Runtime — <https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent>
6. ADK × A2UI integration — <https://adk.dev/integrations/a2ui/>
7. Connect ADK to Google Cloud and Agent Platform — <https://adk.dev/get-started/google-cloud/>
8. Python Quickstart for ADK — <https://adk.dev/get-started/python/>
9. A2UI project — <https://a2ui.org/>
10. A2UI repo — <https://github.com/a2ui-project/a2ui>
11. Google Workspace Chat A2UI agent quickstart (launch-stage reference) — <https://developers.google.com/workspace/chat/quickstart-a2ui-agent>

### Not grounded by Google docs (flagged inline)

- The A2UI **Composer** tool's own UI/key handling (Part 3) — not in the GCP corpus.
- A2UI **Python SDK** internals (`DirectJsonFormat`, `SendA2uiToClientToolset`) —
  community sample APIs, not GCP-documented; the ADK-documented equivalent is the
  schema-manager flow in Part 4.5.
