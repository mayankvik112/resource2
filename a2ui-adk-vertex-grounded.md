---
title: "Agent → UI with A2UI and ADK on Google Cloud Agent Platform: A Grounded Developer's Guide"
date: 2026-10-11
tags: [a2ui, adk, agent-platform, vertex-ai, ag-ui, copilotkit, genui, agents]
grounded_with: "Google Developer Knowledge MCP server (search_documents / get_documents / answer_query)"
sources:
  - https://adk.dev/integrations/a2ui/
  - https://adk.dev/integrations/ag-ui/
  - https://adk.dev/integrations/copilotkit/
  - https://adk.dev/get-started/google-cloud/
  - https://adk.dev/agents/models/google-gemini/
  - https://adk.dev/agents/llm-agents/
  - https://adk.dev/sessions/session/
  - https://adk.dev/runtime/resume/
  - https://adk.dev/deploy/cloud-run/
  - https://a2ui.org/
  - https://docs.ag-ui.com
versions:
  a2ui: v0.9.x (protocol); a2ui-agent-sdk (pip) for ADK
  ag-ui-adk: current (PyPI `ag-ui-adk`)
  adk: google-adk 1.x (Python 3.10+; 1.16+ for resumability/HITL)
  copilotkit: "@copilotkit/*/v2" (Python 3.10–3.14, Node 18+)
  agent-platform: "Google Cloud Agent Platform (formerly Vertex AI / Gemini Enterprise Agent Platform)"
---

# Building an Agent → User Interface with A2UI and an ADK Agent on Google Cloud Agent Platform

> **Hook:** Your agent finds the perfect flights, answers in a wall of text, and the user retypes their choice back. Every option is a turn. Every turn is friction. A2UI fixes this: instead of returning prose, the agent returns a *declarative JSON description of a UI* — cards, date pickers, buttons — and the client paints it natively from a pre-approved catalog.

By the end you'll have a **Gemini-powered ADK agent (running on Google Cloud Agent Platform, no API keys) that streams real A2UI surfaces into a CopilotKit (React/Next.js) chat**, plus a custom component catalog you control.

> 📚 **Grounded revision.** Every Google-product claim below is grounded against official Google developer documentation (ADK docs at `adk.dev`, Google Cloud docs, and the A2UI spec), retrieved via the Google Developer Knowledge MCP server. Where the source tutorial used older names or an unofficial integration path, this copy follows the documented path and flags the change inline with **🔄 Grounded**.

---

## Part 0 — Prerequisites (Agent Platform edition)

| Requirement | Why |
|---|---|
| **Python 3.10+** (3.10–3.14 for the CopilotKit path) | ADK Python requires 3.10 or later; the CopilotKit integration documents 3.10–3.14 |
| **Node.js 18+** | CopilotKit / Next.js frontend (documented prerequisite) |
| **A Google Cloud project with the Agent Platform API enabled** | All model + platform calls go through Google Cloud |
| **`gcloud` CLI, authenticated** | For ADC (`gcloud auth application-default login`) or a service account |
| **A principal with `roles/aiplatform.user`** ("Agent Platform User") | Permission to call Gemini and Agent Platform services |
| `uv` (optional, recommended for speed) | Fast Python env management |

```bash
# 1. Install the Google Cloud CLI, then authenticate.
gcloud auth login
gcloud auth application-default login        # writes ADC credentials for local dev

# 2. Project + APIs
gcloud config set project YOUR_PROJECT_ID
gcloud services enable aiplatform.googleapis.com   # the "Agent Platform API"

# 3. Grant the caller the Agent Platform User role
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:you@example.com" \
  --role="roles/aiplatform.user"
```

> ✅ **No `GOOGLE_API_KEY`.** On Google Cloud, ADK authenticates with **ADC** (user credentials) for local development, or a **service account** in production. In production/CI, either rely on the environment-provided identity (on Agent Runtime / Cloud Run / GKE) or set `GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account-key.json` explicitly when running externally. Source: <https://adk.dev/get-started/google-cloud/>

> 🔄 **Grounded — API + role naming.** The enable command is still `aiplatform.googleapis.com`, but Google Cloud documentation now brands it the **Agent Platform API** and the role **Agent Platform User** (`roles/aiplatform.user`). Earlier tutorials called this "Vertex AI". Same API, current product name.

> ⚠️ **Version note on A2UI.** A2UI is a JSON streaming UI protocol; the ADK integration ships via the Python SDK package **`a2ui-agent-sdk`** and is **transport-agnostic** (works over A2A, MCP, REST, WebSockets, or any protocol). The A2UI specification and component catalog live at <https://a2ui.org/>. Source: <https://adk.dev/integrations/a2ui/>

---

## Part 1 — Background: what A2UI is, and where it sits

### The problem
In a multi-agent system the agent doing the work is often *remote* — a different server, a different company. It can't touch your DOM. The old workaround was shipping HTML/JS into a sandboxed iframe: heavy, visually disjointed, and a security headache (XSS, UI injection).

**A2UI's answer: transmit UI that is safe like data, but expressive like code.** As the ADK docs put it: *"The agent describes what to show; the client decides how to render it."* Source: <https://adk.dev/integrations/a2ui/>

### The core idea
The agent emits structured JSON describing the UI. A renderer on the client turns it into interactive components (cards, forms, charts, tables). The client renders only components from a catalog it already trusts — no arbitrary code runs.

### Three properties that make it work
| Property | Meaning |
|---|---|
| **Declarative, not executable** | It's data. The client renders catalog components only — no code execution, no UI injection. |
| **Streaming-friendly** | Structured JSON messages, so the client can paint as they arrive over any transport (A2A, MCP, REST, WebSockets). |
| **Framework-agnostic / transport-agnostic** | The same payload renders through many clients. The agent doesn't know or care. ADK's A2UI docs stress it works "over A2A, MCP, REST, WebSockets, or any other protocol." |

### The four layers (don't conflate them)
| Layer | Owns | Examples |
|---|---|---|
| App experience | Chat shell + conversation state | CopilotKit, AG-UI |
| Pixel drawing | Turning component descriptions into UI | React, Angular, Vue, React Native, Flutter, Jetpack Compose |
| Conversation pipeline | Client ↔ server transport | **AG-UI**, A2A, MCP, SSE, WebSockets |
| **Cargo (data format)** | **The thing describing the UI** | **A2UI** |

For **this** tutorial: **ADK agent (Gemini on Google Cloud Agent Platform) → `ag-ui-adk` middleware → AG-UI transport → CopilotKit React renderer → A2UI components.**

### A2UI vs A2A vs AG-UI
- **A2A** = agent ↔ agent. (A2UI can *travel* over A2A, wrapped as a `DataPart`.)
- **A2UI** = agent ↔ **human** (structured UI).
- **AG-UI** = the transport carrying agent/UI events between agent and app. (A2UI can also travel over AG-UI.)

> 🔄 **Grounded — A2UI over A2A.** The ADK A2UI page shows A2UI payloads wrapped in an A2A `DataPart` with MIME type `application/json+a2ui` so renderers can identify them, and shows a multi-agent orchestrator aggregating catalog IDs from sub-agent cards. Source: <https://adk.dev/integrations/a2ui/>

---

## Part 2 — Two ways to emit A2UI (choose before you code)

Google's ADK documentation presents two documented patterns, and they are more specific than "fixed vs dynamic":

| Mode | Who owns the layout | How it works | Use for |
|---|---|---|---|
| **Schema-managed (recommended default)** | A schema manager injects catalogs + examples into the prompt; the LLM outputs A2UI JSON that you **validate** before sending | `A2uiSchemaManager` generates the system prompt; `parse_response` + catalog validator gate the output | Most agents — you keep full control of which components are allowed |
| **Multi-agent / dynamic catalog** | Orchestrator + sub-agents compose from context-selected catalogs | Resolve catalog at runtime from client capabilities, store in session state, aggregate catalog IDs in the agent card | Charts-for-data, forms-for-config, delegated UI |

> 🔄 **Grounded — the documented ADK path.** The official ADK A2UI integration is built around **`a2ui-agent-sdk`** (`A2uiSchemaManager` / `BasicCatalog` / `parse_response` / catalog validators), *not* an auto-injected `generate_a2ui` tool. This tutorial builds the schema-managed pattern first, then adds a multi-agent/dynamic-catalog variant. Source: <https://adk.dev/integrations/a2ui/>

---

## Part 3 — Step-by-step: the ADK agent on Google Cloud Agent Platform

### Step 3.1 — Scaffold

```bash
# Python 3.10+ required
python3 -m venv .venv && source .venv/bin/activate
pip install google-adk ag-ui-adk "uvicorn[standard]" fastapi
# For the A2UI schema manager:
pip install a2ui-agent-sdk
```

> 🔄 **Grounded — dependencies.** ADK installs as `google-adk`; the AG-UI middleware package is **`ag-ui-adk`** (import name `ag_ui_adk`); the A2UI Python SDK is **`a2ui-agent-sdk`**. CopilotKit docs list backend packages as `google-adk ag-ui-adk fastapi "uvicorn[standard]"`. Sources: ADK install docs; <https://adk.dev/integrations/copilotkit/>

### Step 3.2 — Point ADK at Google Cloud Agent Platform

ADK selects the backend from environment variables. **Unset `GOOGLE_API_KEY` and set the Google Cloud variables instead:**

```bash
# --- Google Cloud Agent Platform configuration (replaces GOOGLE_API_KEY) ---
export GOOGLE_GENAI_USE_ENTERPRISE=TRUE       # current name (see note)
export GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
export GOOGLE_CLOUD_LOCATION=us-central1     # a region where your model is available
# Optional alternative to ADC (only when running externally):
# export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account-key.json

# Make sure no stray key shadows Google Cloud:
unset GOOGLE_API_KEY
```

| Variable | Purpose |
|---|---|
| `GOOGLE_GENAI_USE_ENTERPRISE` | `TRUE` routes the Google GenAI client through Google Cloud Agent Platform instead of the Gemini Developer API |
| `GOOGLE_CLOUD_PROJECT` | Your Google Cloud project id |
| `GOOGLE_CLOUD_LOCATION` | Region (e.g. `us-central1`); must support your Gemini model |
| `GOOGLE_APPLICATION_CREDENTIALS` | *Optional* — explicit service-account JSON; omit to use ADC |

> 🔄 **Grounded — variable rename.** Google's ADK docs state: **"`GOOGLE_GENAI_USE_ENTERPRISE` was previously `GOOGLE_GENAI_USE_VERTEXAI`. These variable names are equivalent and do the same thing."** If you set `GOOGLE_GENAI_USE_ENTERPRISE` and your agent does not connect, you're on an older ADK version — use `GOOGLE_GENAI_USE_VERTEXAI` instead, or upgrade. Sources: <https://adk.dev/get-started/google-cloud/>, <https://adk.dev/deploy/cloud-run/>

**Expected result:** `gcloud auth application-default print-access-token` succeeds, and no API key is present in the environment.

### Step 3.3 — Baseline agent (text only, no A2UI yet)

`agent.py`:
```python
from fastapi import FastAPI
from google.adk.agents import Agent          # LlmAgent is aliased as Agent
from google.adk.apps import App, ResumabilityConfig
from ag_ui_adk import ADKAgent, AGUIToolset, add_adk_fastapi_endpoint

root_agent = Agent(
    model="gemini-2.5-flash",                # see the model-id note below
    name="assistant",
    instruction="Be helpful and fun!",
)

adk_app = App(
    name="demo_app",
    root_agent=root_agent,
    resumability_config=ResumabilityConfig(is_resumable=True),
)

ag_ui_agent = ADKAgent.from_app(
    adk_app,
    user_id="demo_user",
    use_in_memory_services=True,
)

app = FastAPI()
add_adk_fastapi_endpoint(app, ag_ui_agent, path="/ag-ui")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="localhost", port=8000)
```

> 🔄 **Grounded — agent API + middleware factory.** ADK aliases `LlmAgent` as `Agent` (`from google.adk.agents import Agent`). The CopilotKit/ADK docs create the AG-UI middleware via **`ADKAgent.from_app(adk_app, ...)`** with an `App(...)` + `ResumabilityConfig`, and expose it with `add_adk_fastapi_endpoint(app, agent, path="/ag-ui")`. Sources: <https://adk.dev/agents/llm-agents/>, <https://adk.dev/integrations/copilotkit/>

> 🔄 **Grounded — model ids.** ADK docs recommend **`gemini-flash-latest`** as the "latest available" Gemini Flash selector, but warn: *"if you access Gemini from a regional endpoint, such as `us-central1`, this selection string may not work. In that case, use a specific model version string."* So on a regional endpoint, prefer an explicit id like `gemini-2.5-flash`. Source: <https://adk.dev/agents/models/google-gemini/>

Run: `python agent.py` (or `uvicorn agent:app --reload --port 8000`) → **Expected result:** server on `http://localhost:8000/ag-ui` returning plain text, with every model call billed to your Google Cloud project.

> 🧭 **Explicit-model alternative.** To pin the model object in code rather than via env vars:
> ```python
> from google.adk.models.google_llm import Gemini
> model = Gemini(model="gemini-2.5-flash")   # honours the Google Cloud env vars
> ```
> Keep the env vars set either way — they tell ADK *which backend* to use.

### Step 3.4 — Add schema-managed A2UI (the documented default)

The ADK A2UI SDK loads component catalogs and generates a system prompt that teaches the LLM how to produce valid A2UI JSON. You then **validate** the output before sending it to the client.

```python
from a2ui.core.schema.manager import A2uiSchemaManager
from a2ui.basic_catalog.provider import BasicCatalog
from a2ui.core.parser.parser import parse_response
from a2ui.a2a import parse_response_to_parts, create_a2ui_part
from google.adk.agents.llm_agent import LlmAgent

# 1) Schema manager — omit `catalogs` to use the A2UI team's Basic Catalog
schema_manager = A2uiSchemaManager(
    catalogs=[BasicCatalog.get_config(examples_path="examples")],
)

# 2) Generate the system prompt (role + schema + few-shot examples)
instruction = schema_manager.generate_system_prompt(
    role_description="You are a helpful assistant that presents information with rich UI.",
    workflow_description="Analyze the user's request and return structured UI when appropriate.",
    ui_description="Use cards for summaries, tables for comparisons, and forms for user input.",
    include_schema=True,
    include_examples=True,
    allowed_components=["Heading", "Text", "Card", "Button", "Table"],
)

# 3) The ADK agent uses that instruction
ui_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="ui_agent",
    description="An agent that generates rich UI responses.",
    instruction=instruction,
)

# 4) Validate LLM output before it reaches the client
selected_catalog = schema_manager.get_selected_catalog()
response_parts = parse_response(llm_output_text)
for part in response_parts:
    if part.a2ui_json:
        selected_catalog.validator.validate(part.a2ui_json)

# Or the one-liner that returns A2A Parts, with a text fallback:
parts = parse_response_to_parts(
    llm_output_text,
    validator=selected_catalog.validator,
    fallback_text="Here's what I found.",
)

# A2UI payloads are wrapped so renderers can identify them:
part = create_a2ui_part({"type": "Card", "props": {"title": "Hello"}})
# → DataPart(data={...}, metadata={"mimeType": "application/json+a2ui"})
```

**Why it works:** the schema manager injects the catalog + examples into the prompt and hands you a validator; you gate the model's JSON against that validator before it hits the wire. The schema manager auto-detects the A2UI version from the incoming client request, or you can pin it explicitly (`version=VERSION_0_9`). Source: <https://adk.dev/integrations/a2ui/>

### Step 3.5 — Dynamic catalogs / multi-agent variant (optional, same app)

For agents that need different components by context, resolve the catalog at runtime from client capabilities and store it in session state:

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

Custom catalogs (mix the Basic Catalog with your own, from a JSON path):
```python
from a2ui.core.schema.manager import CatalogConfig
schema_manager = A2uiSchemaManager(
    catalogs=[
        BasicCatalog.get_config(),
        CatalogConfig.from_path(
            name="my_dashboard_catalog",
            catalog_path="catalogs/dashboard.json",
            examples_path="catalogs/dashboard_examples",
        ),
    ],
)
```

> 📎 **Samples to run immediately** (from the A2UI repo): `restaurant_finder` (static schema), `rizzcharts` (dynamic catalog), `orchestrator` (multi-agent, aggregates UI capabilities). Source: <https://adk.dev/integrations/a2ui/>

### Step 3.6 — Frontend tools + Human-in-the-loop (optional, ADK ≥ 1.16.0)

`AGUIToolset()` makes frontend-registered tools callable by the agent; combining it with `ADKAgent.from_app()` + `ResumabilityConfig` lets a run **pause on a frontend tool call and resume when the result arrives**. This is the same `App`/`ResumabilityConfig` pattern shown in Step 3.3.

```python
from google.adk.agents import Agent
from google.adk.apps import App, ResumabilityConfig
from ag_ui_adk import ADKAgent, AGUIToolset, add_adk_fastapi_endpoint

root_agent = Agent(
    model="gemini-2.5-flash",
    name="copilotkit_agent",
    instruction="You are a helpful assistant. Use the frontend tools when they fit the request.",
    tools=[AGUIToolset()],                      # exposes frontend tools to the agent
)

adk_app = App(
    name="copilotkit_app",
    root_agent=root_agent,
    resumability_config=ResumabilityConfig(is_resumable=True),
)

ag_ui_agent = ADKAgent.from_app(adk_app, user_id="local_user", use_in_memory_services=True)
```

> 🔄 **Grounded — resumability version.** ADK's resume docs state: *"In ADK Python 1.16 and higher, you can configure an ADK workflow to be resumable."* Sources: <https://adk.dev/runtime/resume/>, <https://adk.dev/integrations/copilotkit/>

> 🏢 **Production session store.** `use_in_memory_services=True` is for local dev (sessions/artifacts lost on restart). For production on Google Cloud, use **`VertexAiSessionService`** — it persists sessions via Agent Runtime and picks up project/location from your Google Cloud config:
> ```python
> # Requires: pip install google-adk[gcp]
> from google.adk.sessions import VertexAiSessionService
> session_service = VertexAiSessionService(project="YOUR_PROJECT_ID", location="us-central1")
> # app_name = the Reasoning Engine ID/name:
> # "projects/<project>/locations/us-central1/reasoningEngines/<engine-id>"
> ```
> Source: <https://adk.dev/sessions/session/>

---

## Part 4 — Runtime + CopilotKit frontend

### Step 4.1 — Create the Next.js app

```bash
npx create-next-app@latest my-copilot-app
cd my-copilot-app
npm install @copilotkit/react-core @copilotkit/runtime @ag-ui/client hono zod
```

> 🔄 **Grounded — packages + versions.** The ADK CopilotKit docs install `@copilotkit/react-core @copilotkit/runtime @ag-ui/client hono zod` and use the **v2** entry points (`@copilotkit/react-core/v2`, `@copilotkit/runtime/v2`). Note the runtime route uses **`hono`** (`import { handle } from "hono/vercel"`). Source: <https://adk.dev/integrations/copilotkit/>

### Step 4.2 — Wire the runtime to your ADK endpoint

`app/api/copilotkit/[[...slug]]/route.ts`:
```ts
import { HttpAgent } from "@ag-ui/client";
import {
  CopilotRuntime,
  InMemoryAgentRunner,
  createCopilotEndpoint,
} from "@copilotkit/runtime/v2";
import { handle } from "hono/vercel";

const runtime = new CopilotRuntime({
  agents: {
    default: new HttpAgent({
      url: process.env.ADK_AG_UI_URL ?? "http://localhost:8000/ag-ui",
    }),
  },
  runner: new InMemoryAgentRunner(),
});

const app = createCopilotEndpoint({
  runtime,
  basePath: "/api/copilotkit",
});

export const GET = handle(app);
export const POST = handle(app);
export const PATCH = handle(app);
export const DELETE = handle(app);
```

> 🔄 **Grounded — runtime wiring.** The documented v2 pattern uses **`createCopilotEndpoint`** + **`handle` from `hono/vercel`**, an **`InMemoryAgentRunner`**, and points `HttpAgent` at the `/ag-ui` path you mounted in Step 3.3. Source: <https://adk.dev/integrations/copilotkit/>

### Step 4.3 — Mount the provider + chat

`app/providers.tsx`:
```tsx
"use client";

import { CopilotKit } from "@copilotkit/react-core/v2";
import "@copilotkit/react-core/v2/styles.css";

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <CopilotKit runtimeUrl="/api/copilotkit" useSingleEndpoint={false}>
      {children}
    </CopilotKit>
  );
}
```

`app/page.tsx`:
```tsx
"use client";

import { CopilotChat } from "@copilotkit/react-core/v2";

export default function Page() {
  return (
    <main style={{ height: "100vh" }}>
      <CopilotChat agentId="default" />
    </main>
  );
}
```

> 🔄 **Grounded — provider props.** The docs mount `<CopilotKit runtimeUrl="/api/copilotkit" useSingleEndpoint={false}>` and render `<CopilotChat agentId="default" />` from `@copilotkit/react-core/v2`; the styles import is `@copilotkit/react-core/v2/styles.css`. Source: <https://adk.dev/integrations/copilotkit/>

### Step 4.4 — Register a frontend tool (browser-side)

Frontend tools are registered in the browser and exposed to the agent by `AGUIToolset()` on the ADK side:

```tsx
"use client";

import { useFrontendTool } from "@copilotkit/react-core/v2";
import { z } from "zod";

export function SearchTool() {
  useFrontendTool({
    name: "searchDocs",
    description: "Search the current application documentation.",
    parameters: z.object({ query: z.string() }),
    handler: async ({ query }) => {
      const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
      return response.text();
    },
  });
  return null;
}
```

**Available CopilotKit hooks (all from `@copilotkit/react-core/v2`):** `useFrontendTool`, `useRenderTool`, `useComponent`, `useHumanInTheLoop`, `useAgentContext`, `useAgent`. Source: <https://adk.dev/integrations/copilotkit/>

### Step 4.5 — Run it
Terminal 1: `uvicorn agent:app --reload --port 8000` (with the Google Cloud env vars exported) · Terminal 2: `npm run dev` → open `localhost:3000`.

**Expected result:** the agent renders catalog components as it streams and a custom/frontend tool call round-trips back to the agent.

---

## Part 5 — Production deploy on Cloud Run

Google documents two paths to Cloud Run. **`adk deploy cloud_run` is the recommended path for Python**; `gcloud run deploy` with a `Dockerfile` is the manual alternative.

### Option A — `adk deploy cloud_run` (recommended)

```bash
gcloud auth login
gcloud config set project $GOOGLE_CLOUD_PROJECT
export GOOGLE_CLOUD_LOCATION="us-central1"
export AGENT_PATH="./my_agent"        # must contain __init__.py + agent.py

adk deploy cloud_run \
  --project=$GOOGLE_CLOUD_PROJECT \
  --region=$GOOGLE_CLOUD_LOCATION \
  --service_name=my-agent-service \
  --app_name=my_agent_app \
  --session_service_uri=agentengine://<agent_engine> \
  --with_ui \
  $AGENT_PATH
```

> 🔑 **Do not lose sessions in production.** ADK's deploy docs warn: *"When `--session_service_uri` and `--artifact_service_uri` are not set, the deployed container falls back to the in-memory session and artifact services, and sessions and artifacts are lost whenever a Cloud Run instance is recycled."* Set `agentengine://<agent_engine>` (managed Agent Runtime session service) or a SQLAlchemy URL. Source: <https://adk.dev/deploy/cloud-run/>

### Option B — `gcloud run deploy` (manual Dockerfile)

```bash
gcloud run deploy a2ui-adk-agent \
  --source . \
  --project $GOOGLE_CLOUD_PROJECT \
  --region $GOOGLE_CLOUD_LOCATION \
  --set-env-vars="GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,GOOGLE_GENAI_USE_ENTERPRISE=$GOOGLE_GENAI_USE_ENTERPRISE"
```

> ⚠️ `--set-env-vars` is **destructive**: it deletes previously set environment variables not included in the new list. Source: <https://docs.cloud.google.com/run/docs/configuring/services/environment-variables>

> 🔄 **Grounded — auth on Cloud Run.** On Google Cloud (Agent Runtime, Cloud Run, GKE) the environment **automatically provides service-account credentials** — no key file to bake in. Grant that service account the Agent Platform User role (`roles/aiplatform.user`). Only when running *externally* do you create a service-account key and set `GOOGLE_APPLICATION_CREDENTIALS`. Source: <https://adk.dev/get-started/google-cloud/>

> ℹ️ If you deploy via `adk deploy cloud_run`, the command uses Google Cloud Build; give the default compute service account `roles/cloudbuild.builds.builder`. Source: <https://adk.dev/deploy/cloud-run/>

---

## Part 6 — What the wire actually looks like (debug view)

| Layer | What you see |
|---|---|
| AG-UI stream | Run lifecycle events, tool-call start/args for the agent's tools, streamed UI payloads, and a run-finished/run-error terminator |
| Middleware | `ag-ui-adk` forwards runs, exposes frontend tools via `AGUIToolset`, and supports pausing/resuming on frontend tool calls |
| A2UI payload | Wrapped so renderers can identify it: `DataPart(data={...}, metadata={"mimeType": "application/json+a2ui"})` (A2A form) |

> 🔄 **Grounded — MIME marker.** The A2UI ADK docs show A2UI payloads carried as an A2A `DataPart` with MIME type `application/json+a2ui`. Source: <https://adk.dev/integrations/a2ui/>

---

## Part 7 — Verification checklist

- [ ] `gcloud auth application-default print-access-token` succeeds; **no `GOOGLE_API_KEY` set** (for the Google Cloud path).
- [ ] `GOOGLE_GENAI_USE_ENTERPRISE=TRUE` (or legacy `GOOGLE_GENAI_USE_VERTEXAI`), project, and location are exported.
- [ ] Python 3.10+ and Node 18+ in use; typecheck/lint pass.
- [ ] The agent streams a run to completion, ending with a run-finished **or** run-error event (never swallowed).
- [ ] Schema-managed: LLM output is **validated against the selected catalog** before sending.
- [ ] **A surface renders — not just a text explanation.**
- [ ] A frontend tool call round-trips back to the agent.
- [ ] No schema/hydration/stream/action-bridge errors in console or backend logs.

---

## Part 8 — Common pitfalls

| Symptom | Likely cause | Fix |
|---|---|---|
| `403 PERMISSION_DENIED` / `aiplatform` errors | Caller lacks Agent Platform access | Grant `roles/aiplatform.user`; `gcloud auth application-default login` |
| Agent won't connect to Google Cloud after upgrade | Old `GOOGLE_GENAI_USE_VERTEXAI` vs new `..._ENTERPRISE` mismatch | Use `GOOGLE_GENAI_USE_ENTERPRISE=TRUE` on current ADK; or set `..._VERTEXAI` on older ADK |
| Model calls fail after switching | Stray `GOOGLE_API_KEY` shadowing Google Cloud | `unset GOOGLE_API_KEY`; keep only the Google Cloud vars |
| `Location ... not supported` / model selector fails | Region doesn't host the model, or `gemini-flash-latest` used on a regional endpoint | Pick a supported `GOOGLE_CLOUD_LOCATION`; use an explicit model id like `gemini-2.5-flash` |
| Sessions vanish on redeploy/scale-down | In-memory session service in production | Set `--session_service_uri=agentengine://<agent_engine>` or a SQLAlchemy URL |
| No surface appears | A2UI not enabled/wired end-to-end | Generate + validate A2UI on the backend and render catalog components on the client |
| Agent *describes* the UI in prose | System prompt lacks the A2UI schema/examples | Generate the prompt with `A2uiSchemaManager.generate_system_prompt(include_schema=True, include_examples=True)` |
| Invalid UI JSON reaches client | Output not validated | Gate with `selected_catalog.validator.validate(...)` or `parse_response_to_parts(..., validator=...)` |
| Frontend tool never runs | `AGUIToolset()` not attached | Add `tools=[AGUIToolset()]` to the ADK agent |
| `--set-env-vars` wiped other vars | Flag is destructive | Re-list **all** env vars each time, or use a config file |
| Cloud Build deploy fails | Default compute SA lacks build permission | Grant `roles/cloudbuild.builds.builder` |
| 429 RESOURCE_EXHAUSTED | Quota exceeded | Request higher quota; enable client-side retries via `HttpRetryOptions` |

---

## Part 9 — Product/name changes to watch (was "Vertex AI")

| Old name in tutorials | Current documented name | Notes |
|---|---|---|
| Vertex AI (agent platform) | **Google Cloud Agent Platform** (Gemini Enterprise Agent Platform) | Same platform; docs rebranded |
| `GOOGLE_GENAI_USE_VERTEXAI` | **`GOOGLE_GENAI_USE_ENTERPRISE`** | Both work; `ENTERPRISE` needs newer ADK |
| "Vertex AI API" | **Agent Platform API** (`aiplatform.googleapis.com`) | Same enable command |
| "Vertex AI User" | **Agent Platform User** (`roles/aiplatform.user`) | Same role |
| Vertex AI Session Service | `VertexAiSessionService` (`google-adk[gcp]`) | Managed via Agent Runtime |

> Source: <https://adk.dev/get-started/google-cloud/>, <https://adk.dev/deploy/cloud-run/>

---

## What's next

- **A2UI spec + catalog + renderers:** <https://a2ui.org/> · repo <https://github.com/a2ui-project/a2ui>
- **A2UI Python SDK:** `a2ui-agent-sdk` on PyPI
- **ADK × A2UI:** <https://adk.dev/integrations/a2ui/>
- **ADK × AG-UI:** <https://adk.dev/integrations/ag-ui/>
- **ADK × CopilotKit:** <https://adk.dev/integrations/copilotkit/>
- **Connect ADK to Google Cloud:** <https://adk.dev/get-started/google-cloud/>
- **ADK deploy (Cloud Run):** <https://adk.dev/deploy/cloud-run/>
- **AG-UI docs:** <https://docs.ag-ui.com> · **AG-UI Dojo:** <https://dojo.ag-ui.com>

**Takeaway:** with ADK on Google Cloud Agent Platform you build the *agent* and the *A2UI it emits*; authentication is silent (ADC / service account), the `ag-ui-adk` middleware handles transport, and CopilotKit owns the shell and rendering. No API keys to leak — just a project, a region, and the Agent Platform User role.

---

### Sources (all retrieved/validated via the Google Developer Knowledge MCP server)

1. A2UI — Agent-to-UI for ADK — https://adk.dev/integrations/a2ui/
2. AG-UI user interface for ADK — https://adk.dev/integrations/ag-ui/
3. CopilotKit user interface for ADK — https://adk.dev/integrations/copilotkit/
4. Connect to Google Cloud and Agent Platform — https://adk.dev/get-started/google-cloud/
5. Google Gemini models for ADK agents — https://adk.dev/agents/models/google-gemini/
6. Simple agents with LlmAgent — https://adk.dev/agents/llm-agents/
7. Sessions — https://adk.dev/sessions/session/
8. Resume stopped agents (HITL) — https://adk.dev/runtime/resume/
9. Deploy to Cloud Run — https://adk.dev/deploy/cloud-run/
10. A2UI specification — https://a2ui.org/
11. AG-UI docs — https://docs.ag-ui.com
12. Cloud Run environment variables — https://docs.cloud.google.com/run/docs/configuring/services/environment-variables
