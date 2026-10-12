# AgentWriter-AI

Multi-agent technical blog writer built with **LangGraph**, **FastAPI** and **PostgreSQL**.

Give it a topic. It decides whether web research is needed, plans the article, writes every
section in parallel, generates illustrations, and streams the whole process to a web UI in real time.

---

## Table of contents

- [Features](#features)
- [Architecture](#architecture)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [LLM providers and models](#llm-providers-and-models)
- [API reference](#api-reference)
- [Project structure](#project-structure)
- [Troubleshooting](#troubleshooting)
- [Known limitations](#known-limitations)
- [License](#license)

---

## Features

- **Agentic workflow** with LangGraph: routing, optional research, planning, parallel writing, assembly.
- **Grounded writing**: in research modes, claims are cited with links taken from the collected evidence.
- **Real-time UI**: stages, plan, finished sections and final article streamed with Server-Sent Events.
- **Model selection** in the UI: OpenRouter, Google Gemini or Groq (free models available).
- **Image generation** with Cloudflare Workers AI (FLUX).
- **Durable state**: graph checkpoints stored in PostgreSQL through a connection pool.
- **Markdown output**: live preview, copy to clipboard and download.
- **Optional tracing** with LangSmith.

## Architecture

```
START
  │
  ▼
router ──(needs research)──► research ──┐
  │                                     │
  └────(no research)────────────────────┤
                                        ▼
                                  orchestrator        builds the plan
                                        │
                              fan-out (Send per section)
                                        ▼
                                workers (parallel)    one per section
                                        │
                                        ▼
                      reducer (subgraph)
                        merge_content ► decide_images ► generate_and_place_images
                                        │
                                        ▼
                                       END
```

| Node | Responsibility |
| --- | --- |
| `router` | Chooses a mode: `closed_book` (evergreen), `hybrid` (needs fresh facts) or `open_book` (news roundup), and produces search queries. |
| `research` | Runs Tavily searches and extracts a deduplicated list of evidence (title, URL, date). |
| `orchestrator` | Produces a structured plan: title, audience, tone and 5–9 sections with goal, bullets and target length. |
| `worker` | Writes one section in Markdown. Runs in parallel for all sections. |
| `merge_content` | Orders and concatenates the sections under the article title. |
| `decide_images` | Lets the LLM choose up to 3 places for images and write their prompts. |
| `generate_and_place_images` | Generates the images and inserts them in the Markdown. A failed image becomes a visible note instead of crashing the run. |

## Requirements

- Python 3.11 or newer
- A PostgreSQL database (Supabase, Render, or any other)
- API keys for the services you use (see [Configuration](#configuration))

## Installation

```bash
git clone <repository-url>
cd AgentWriter-AI

python -m venv .venv
.venv\Scripts\activate            # Windows
# source .venv/bin/activate       # macOS / Linux

pip install -r requirements.txt
```

## Configuration

Create a `.env` file at the project root. It is ignored by Git.

```env
# Database (required)
DATABASE_URL=postgresql://user:password@host:5432/postgres

# LLM providers: set the key of each provider you want to use
OPENROUTER_API_KEY=
GOOGLE_API_KEY=
GROQ_API_KEY=

# Web research (required for hybrid / open_book topics)
TAVILY_API_KEY=

# Image generation (Cloudflare Workers AI)
CLOUDFLARE_ACCOUNT_ID=
CLOUDFLARE_API_TOKEN=

# Optional: LangSmith tracing
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_PROJECT=AgentWriter-AI
```

| Variable | Required | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | Yes | PostgreSQL connection string for the LangGraph checkpointer. `sslmode=require` is added automatically if missing. |
| `OPENROUTER_API_KEY` / `GOOGLE_API_KEY` / `GROQ_API_KEY` | At least one | Key of the provider selected in the UI. |
| `TAVILY_API_KEY` | For research | Web search. |
| `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_API_TOKEN` | For images | Image generation. Without them, images are replaced by an error note. |
| `LANGSMITH_*` | No | Tracing and debugging. |

## Usage

```bash
python app.py
```

Open <http://127.0.0.1:8000>, enter a topic, pick a provider and a model, then click **Run agent**.

Each run is saved to `outputs/<run_id>/blog.md`. Generated images are stored in `images/`.

## LLM providers and models

The list of models offered in the UI is defined by `MODEL_CATALOG` in [backend.py](backend.py).
A provider whose API key is missing is shown as "(no API key)" and is refused by the API.

| Provider | Models |
| --- | --- |
| OpenRouter | `nvidia/nemotron-3-ultra-550b-a55b:free` (default), `openai/gpt-oss-120b:free`, `openai/gpt-oss-20b:free`, `meta-llama/llama-3.3-70b-instruct:free` |
| Google Gemini | `gemini-3.5-flash-lite`, `gemini-3.1-flash-lite`, `gemini-3.8-flash` |
| Groq | `llama-3.3-70b-versatile`, `openai/gpt-oss-120b`, `openai/gpt-oss-20b`, `qwen/qwen3-32b` |

Model names, especially free ones, change often. If a model returns "not found", update `MODEL_CATALOG`.

Images use `@cf/black-forest-labs/flux-2-klein-9b` (`multipart/form-data` request, base64 response).

## API reference

| Method | Route | Description |
| --- | --- | --- |
| `GET` | `/` | Web interface |
| `GET` | `/api/health` | Health check |
| `GET` | `/api/models` | Providers, models and whether each API key is configured |
| `POST` | `/api/run` | Starts a run and streams events (SSE). Body: `{"topic": "...", "provider": "...", "model": "..."}` |
| `GET` | `/api/runs/{run_id}/download` | Downloads the generated Markdown |

## Project structure

```
app.py              FastAPI server and SSE streaming of the workflow
backend.py          LangGraph workflow: router, research, orchestrator, workers, reducer
templates/          HTML page
static/             CSS and JavaScript of the interface
images/             Generated images (served at /images)
outputs/<run_id>/   Generated articles (blog.md)
```

## Troubleshooting

| Symptom | Likely cause and fix |
| --- | --- |
| `failed to resolve host 'db.<ref>.supabase.co'` | The direct Supabase host is IPv6-only. Use the **Session pooler** connection string (`...pooler.supabase.com:5432`, user `postgres.<ref>`) in `DATABASE_URL`. |
| `server closed the connection unexpectedly` | Network drop or idle connection closed by the database. The app uses a connection pool that replaces dead connections; check your network. |
| Article contains only the title | The LLM returned incomplete output during image planning. The app falls back to placing the image markers itself; try a stronger model. |
| `Cloudflare API Error` or an image block marked "FAILED" | Check `CLOUDFLARE_ACCOUNT_ID` / `CLOUDFLARE_API_TOKEN` and your Workers AI quota. |
| Images cannot be opened | Delete the files in `images/` and run again: existing files are never regenerated. |
| A model returns "not found" | Update `MODEL_CATALOG` in `backend.py`. |

## Known limitations

- Image models cannot reliably render text: labels inside generated diagrams may contain spelling mistakes.
- Free LLMs can be slow, rate-limited or return incomplete structured output.
- Existing files in `images/` are reused, not regenerated.

## License

See [LICENSE](LICENSE).
