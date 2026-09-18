# Overview

This is an AI-powered content moderation system for customer service interactions at a fictional company called ACME Enterprise.

What it does:
- Moderates text, images, videos, and audio before they're sent to customers
- Detects issues like: PII (personally identifiable information), unprofessional tone, unfriendly content, disturbing images/videos, and low-quality media
- Blocks harmful content and provides detailed explanations for why content was flagged

The app allows a trainee customer agent to interact with a simulated angry customer, played by an LLM. The LLM-customer has bought a product from ACME (the ACME Power Widget Pro) and the product stopped working. The customer agent needs to handle this case by
chatting with the customer, and every message and content provided by the trainee agent is moderated and observed to make sure all
communications following company standards.

Architecture:
1. Specialized Agents - Four moderation agents (text, image, video, audio), each using Google Gemini AI with custom prompts to check their specific content type (agents are in the `agents` folder)
2. LLM-as-a-customer: an agent (`agents/customer_agent.py`)
2. Structured Results - Each agent returns a Pydantic model with specific flags (e.g., contains_pii, is_unfriendly) plus a rationale. 
   The definitions are in the the `types/moderation_result.py` folder
3. Frontend: Gradio Chat UI - Interactive web interface where users can chat and upload files, with real-time moderation. This is `gradio_app.py`.
4. Backend: FastAPI REST API - HTTP endpoints for programmatic access (/moderate/text, /moderate/image, etc.). These are provided as-is. They just wrap in HTTP endpoints the functionality offered by the agents. This is `fastapi_app.py`. The division in frontend/backend services is typical of web applications, and allow different front-ends to utilize the same services from the backend. For example, in the hypothetical scenario of this app, after the initial PoC phase using the Gradio app, we might want to move to a more production-grade React/Vue/Angular app. This new app can use the same backend, and the two apps can even co-exist for a time until the new app is proved to work. No change is needed in the AI agents or on the backend.
5. Observability - Phoenix integration for tracing and monitoring AI agent behavior. Some setup is in `tracing.py`.
6. A convenience executable that starts the 3 services: the backend (fastAPI APIs), the frontend (the gradio app) as well as 
   Arize Phoenix for tracing.
   
It's generative Gemini models (reasoning/thinking explicitly turned off), combined via late (decision-level) fusion across independent per-modality agents.


## Agent Roles & Architecture

| Role | Is it AI? | Where it lives | How it's invoked |
|---|---|---|---|
| Customer | Yes — Gemini via `pydantic_ai.Agent` | `agents/customer_agent.py` | `customer_agent.run(...)` inside `ChatSessionWithTracing.chat_with_gemini()` in `gradio_app.py` |
| Trainee/agent | No — human | No agent file; it's the Gradio UI's chat input | Text/files typed/uploaded by the user in the browser |
| Moderation agents (text/image/video/audio) | Yes — Gemini, but a different role (content reviewer, not persona) | `agents/text_agent.py`, `image_agent.py`, `video_agent.py`, `audio_agent.py` | Called via FastAPI (`fastapi_app.py`) before the human's message is allowed to reach `customer_agent` |


## Requirements and Tools
### Core dependencies

| Library | Why it's used |
| :---- | :---- |
| `pydantic-ai` | Agent framework. Every agent (`text_agent`, `image_agent`, `video_agent`, `audio_agent`, `customer_agent`) is a `pydantic_ai.Agent` — it handles calling the LLM, passing multimodal input (`BinaryContent`), and forcing structured output. |
| `google-genai` | SDK that `pydantic-ai`'s `GoogleModel`/`GoogleProvider` use to talk to the Gemini API (auth, requests/responses). |
| `pydantic` | Defines the structured output schemas (`TextModerationResult`, `ImageModerationResult`, `VideoModerationResult`, `AudioModerationResult` in `types/moderation_result.py`), so Gemini's replies come back as validated typed objects instead of raw text to parse. |
| `fastapi` | Backend API layer (`fastapi_app.py`) exposing `/api/v1/moderate_text`, `/moderate_image_file`, etc., with `HTTPBearer` auth. Decouples moderation logic from any specific frontend. |
| `uvicorn[standard]` | ASGI server that runs the FastAPI app. |
| `gradio` | Frontend chat UI (`gradio_app.py`) — the trainee's chat box, file upload widget, and the interface that calls the FastAPI backend and then the `customer_agent`. |
| `requests` | Used by `gradio_app.py` to call the FastAPI moderation endpoints over HTTP. |
| `filetype` | Used in `utils.py`'s `detect_file_type()` to identify uploaded files (image/video/audio) by content instead of trusting the filename extension. |
| `python-dotenv` | Loads `.env` (`GEMINI_API_KEY`, `USER_API_KEY`, `DEFAULT_GOOGLE_MODEL`) into environment variables at startup. |
| `arize-phoenix` | Observability/tracing UI (`app.py` calls `phoenix.launch_app()`) for inspecting every agent call, prompt, and moderation decision at `localhost:6006`. |
| `openinference-instrumentation-pydantic-ai` | Automatically converts `pydantic-ai` agent calls into OpenTelemetry spans (`tracing.py`) so every LLM call shows up in Phoenix without manual instrumentation. |

### Dev-only dependencies

| Library | Why it's used |
| :---- | :---- |
| `pytest` / `pytest-asyncio` | Test runner. `pytest-asyncio` is required because the agents are `async def` (`await agent.run(...)`); `asyncio_mode = "auto"` lets async tests run without extra decorators. |
| `pydantic-evals` | Powers the `evals/` folder — structured evaluation of agent outputs, separate from pass/fail unit tests. |
| `tenacity` | Retry logic for transient failures/rate limits. |
| `black` / `isort` / `flake8` | Formatting and linting, enforcing consistent code style. |
| `uv` | Dependency/environment manager used to run `uv sync` and `uv run`. | 


### `pydantic-evals` vs `arize-phoenix`: why both?

At first glance both seem to be about "evaluating the agent," but they serve different, non-overlapping purposes. `arize-phoenix` is never even imported inside the `evals/` folder — it's used exclusively for live observability of the running app (see `tracing.py`, `app.py`).

| | `pydantic-evals` | `arize-phoenix` |
| :---- | :---- | :---- |
| **Role** | Offline test/scoring framework | Live tracing/observability UI |
| **Where used** | `evals/` folder only | `tracing.py`, `app.py` (the running app) |
| **What it needs** | A dataset of `Case`s with known *expected* outputs (e.g. `expected_pii=True`) | Nothing pre-defined — it just captures whatever spans happen at runtime |
| **How it judges correctness** | Two ways: (1) rule-based `Evaluator`s like `TextModerationCheck` that diff the agent's booleans against ground truth, and (2) `LLMJudge` — a separate LLM grading the rationale against a rubric | Doesn't judge correctness at all — no pass/fail, no rubric |
| **Output** | A pass/fail report (`report.print(...)`) run on demand via `python evals/text/test_cases.py` | A trace viewer at `localhost:6006` where you inspect prompts/responses/latency for each conversation |
| **When it runs** | On demand, as a batch job against curated test data | Continuously, every time someone chats with the app |

**Summary:** `pydantic-evals` answers *"is the model's output correct?"* against known-good test cases with automated grading. `arize-phoenix` answers *"what actually happened during this specific conversation?"* for debugging — it has no concept of "correct," it just records. Phoenix lets you notice a bad moderation call in a real trainee session; `pydantic-evals` is what turns that bad case into a permanent regression test with an expected answer, so it's caught automatically going forward.


### `pydantic-ai` vs `FastAPI`: why need both?

These two libraries work at different layers, and it helps to think of it like a restaurant:

- **`pydantic-ai` is the chef.** It's the code that actually does the work — it takes an input, sends it to Gemini (the AI model), and gets back a structured answer. This happens entirely *inside the kitchen* (in-process, plain Python function calls like `moderate_text()` in `agents/text_agent.py`).
- **FastAPI is the waiter / order counter.** It's how something *outside* the kitchen — a different program, a browser, another service — can ask the chef to do something, without having direct access to that kitchen. A client sends an HTTP request to a FastAPI endpoint (e.g. `/api/v1/moderate_text`), FastAPI hands it to the chef (`pydantic-ai`, which calls Gemini), and returns the result.

**Why this project needs both:**

In `app.py`, the Gradio frontend and the FastAPI backend are started as **two separate processes**:

```python
api_process = subprocess.Popen(["multimodal-moderation-api"])   # FastAPI backend
chat_process = subprocess.Popen(["multimodal-moderation-chat"]) # Gradio frontend