---
title: "Agent → UI with A2UI 0.9 and ADK (Vertex AI): A Developer's Guide"
date: 2026-10-11
tags: [a2ui, adk, vertex-ai, ag-ui, copilotkit, genui, agents]
sources:
  - https://a2ui.org/guides/a2ui-with-any-agent-framework/
  - https://github.com/ag-ui-protocol/ag-ui/tree/main/integrations/adk-middleware/python
  - https://github.com/ag-ui-protocol/ag-ui/tree/main/skills/ag-ui-a2ui-integration
  - https://a2ui.org/specification/v0.9.1-a2ui.md
  - https://docs.copilotkit.ai/generative-ui/a2ui
  - https://docs.copilotkit.ai/google-adk/quickstart
  - https://google.github.io/adk-docs/get-started/quickstart/
  - https://docs.ag-ui.com
versions:
  a2ui: v0.9 (stable family), v0.9.1 (current)
  ag-ui: current
  adk: google-adk 1.x (1.16.0+ for resumable HITL)
  copilotkit: v2 (1.61.2+ for catalog auto-forward / auto-inject)
  vertex-ai: google-genai via ADK's GoogleLLM, region-dependent
---

# Building an Agent → User Interface with A2UI v0.9 and an ADK Agent (Vertex AI)

> **Hook:** Your agent finds the perfect flights, answers in a wall of text, and the user retypes their choice back. Every option is a turn. Every turn is friction. A2UI fixes this: instead of returning prose, the agent returns a *declarative JSON description of a UI* — cards, date pickers, buttons — and the client paints it natively from a pre-approved catalog.

By the end you'll have a **Gemini-powered ADK agent (running on Vertex AI, no API keys) that streams real A2UI surfaces into a CopilotKit (React/Next.js) chat**, plus a custom component catalog you control.

**Change in this revision:** the agent now runs on **Gemini via Vertex AI** using Application Default Credentials (ADC) / a service account — no `GOOGLE_API_KEY` anywhere. Everything else (A2UI, AG-UI transport, CopilotKit renderer) stays the same.

---

## Part 0 — Prerequisites (Vertex AI edition)

| Requirement | Why |
|---|---|
| Python 3.9+ | ADK agent + AG-UI middleware |
| Node.js 20+ | CopilotKit / Next.js frontend |
| **A Google Cloud project with the Vertex AI API enabled** | All model calls go through Vertex AI |
| **`gcloud` CLI, authenticated** | For ADC (`gcloud auth application-default login`) or a service account |
| **A principal with `roles/aiplatform.user`** | Permission to call Gemini on Vertex AI |
| `uv` (recommended) | Fast Python env + `google-adk` tooling |

```bash
# 1. Install the Google Cloud CLI, then authenticate.
gcloud auth login
gcloud auth application-default login        # writes ADC to ~/.config/gcloud/application_default_credentials.json

# 2. Project + APIs
gcloud config set project YOUR_PROJECT_ID
gcloud services enable aiplatform.googleapis.com

# 3. Grant the caller permission to invoke Gemini
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:you@example.com" \
  --role="roles/aiplatform.user"

# 4. uv
curl -LsSf https://astral.sh/uv/install.sh | sh
```

> ✅ **No `GOOGLE_API_KEY`.** Vertex AI authenticates with **ADC** by default — the same credential chain the Google Cloud client libraries use. In production/CI, set `GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json` instead of running `application-default login`.

> ⚠️ **Version note:** A2UI has three live revisions — **v0.8 (legacy)**, **v0.9 (stable family)**, **v0.9.1 (current)**. This tutorial targets **v0.9.x**. v0.9 is *prompt-first*: the schema is injected into the model prompt rather than enforced via structured output, which is more expressive but requires post-generation validation.

---

## Part 1 — Background: what A2UI is, and where it sits

### The problem
In a multi-agent system the agent doing the work is often *remote* — a different server, a different company. It can't touch your DOM. The old workaround was shipping HTML/JS into a sandboxed iframe: heavy, visually disjointed, and a security headache (XSS, UI injection).

**A2UI's answer: transmit UI that is safe like data, but expressive like code.**

### The core idea
The agent emits JSON describing:
- a **tree of components** (`Card`, `Text`, `Button`, `ChoicePicker`, `Image`, …), and
- a separate **data model** holding the values those components display.

The client renders only components from a **pre-approved catalog** it already trusts. No arbitrary code runs.

### Three properties that make it work
| Property | Meaning |
|---|---|
| **Declarative, not executable** | It's data. The client renders catalog components only — no code execution, no UI injection. |
| **Streaming-friendly** | A flat list of small JSON messages, so the LLM emits them incrementally and the client paints as they arrive. |
| **Framework-agnostic** | The same payload renders through Lit, Angular, React, Flutter, or native. The agent doesn't know or care. |

### The four layers (don't conflate them)
| Layer | Owns | Examples |
|---|---|---|
| App experience | Chat shell + conversation state | CopilotKit, AG-UI |
| Pixel drawing | Turning component descriptions into UI | React, Lit, Angular, Flutter |
| Conversation pipeline | Client ↔ server transport | **AG-UI**, A2A, SSE, WebSockets |
| **Cargo (data format)** | **The thing describing the UI** | **A2UI** |

For **this** tutorial: **ADK agent (Gemini on Vertex AI) → `ag_ui_adk` middleware → AG-UI transport → CopilotKit React renderer → A2UI components.**

### A2UI vs A2A vs AG-UI
- **A2A** = agent ↔ agent.
- **A2UI** = agent ↔ **human** (structured UI).
- **AG-UI** = the transport carrying A2UI messages between agent and app.

### The v0.9 protocol in four message types
Every server→client message is one JSON object containing exactly one of:

| Message | Purpose |
|---|---|
| `createSurface` | Create a surface and begin rendering. Fixes `surfaceId` + `catalogId`. |
| `updateComponents` | Add/update components (a **flat adjacency list**, linked by `id`). |
| `updateDataModel` | Populate/replace the data model (optionally at a JSON Pointer `path`). |
| `deleteSurface` | Remove a surface. |

One component must have `id: "root"`. Components may reference children/data that don't exist yet — clients render **placeholders** (progressive rendering).

```json
{ "version": "v0.9.1", "createSurface": { "surfaceId": "main",
  "catalogId": "https://a2ui.org/specification/v0_9_1/catalogs/basic/catalog.json" } }
{ "version": "v0.9.1", "updateComponents": { "surfaceId": "main", "components": [
  { "id": "root", "component": "Column", "children": ["title"] },
  { "id": "title", "component": "Text", "text": "Hello A2UI", "variant": "h1" } ] } }
```

### Data binding
| Form | Example | Meaning |
|---|---|---|
| Static | `"text": "Hello"` | literal |
| Dynamic | `"text": { "path": "/user/name" }` | read from data model (JSON Pointer) |

---

## Part 2 — Two ways to emit A2UI (choose before you code)

| Mode | Who owns the layout | Backend shape | Use for |
|---|---|---|---|
| **Fixed schema** | App developer | Tools return an `a2ui_operations` envelope | Dashboards, search results, known cards |
| **Dynamic schema** | The model (composes from a catalog) | `generate_a2ui` tool → sub-agent streams `render_a2ui` | Open-ended "draw me a UI" requests |

This tutorial builds dynamic first, then adds a fixed-schema tool.

---

## Part 3 — Step-by-step: the ADK agent on Vertex AI

### Step 3.1 — Scaffold

```bash
uv init a2ui-adk-demo && cd a2ui-adk-demo
uv add ag-ui-adk google-adk uvicorn fastapi
```

> ⚠️ The middleware can also be installed from source: clone
> [ag-ui](https://github.com/ag-ui-protocol/ag-ui), `cd integrations/adk-middleware/python`, then `pip install .` or `uv pip install .`. The import name is **`ag_ui_adk`**. If `ag-ui-adk` isn't the exact PyPI name in your environment, install from the repo path above — the import name is stable either way.

### Step 3.2 — Point ADK at Vertex AI

ADK selects the backend from environment variables. **Unset `GOOGLE_API_KEY` and set the Vertex variables instead:**

```bash
# --- Vertex AI configuration (replaces GOOGLE_API_KEY entirely) ---
export GOOGLE_GENAI_USE_VERTEXAI=TRUE
export GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
export GOOGLE_CLOUD_LOCATION=us-central1     # a region where Gemini is available
# Optional alternative to ADC file lookup:
# export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json

# Make sure no stray key shadows Vertex:
unset GOOGLE_API_KEY
```

| Variable | Purpose |
|---|---|
| `GOOGLE_GENAI_USE_VERTEXAI` | `TRUE` routes the Google GenAI client through Vertex AI instead of the Gemini Developer API |
| `GOOGLE_CLOUD_PROJECT` | Your GCP project id |
| `GOOGLE_CLOUD_LOCATION` | Vertex region (e.g. `us-central1`); must support your Gemini model |
| `GOOGLE_APPLICATION_CREDENTIALS` | *Optional* — explicit service-account JSON; omit to use ADC |

**Expected result:** `gcloud auth application-default print-access-token` succeeds, and no API key is present in the environment.

### Step 3.3 — Baseline agent (text only, no A2UI yet)

`main.py`:
```python
from fastapi import FastAPI
from ag_ui_adk import ADKAgent, add_adk_fastapi_endpoint
from google.adk.agents import LlmAgent

# Model string is a Vertex AI model id (env vars route it through Vertex).
agent = LlmAgent(
    name="assistant",
    model="gemini-2.5-flash",
    instruction="Be helpful and fun!",
)

adk_agent = ADKAgent(
    adk_agent=agent,
    app_name="demo_app",
    user_id="demo_user",
    session_timeout_seconds=3600,
    use_in_memory_services=True,
)

app = FastAPI()
add_adk_fastapi_endpoint(app, adk_agent, path="/")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="localhost", port=8000)
```

Run: `uv run main.py` → **Expected result:** server on `http://localhost:8000` returning plain text, with every model call billed to your Vertex AI project.

> 🧭 **Explicit-model alternative.** If you prefer to pin the Vertex client in code rather than via env vars, build the model object explicitly and pass it to `LlmAgent(model=...)`:
> ```python
> from google.adk.models.google_llm import GoogleLLM
> model = GoogleLLM(model="gemini-2.5-pro")   # honours GOOGLE_GENAI_USE_VERTEXAI / project / location
> ```
> Keep the env vars set either way — they tell ADK *which backend* to use.

### Step 3.4 — Turn on dynamic A2UI (auto-injection)

The ADK adapter injects `generate_a2ui` when the runtime forwards `injectA2UITool`, infers the sub-agent model from the agent, and streams nested `render_a2ui` events. You only pass A2UI *preferences*.

```python
CUSTOM_CATALOG_ID = "https://a2ui.org/demos/dojo/dynamic_catalog.json"

COMPOSITION_GUIDE = """
## Available Pre-made Components
You have 4 components. Use Row as the root with structural children to repeat a card per item.

### Row
{"id":"root","component":"Row","children":{"componentId":"card","path":"/items"}}

### HotelCard
Props: name, location, rating (0-5), pricePerNight, amenities (optional), action
{"id":"card","component":"HotelCard","name":{"path":"name"},"location":{"path":"location"},
 "rating":{"path":"rating"},"pricePerNight":{"path":"pricePerNight"},
 "action":{"event":{"name":"book","context":{"name":{"path":"name"}}}}}

### ProductCard
Props: name, price, rating, description (optional), badge (optional), action
### TeamMemberCard
Props: name, role, department, email, avatarUrl, action

## RULES
- Root is ALWAYS a Row with structural children {"componentId":"<card-id>","path":"/items"}
- Inside templates use RELATIVE paths: {"path":"name"} not {"path":"/name"}
- Always provide data in the "data" argument as {"items":[...]}
- Generate 3-4 realistic items with diverse data
"""

SYSTEM_PROMPT = """You are a helpful assistant that creates rich visual UI on the fly.
When the user asks for visual content, use generate_a2ui to create a dynamic A2UI surface.
When the user asks to MODIFY a surface you already rendered, call generate_a2ui with
intent="update" and target_surface_id set to that surface's id.
IMPORTANT: After calling the tool, do NOT repeat the data in text. Just confirm what was rendered."""

dynamic_agent = LlmAgent(
    model="gemini-2.5-pro",          # Vertex AI model id
    name="a2ui_dynamic_schema",
    instruction=SYSTEM_PROMPT,
    # generate_a2ui is auto-injected by the adapter — nothing wired here.
)

adk_a2ui_dynamic = ADKAgent(
    adk_agent=dynamic_agent,
    app_name="demo_app",
    user_id="demo_user",
    session_timeout_seconds=3600,
    use_in_memory_services=True,
    a2ui={
        "default_catalog_id": CUSTOM_CATALOG_ID,
        "guidelines": {"composition_guide": COMPOSITION_GUIDE},
    },
)
```

**Why it works:** the adapter renders your catalog into the sub-agent prompt (`render_as_llm_instructions`) and validates generated components against it (structural/lenient). The sub-agent is forced to call `render_a2ui`, whose args stream onto the AG-UI wire as nested tool-call events so the middleware can paint progressively. Because ADK is running on Vertex, both the planner and the sub-agent calls are Vertex-authenticated automatically.

### Step 3.5 — Add a fixed-schema tool (optional, same app)

```python
from ag_ui_a2ui_toolkit import (
    A2UI_OPERATIONS_KEY, create_surface, update_components, update_data_model,
)

CUSTOM_CATALOG_ID = "https://a2ui.org/demos/dojo/fixed_catalog.json"

def search_hotels(hotels: list[dict]) -> dict:
    """Search for hotels and display results as rich cards.
    Each hotel needs: id, name, location, rating (0-5 float), price (per night)."""
    return {
        A2UI_OPERATIONS_KEY: [
            create_surface("hotel-search-results", catalog_id=CUSTOM_CATALOG_ID),
            update_components("hotel-search-results", HOTEL_SCHEMA),
            update_data_model("hotel-search-results", {"hotels": hotels}),
        ]
    }
```

> 🔑 **Return a `dict`, not a JSON string.** ADK keeps a dict tool-return as-is; the middleware then `json.dumps` it into the `{"a2ui_operations": [...]}` string the client scans for. Returning a string makes ADK wrap it as `{"result": "..."}`, which the middleware **won't** recognize.

Add `tools=[search_hotels]` to a fixed-schema agent and expose it on its own path.

### Step 3.6 — Human-in-the-loop (optional, ADK ≥ 1.16.0)

```python
from google.adk.apps import App, ResumabilityConfig
from ag_ui_adk import ADKAgent, AGUIToolset, add_adk_fastapi_endpoint

my_agent = LlmAgent(name="assistant", model="gemini-2.5-pro",
                    instruction="...", tools=[AGUIToolset()])

adk_app = App(name="my_app", root_agent=my_agent,
              resumability_config=ResumabilityConfig(is_resumable=True))

agent = ADKAgent.from_app(adk_app, user_id="user123",
                          session_timeout_seconds=3600, use_in_memory_services=True)
```

> ⚠️ Fire-and-forget HITL via `ADKAgent(adk_agent=...)` is **deprecated** for client-side tools. Use `from_app()`.

> 🏢 **Production session store.** `use_in_memory_services=True` is for local dev. On Vertex AI you'll typically swap in a persistent ADK session service (e.g. Vertex AI Session Service), which picks up project/location from the same env vars.

---

## Part 4 — Runtime + CopilotKit frontend

### Step 4.1 — Create the Next.js app

```bash
npx create-next-app@latest my-copilot-app
cd my-copilot-app
npm install @copilotkit/react-core @copilotkit/runtime @copilotkit/a2ui-renderer @ag-ui/client zod
```

> ⚠️ CopilotKit **v2** is current. Import from `/v2`: `@copilotkit/react-core/v2`, `@copilotkit/runtime/v2`. The package root is deprecated v1 and still resolves — mixing them breaks at runtime.

### Step 4.2 — Wire the runtime to your ADK agent

`app/api/copilotkit/[[...slug]]/route.ts`:
```ts
import { CopilotRuntime, createCopilotRuntimeHandler } from "@copilotkit/runtime/v2";
import { HttpAgent } from "@ag-ui/client";

const runtime = new CopilotRuntime({
  agents: {
    a2ui_dynamic_schema: new HttpAgent({ url: "http://localhost:8000/" }),
  },
  a2ui: { injectA2UITool: true }, // enables A2UI for all agents
});

const handler = createCopilotRuntimeHandler({ runtime, basePath: "/api/copilotkit" });
export const GET = handler;
export const POST = handler;
```

> Prefer the client-forwarded catalog path (Step 4.4): with CopilotKit ≥ 1.61.2, `<CopilotKit a2ui={{ catalog }}>` auto-enables A2UI, defaults tool injection on, and auto-derives `defaultCatalogId` from the catalog's `catalogId` — so you can drop the runtime `a2ui` block. Use the explicit block only to override/scope. **Never apply the middleware twice** to the same agent.

### Step 4.3 — Define a custom catalog (BYOC)

`lib/a2ui/definitions.ts`:
```ts
import { z } from "zod";

export const myDefinitions = {
  StatusBadge: {
    description: "A colored status badge.",
    props: z.object({
      text: z.string(),
      variant: z.enum(["success", "warning", "error"]).optional(),
    }),
  },
  Metric: {
    description: "A key metric with label and value.",
    props: z.object({
      label: z.string(),
      value: z.string(),
      trend: z.enum(["up", "down"]).optional(),
    }),
  },
};
export type MyDefinitions = typeof myDefinitions;
```

`lib/a2ui/renderers.tsx`:
```tsx
"use client";
import { createCatalog, type CatalogRenderers } from "@copilotkit/a2ui-renderer";
import { myDefinitions, type MyDefinitions } from "./definitions";

const myRenderers: CatalogRenderers<MyDefinitions> = {
  StatusBadge: ({ props }) => (
    <span style={{ padding: "2px 8px", borderRadius: 9999, fontSize: "0.75rem",
      background: props.variant === "error" ? "#fee2e2" : "#dcfce7",
      color: props.variant === "error" ? "#991b1b" : "#166534" }}>
      {props.text}
    </span>
  ),
  Metric: ({ props }) => (
    <div>
      <div style={{ fontSize: "0.75rem", color: "#6b7280" }}>{props.label}</div>
      <div style={{ fontSize: "1.5rem", fontWeight: 700 }}>
        {props.value} {props.trend === "up" ? "↑" : props.trend === "down" ? "↓" : ""}
      </div>
    </div>
  ),
};

export const myCatalog = createCatalog(myDefinitions, myRenderers, {
  catalogId: "https://example.com/catalogs/my-app-catalog.json",
  includeBasicCatalog: true, // keep built-ins (Text, Card, …) alongside yours
});
```

> `createCatalog` is generic over the definitions type, so a typo like `props.txt` is a **compile error**. `catalogId` is the stable handle the agent uses; the runtime stamps a matching `defaultCatalogId`.

### Step 4.4 — Register the catalog + chat

`app/providers.tsx`:
```tsx
"use client";
import { CopilotKit } from "@copilotkit/react-core/v2";
import { myCatalog } from "@/lib/a2ui/renderers";

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <CopilotKit
      runtimeUrl="/api/copilotkit"
      agent="a2ui_dynamic_schema"
      a2ui={{ catalog: myCatalog }}   // ← auto-enables + supplies catalogId
    >
      {children}
    </CopilotKit>
  );
}
```

`app/layout.tsx` imports `./providers` (layout is a server component) and `"@copilotkit/react-core/v2/styles.css"`. `app/page.tsx` renders `<CopilotChat />` (or `<CopilotSidebar />`).

### Step 4.5 — Run it
Terminal 1: `uv run main.py` (port 8000, with the Vertex env vars exported) · Terminal 2: `npm run dev` → open `localhost:3000`.

**Expected result:** ask *"compare three laptops"* → the agent calls `generate_a2ui`, streams `render_a2ui` args, and a **Row of ProductCards** paints progressively (not just prose). Clicking a card action flows back to the agent as a synthetic `log_a2ui_event` tool message.

---

## Part 5 — Production / Cloud Run note (Vertex-friendly)

For an **AG-UI + Vertex AI** deployment on Cloud Run, the agent container authenticates to Vertex the same way it does locally:

- Attach a **runtime service account** to the Cloud Run service that has `roles/aiplatform.user`.
- Do **not** bake a key into the image; the Google auth libraries pick up the service account automatically.
- Set `GOOGLE_GENAI_USE_VERTEXAI=TRUE`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION` as container env vars.

```bash
gcloud run deploy a2ui-adk-agent \
  --source . \
  --region us-central1 \
  --service-account a2ui-agent@YOUR_PROJECT_ID.iam.gserviceaccount.com \
  --set-env-vars GOOGLE_GENAI_USE_VERTEXAI=TRUE,GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID,GOOGLE_CLOUD_LOCATION=us-central1
```

> Note: the Gemini Enterprise (A2A) path has its own Discovery Engine service-agent setup; that does not apply to this AG-UI + Vertex AI deployment.

---

## Part 6 — What the wire actually looks like (debug view)

| Layer | What you see |
|---|---|
| AG-UI stream | `RUN_STARTED` → `TOOL_CALL_START` (`generate_a2ui`) → nested `render_a2ui` deltas → `ACTIVITY_SNAPSHOT` (`activityType: "a2ui-surface"`) → `RUN_FINISHED` |
| Middleware | Detects `a2ui_operations` in tool results, paints the surface, forwards user actions back |
| Client | Renders catalog components; a user click returns via `forwardedProps.a2uiAction` |

---

## Part 7 — Verification checklist

- [ ] `gcloud auth application-default print-access-token` succeeds; **no `GOOGLE_API_KEY` set**.
- [ ] `GOOGLE_GENAI_USE_VERTEXAI=TRUE`, project, and location are exported.
- [ ] Dependencies install cleanly; typecheck/lint pass.
- [ ] Stream starts with a valid run, ends with `RUN_FINISHED` **or** `RUN_ERROR` (never swallowed).
- [ ] Dynamic: `generate_a2ui` → streamed `render_a2ui` args → `ACTIVITY_SNAPSHOT` with `a2ui-surface`.
- [ ] Fixed: the backend tool result contains an `a2ui_operations` envelope.
- [ ] **A surface renders — not just a text explanation.**
- [ ] One user interaction in the rendered surface flows back to the agent.
- [ ] No schema/hydration/stream/action-bridge errors in console or backend logs.

---

## Part 8 — Common pitfalls

| Symptom | Likely cause | Fix |
|---|---|---|
| `403 PERMISSION_DENIED` / `aiplatform` errors | Caller lacks Vertex access | Grant `roles/aiplatform.user`; `gcloud auth application-default login` |
| Model calls fail after switching | Stray `GOOGLE_API_KEY` shadowing Vertex | `unset GOOGLE_API_KEY`; keep only Vertex vars |
| `Location ... not supported` | Region doesn't host the model | Pick a supported `GOOGLE_CLOUD_LOCATION` (e.g. `us-central1`) |
| No surface appears | A2UI enabled on only one side | Enable catalog client-side **and** middleware/runtime `a2ui` server-side |
| Agent *describes* the UI in prose | Agent lacks `generate_a2ui` / fixed tools | Use the adapter auto-injection or return `a2ui_operations` |
| Custom component never renders | Catalog id / names drift server↔client | Align `defaultCatalogId` / `catalogId` / names; forward the catalog |
| Surface only appears at the end | Nested `render_a2ui` not streaming | Use the adapter's streaming path, not a non-streaming invoke |
| Action clicks do nothing | Action bridge not reaching `forwardedProps.a2uiAction` | Verify client forwarding; middleware emits `log_a2ui_event` |
| Skeletons duplicate/flicker | Middleware applied twice | Runtime-level **or** per-agent, never both |
| `Catalog not found` | Server stamped an unregistered catalog id | Forward the catalog so its id is auto-derived |
| Blank UI | v0.8/v0.9 syntax mixed | See the diff table (Part 9) |

---

## Part 9 — The v0.8 → v0.9 syntax table (keep this open)

| Aspect | v0.8 | v0.9 |
|---|---|---|
| Component wrapper | `"component": { "Text": { … } }` | `"component": "Text", …props` |
| String values | `{ "literalString": "Hello" }` | `"Hello"` |
| Children | `{ "explicitList": ["a","b"] }` | `["a","b"]` |
| Text/Image styling | `usageHint` | `variant` |
| Button styling | `"primary": true` | `"variant": "primary"` |
| Action format | `{ "name": "..." }` | `{ "event": { "name": "..." } }` |
| Choice component | `MultipleChoice` | `ChoicePicker` |
| Layout alignment | `distribution`, `alignment` | `justify`, `align` |
| TextField value | `text` | `value` |
| Start render | `beginRendering` | `createSurface` |

---

## What's next

- **Spec:** <https://a2ui.org/> · v0.9.1 at `/specification/v0.9.1-a2ui.md`
- **Composer (design UI visually):** <https://a2ui-project.github.io/composer/>
- **AG-UI A2UI skill:** `skills/ag-ui-a2ui-integration` in the AG-UI repo
- **ADK middleware:** `integrations/adk-middleware/python` (`TOOLS.md`, `CONFIGURATION.md`, `USAGE.md`)
- **ADK + Vertex quickstart:** <https://google.github.io/adk-docs/get-started/quickstart/>
- **CopilotKit A2UI docs:** <https://docs.copilotkit.ai/generative-ui/a2ui>
- **AG-UI docs:** <https://docs.ag-ui.com>

**Takeaway:** with ADK on Vertex AI you build the *agent* and the *A2UI it emits*; authentication is silent (ADC / service account), the middleware handles transport, and CopilotKit owns the shell and rendering. No API keys to leak — just a project, a region, and an IAM role.

---

### Sources
1. A2UI with any agent framework — https://a2ui.org/guides/a2ui-with-any-agent-framework/
2. AG-UI ADK middleware (Python) — https://github.com/ag-ui-protocol/ag-ui/tree/main/integrations/adk-middleware/python
3. AG-UI A2UI integration skill — https://github.com/ag-ui-protocol/ag-ui/tree/main/skills/ag-ui-a2ui-integration
4. A2UI v0.9.1 spec — https://raw.githubusercontent.com/a2ui-project/a2ui/main/specification/v0_9_1/docs/a2ui_protocol.md
5. CopilotKit A2UI docs — https://docs.copilotkit.ai/generative-ui/a2ui
6. CopilotKit ADK quickstart — https://docs.copilotkit.ai/google-adk/quickstart
7. ADK quickstart (Vertex AI) — https://google.github.io/adk-docs/get-started/quickstart/
8. AG-UI docs — https://docs.ag-ui.com
