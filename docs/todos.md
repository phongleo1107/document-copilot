# Document Copilot — Build Checklist

Build order for the Driftwood Capital analyst pilot. This is a companion to
[client-brief.md](client-brief.md) (what the client wants) and
[architecture.md](architecture.md) (how it is meant to work). Conventions live in
[../AGENTS.md](../AGENTS.md), [../backend/AGENTS.md](../backend/AGENTS.md), and
[../frontend/AGENTS.md](../frontend/AGENTS.md).

## How to use this file

- Work top to bottom. A phase is done only when its **Verify** block passes.
- Tick a box only when the artefact exists *and* the verify step proves it.
- Phases are ordered by dependency, not by perceived difficulty.
- Legend: `[ ]` not started, `[~]` in progress, `[x]` done.

## Why backend first

The product's value and its risk both live in the backend.

1. The brief's trust contract — never invent, always cite, show the passage — is enforced server-side. The architecture is explicit: the browser stays thin, the backend is authoritative for retrieval, grounding, citation checks, and persistence.
2. The frontend consumes a contract that does not exist yet: `POST /chat/stream` emitting AI SDK message parts. Building the chat UI first means building against guesses.
3. Retrieval quality is the only genuinely unknown layer: chunking, citation granularity, hybrid ranking, RRF fusion. The UI is bounded, well-understood work.
4. Ingestion and retrieval need no auth, no LLM, and no UI, so the highest-value work is also the cheapest to start and to verify.

Within the backend the order is ingestion → retrieval → grounding/agent → API → frontend.
Do **not** start with the LLM: an agent wired to weak retrieval is a confident hallucination machine, and `backend/AGENTS.md` explicitly rejects mocking the LLM without also testing the grounding contract.

Backend-first is not backend-only-until-done. Land a walking skeleton early (stub streaming
endpoint at the start of Phase 5), then wire the real SPA against the real endpoint before tuning retrieval quality in anger.

## Critical path

```text
Phase 0  Foundations
   └─ Phase 1  Data model + migrations
        └─ Phase 2  Ingestion ................. offline, no LLM
             └─ Phase 3  Retrieval ........... highest risk, no LLM
                  └─ Phase 4  Grounding + agent ......... the trust contract
                       └─ Phase 5  API surface + streaming
                            └─ Phase 6  Frontend
                                 └─ Phase 7  Acceptance + deploy + pilot
```

## Phase 0 — Foundations

Goal: both services boot, config is fail-fast single-source, no product code yet.

Setup, once:

- [x] Create the Supabase project and collect URL / anon key / service_role key / direct DB
      URL — [guides/supabase-setup.md](guides/supabase-setup.md)
- [x] Create an OpenAI API key
- [x] `backend/.env` written from `backend/.env.example` (gitignored, never committed)
- [x] `frontend/.env` written from `frontend/.env.example` (public values only)

Backend:

- [x] `cd backend && uv sync`
- [x] `uv add fastapi uvicorn pydantic pydantic-settings httpx structlog openai supabase pydantic-ai sqlalchemy alembic "psycopg[binary]" pgvector`
- [x] `uv add --dev pytest ruff`
- [x] Add `[build-system]` + `[tool.hatch.build.targets.wheel]` so `app/` installs editable
      and `from app...` imports resolve everywhere — see [guides/backend-setup.md](guides/backend-setup.md)
- [ ] `app/config.py` — pydantic-settings, every var from `.env.example`, fails fast when a
      required value is missing (no silent fallbacks)
- [ ] `app/main.py` — FastAPI app, CORS from `ALLOWED_ORIGINS`, `GET /health`
- [ ] Confirm the local corpus is on disk: `data/downloads/{2021..2025}/` + `manifest.json`
      (25 10-Ks, AAPL/MSFT/NVDA/AMZN/GOOGL) — already downloaded
- [ ] Add `pytest` config and the `integration` marker so
      `pytest -m "not integration"` is the default fast suite

Frontend:

- [ ] `cd frontend && pnpm create vite . --template react-ts`
- [ ] `pnpm add react-router-dom @supabase/supabase-js ai @ai-sdk/react`
- [ ] `pnpm add -D tailwindcss @tailwindcss/vite` then `pnpm dlx shadcn@latest init`
- [ ] `src/lib/env.ts` validating `VITE_API_BASE_URL`, `VITE_SUPABASE_URL`,
      `VITE_SUPABASE_ANON_KEY` at boot
- [ ] Confirm `@/*` path alias works from both `tsconfig.json` and `vite.config.ts`

**Verify**

- [ ] `cd backend && uv run uvicorn app.main:app --reload` starts; `GET /health` returns 200
- [ ] `cd backend && uv run python -c "from app.config import settings"` succeeds
- [ ] Remove one required env var and confirm startup fails loudly (prove fail-fast, don't assume)
- [ ] `cd frontend && pnpm dev` renders; `pnpm tsc --noEmit` clean; `pnpm lint` clean
- [ ] No `package-lock.json` or `yarn.lock` exists (pnpm only)

**Watch-outs**

- `frontend/.npmrc` sets `minimum-release-age=10080` (7 days). If `ai` / `@ai-sdk/react`
  fail to resolve, that is the guard working, not a bug — pick a version older than 7 days
  rather than lowering the threshold.
- `ai` + `@ai-sdk/react` are new runtime deps. Per the dependency policy in
  `../AGENTS.md`, justify them in the commit message: they own the streaming wire format and
  `useChat` message state, which is not a <30-line hand-roll.

## Phase 1 — Data model and migrations

Goal: the schema exists in Supabase, owned by Alembic, with the Postgres features that
support hybrid retrieval.

- [ ] `app/database/models.py` — SQLAlchemy models for the six tables in
      [architecture.md](architecture.md#data-model):
  - [ ] `profiles` (keyed by `auth.users.id`)
  - [ ] `chat_threads` (owner, title, timestamps)
  - [ ] `chat_messages` (ordered user/assistant messages, AI SDK-compatible JSON where useful)
  - [ ] `message_citations` (normalized citation rows linked to assistant messages)
  - [ ] `source_documents` (filing metadata, source URL, normalized Markdown)
  - [ ] `document_chunks` (chunk text, metadata, embedding, `tsvector`, token count)
- [ ] `app/database/supabase.py` — user-scoped and admin client construction
- [ ] `cd backend && uv run alembic init alembic`
- [ ] `alembic/env.py` imports app metadata and reads the URL from `app.config.settings`
- [ ] First migration generated with `--autogenerate`, then **reviewed line by line**
- [ ] Explicit operations autogenerate cannot infer:
  - [ ] `create extension if not exists vector`
  - [ ] `vector(1536)` embedding column (matches `OPENAI_EMBEDDING_DIMENSIONS`)
  - [ ] generated `tsvector` column on `document_chunks`
  - [ ] HNSW index for the embedding column
  - [ ] GIN indexes for `tsvector` and the JSON metadata column
  - [ ] RLS enablement + policies (posture decided below)
- [ ] Confirm the migration uses the **direct/session** connection string, not the
      transaction pooler

**Verify**

- [ ] `cd backend && uv run alembic upgrade head` succeeds against the hosted project
- [ ] `uv run alembic current` reports head; `alembic downgrade -1 && upgrade head` round-trips
- [ ] Inspect the live schema: all six tables present, `vector` extension installed,
      HNSW and GIN indexes listed
- [ ] No table was created or altered by hand in the Supabase dashboard

## Phase 2 — Ingestion

Goal: the 25 downloaded 10-Ks become retrieval-ready chunks in Supabase, with citation-grade
metadata and no LLM involved.

- [ ] `backend/ingest/` — one-off scripts, not request-path code
- [ ] HTML → normalized Markdown extraction, preserving section structure and page or
      section anchors (granularity decided below)
- [ ] Chunker with metadata on every chunk: ticker, company, form, filing date, fiscal year,
      accession number, page or section, source offsets, token count
- [ ] Embeddings via the configured OpenAI model, batched, with retry on transient failures
- [ ] Writes to `source_documents` (one per filing) and `document_chunks`
- [ ] Idempotent re-runs — re-ingesting a filing must not duplicate documents or chunks
- [ ] `structlog` output for progress and counts, no `print`
- [ ] Unit tests next to the code (`tests/ingest/test_chunker.py`, etc.) with no network and
      no DB

**Verify**

- [ ] `pytest -m "not integration"` green — chunker, parser, and metadata tests
- [ ] Ingest the full corpus; chunk count per filing is sane and non-zero for all 25
- [ ] Spot-check one chunk in Supabase: text reads cleanly, metadata names the right
      company/year/page, embedding is 1536-dim
- [ ] Trace one citation by hand: filing → document row → chunk row → readable passage that
      actually appears in the source HTML
- [ ] Re-run ingestion and confirm row counts are unchanged (idempotency)

**Watch-outs**

- Page fidelity is load-bearing. [client-brief.md](client-brief.md) requires citing the specific page,
  and the analyst's one-click verification depends on the excerpt matching the source. If
  page anchors are wrong here, every citation downstream is wrong and no amount of
  retrieval tuning fixes it.

## Phase 3 — Retrieval (hybrid search)

Goal: a query string comes back as ranked, citation-ready source passages. No LLM anywhere in
this phase — that is the point.

- [ ] `app/retrieval/queries.py` — the two bounded SQL queries:
  - [ ] semantic search over `document_chunks.embedding` via pgvector
  - [ ] lexical search over `document_chunks.search_vector` via Postgres full-text search
- [ ] `app/retrieval/fusion.py` — Reciprocal Rank Fusion as a **pure function** over two
      ranked lists (no DB, no I/O)
- [ ] `app/retrieval/retriever.py` — query → embed → run both searches → fuse → fetch selected
      chunks plus neighbouring context → typed `SourcePassage` objects
- [ ] Passage metadata carries everything the UI needs to render a citation: company, ticker,
      form, filing date, fiscal year, page or section, excerpt, source URL
- [ ] Bounded result counts on both queries; no unbounded scans
- [ ] Unit tests: `tests/retrieval/test_fusion.py` (pure, exhaustive edge cases),
      `tests/retrieval/test_retriever.py` (mocked at the DB/service boundary)

**Verify**

- [ ] `pytest -m "not integration"` green, including fusion edge cases (empty lists, single
      item, disjoint lists, duplicate ids, ties)
- [ ] For each of the 10 analyst questions in
      [client-brief.md](client-brief.md#example-analyst-questions), the correct passage comes
      back in the top-k — checked at passage level, before any LLM is involved
- [ ] A cross-year question (e.g. Apple revenue mix 2021–2025) returns passages from
      *multiple* filings, not five chunks from one year
- [ ] A question with no support in the corpus (e.g. a company not in the corpus) returns
      low-relevance or empty results rather than plausible-looking noise

**Watch-outs**

- Run this phase before Phase 4 and resist wiring the agent early. Retrieval failures are
  invisible once an LLM is smoothing over them, and expensive to debug afterwards.
- Do not let the agent generate SQL. `architecture.md` is explicit: the agent gets bounded
  tools (`search_filings`, `read_chunk`, `read_surrounding_chunks`).

## Phase 4 — Grounding and the agent

Goal: the trust contract is enforced in code, not hoped for in a prompt.

- [ ] `app/assistant/outputs.py` — `GroundedAnswer`, `Citation`, `SourcePassage` as Pydantic
      models
- [ ] `app/assistant/deps.py` — `DocumentAgentDeps` dataclass (user id, thread id, retriever,
      grounding validator); no globals, per `backend/AGENTS.md`
- [ ] `app/assistant/instructions.md` — the product contract in plain language:
      answer only from retrieved passages; cite every factual claim; say the corpus does not
      contain enough evidence when it does not; no stock recommendations or investment
      advice; stay concise enough for analyst review
- [ ] `app/assistant/agent.py` — PydanticAI agent with typed deps, typed output, and the
      bounded retrieval tools
- [ ] `app/grounding/validator.py` — enforce the invariants:
  - [ ] every answer has at least one citation unless it explicitly reports insufficient
        evidence
  - [ ] every citation maps to a passage actually retrieved for *this* request
  - [ ] citations include enough metadata for the UI to render company, filing, date,
        page or section, and excerpt
  - [ ] a citation to a non-retrieved document is rejected, not silently dropped
  - [ ] formatting or grounding failure returns a controlled failure, never a polished
        unsupported answer
- [ ] Tests for citation extraction and grounding enforcement — required coverage per
      `backend/AGENTS.md`

**Verify**

- [ ] `pytest -m "not integration"` green on grounding and citation tests
- [ ] Injection test: a fixture where the model cites a chunk that was not retrieved is
      rejected by the validator
- [ ] Insufficient-evidence path: an out-of-corpus question produces an explicit "not enough
      evidence" answer with zero citations, not a guess
- [ ] The mock-the-LLM tests also assert the grounding contract, not just the happy path

**Watch-outs**

- Architecture supports a deliberate refusal. Question 10 in the brief ("do the filings prove
  generative AI improved margins?") is a *refusal* test, not a retrieval test. Treat a
  confident answer there as a bug.

## Phase 5 — API surface, auth, and streaming

Goal: an authenticated request produces a streamed, grounded answer in the exact wire format
`useChat` expects. Split into two steps on purpose: prove streaming with a stub, then plug the
real agent in.

### 5a — Walking skeleton (stub the answer)

- [ ] `app/main.py` mounts the routers; no business logic in route handlers
- [ ] `app/auth/dependencies.py` — verify `Authorization: Bearer <token>`, expose
      `get_current_user`; reject unauthenticated requests *before* any retrieval or LLM work
- [ ] Token verification by calling Supabase Auth's user endpoint (per `architecture.md`);
      keep it behind an `AuthService`-style boundary so local JWT validation can slot in later
- [ ] `app/chat/streaming.py` — emit AI SDK UI message stream parts
- [ ] `app/chat/messages.py` — convert the AI SDK wire format to internal Pydantic models and back
- [ ] `POST /chat/stream` returns a hardcoded answer + one fake citation, proving the contract
- [ ] `app/database/chats.py` — thread and message persistence, always scoped to the
      authenticated `user_id`

### 5b — Real turn orchestration

- [ ] `app/chat/orchestrator.py` — coordinates one full turn: auth → retrieve → agent →
      validate grounding → stream → persist
- [ ] `app/api/chat.py` — thread CRUD: create, list, fetch messages (user-scoped)
- [ ] Wiring: `POST /chat/stream` now runs the real agent and validates citations
- [ ] Citation metadata streamed as structured parts, so the UI can render sources as they arrive
- [ ] Typed error events for auth failure, missing thread, retrieval failure, and grounding
      failure
- [ ] Persist the user message, assistant message, cited chunks, and usage metadata only
      after the run completes successfully

**Streaming contract (AI SDK v5, verified against the docs)**

- [ ] Response is SSE: `data: {json}\n\n` frames, terminated by `data: [DONE]`
- [ ] Response header `x-vercel-ai-ui-message-stream: v1` is set — required for a custom
      (non-JS) backend
- [ ] Part sequence emitted: `start` (with `messageId`) → `start-step` →
      `text-start` / `text-delta` / `text-end` for answer text → `data-citation` parts →
      `finish-step` → `finish`
- [ ] Citations use data parts: `{"type":"data-citation","id":"<stable-id>","data":{...}}`;
      reusing an `id` reconciles that part rather than appending
- [ ] Status parts that should not land in history are marked `transient: true`
- [ ] `error` parts are used for failures, with the HTTP error classes from
      [architecture.md](architecture.md#error-handling)

**Verify**

- [ ] `curl` against a real Supabase access token returns a well-formed SSE stream ending in
      `[DONE]`, with the `x-vercel-ai-ui-message-stream: v1` header present
- [ ] Missing / expired / garbage token → 401 with no retrieval or LLM call made
- [ ] Another user's thread id → 403; nonexistent thread → 404; bad payload → 422
- [ ] Upstream LLM or Supabase failure surfaces as a typed error event, not a broken stream
- [ ] A successful turn leaves the expected rows in `chat_threads`, `chat_messages`, and
      `message_citations`
- [ ] Integration tests behind `@pytest.mark.integration` cover the streaming endpoint;
      `pytest -m "not integration"` still passes with no network

**Watch-outs**

- `architecture.md` warns the exact AI SDK API surface must be confirmed against the
  *installed* version. Treat 5a as a spike: build the stub, verify the part format against the
  real client, then build 5b on the confirmed format.
- Never persist a partial turn. Persist only after the run completes successfully, unless a
  deliberate partial-message model is introduced later.

## Phase 6 — Frontend

Goal: an analyst signs in and gets a cited answer they can verify in one click.

- [ ] `src/lib/env.ts` — validates the three `VITE_` vars at boot (no direct
      `import.meta.env` reads anywhere else)
- [ ] `src/lib/supabase.ts` — browser Supabase client
- [ ] `src/lib/http.ts` — thin `fetch` wrapper: base URL, JSON, bearer token injection,
      timeouts, typed `ApiError` with `isNetworkError` so CORS/network failures are
      distinguishable from HTTP failures
- [ ] `src/lib/api.ts` — product calls: list threads, create thread, fetch message history
- [ ] Auth: email sign-in / sign-out via Supabase (email only — no Google, no SSO)
- [ ] `src/App.tsx` — React Router with protected routes; unauthenticated users land on sign-in
- [ ] `src/pages/chat/*` — thread list, thread view, new-chat flow
- [ ] Chat wired to FastAPI with `useChat` + `DefaultChatTransport`:
      `api` pointing at `${VITE_API_BASE_URL}/chat/stream`, a `headers` callback supplying a
      fresh Supabase bearer token, and `prepareSendMessagesRequest` / `body` carrying `threadId`
- [ ] `src/components/chat/*`:
  - [ ] message list rendering `message.parts` (text parts and `data-citation` parts)
  - [ ] citation panel: company, form, filing date, page or section, excerpt, link to source
  - [ ] one-click expansion of the underlying passage next to the claim
  - [ ] streaming indicator driven by `status`
  - [ ] empty state (no threads yet) and per-thread empty state
  - [ ] error states distinguishing 401 (session expired → re-auth), network errors, and
        grounding refusals
- [ ] Insufficient-evidence answers render as a clear "not in the corpus" state, never as a
      normal answer

**Verify**

- [ ] Manual browser walkthrough: sign in → ask → streamed answer → open citation → passage
      matches the filing
- [ ] Follow-up question in the same thread works and history persists across a reload
- [ ] Session expiry mid-session produces the re-auth path, not a blank screen
- [ ] Empty and error states are each seen at least once in the browser
- [ ] `pnpm tsc --noEmit` clean; `pnpm lint` clean
- [ ] No test runner was added — correctness is manual + typed, per `frontend/AGENTS.md`

**Watch-outs**

- The bearer token must never be threaded through component props; the `api` client and the
  transport's `headers` callback own it.
- Keep the browser thin: no retrieval logic, no direct OpenAI calls, no service-role key, no
  privileged Supabase writes from the client.

## Phase 7 — Acceptance, deploy, and pilot readiness

Goal: satisfy the client's actual definition of done, not just "the app runs".

### 7a — Acceptance against the brief

- [ ] Run all 10 questions from
      [client-brief.md](client-brief.md#example-analyst-questions) end to end and record the
      result for each: answer, citations, verdict
- [ ] Every answer carries citations that resolve to real passages in the right filing and year
- [ ] Cross-year questions (1, 2, 3, 4, 5, 8) pull passages from multiple filings rather than
      answering from one
- [ ] Question 10 produces an explicit refusal to infer beyond the filings — this is the
      single most important pass/fail in the list
- [ ] Made-up-premise test: a question that assumes something false about the corpus corrects
      the premise or reports insufficient evidence instead of inventing support
- [ ] Out-of-corpus test: ask about a company not in the corpus and confirm a clear "not in
      the corpus" response with no citations
- [ ] No trading recommendations or stock picks, per the explicit out-of-scope list
- [ ] Record p50 / p95 end-to-end latency and cost per query; both must be defensible for a
      tool analysts use many times a day
- [ ] Write the results into a short evaluation note so the pilot team can compare against it

### 7b — Deploy

- [ ] Railway service 1: FastAPI backend, all backend env vars set, `DATABASE_URL` on the
      direct connection for migrations
- [ ] Railway service 2: Vite frontend build; `VITE_*` values set at build time
- [ ] `ALLOWED_ORIGINS` on the backend includes the deployed frontend origin (and the local
      dev origin, so local development keeps working)
- [ ] `alembic upgrade head` run against the production database
- [ ] Smoke test the deployed pair: sign in → ask → cited streamed answer
- [ ] Service-role key confirmed absent from the frontend bundle and from git history
- [ ] Fill in the "Running locally" section of [../README.md](../README.md) — it still says
      "to be added during the build"

### 7c — Pilot readiness

- [ ] Seed the pilot accounts (5 senior analysts + partners) with Driftwood email addresses
- [ ] Share the 10 curated example questions as a guided first-run path
- [ ] Instrument enough to answer "did this save 3 hours per analyst per week?" — at minimum
      per-user query counts, threads created, and citations opened
- [ ] Decide the feedback capture mechanism (in-app link, shared doc, weekly call) and tell the
      pilot group where to send it
- [ ] Named single point of contact for bugs during the pilot week

## Open decisions

Each of these blocks a phase. Settle it before starting that phase, and record the answer
here — an unsettled decision that leaks into code is expensive to undo.

- [ ] **Citation granularity: page vs section** (blocks Phase 2)
      iXBRL filing HTML has no true page breaks, but the brief promises "the specific page".
      Decide whether citations point at a derived page number, a section heading, or both, and
      how the UI shows it. This changes chunk metadata, the citation table, and the citation UI.
      Cheap to decide now, expensive to retrofit after ingestion.
- [ ] **RLS posture** (blocks Phase 1)
      `architecture.md` wants RLS enabled with policies; the backend also holds a service-role
      key. Choose: user-scoped clients with RLS actually enforced, or service-role everywhere
      with app-level `user_id` filtering. State which tables get RLS policies either way, and
      do not leave it half-done.
- [ ] **Pilot corpus scope** (blocks Phase 2)
      The brief says 10-Ks *and* 10-Qs for 2020–2025; `data/download.py` fetches 10-Ks only,
      5 per company (2021–2025). Confirm the 25 10-Ks already on disk are the pilot corpus, or
      widen the downloader and re-run.
- [ ] **`profiles` creation path** (blocks Phase 5)
      Supabase trigger on `auth.users` insert, or backend upsert on first authenticated
      request. Pick one and make it the only path.
- [ ] **Streaming approach** — decided: AI SDK `ai` + `@ai-sdk/react` with
      `useChat` + `DefaultChatTransport`, and FastAPI emitting AI SDK UI message stream parts.
      Justify the two new runtime deps in the commit message per the dependency policy.

## Parking lot (not needed for the pilot)

Keep out of the critical path. Do not build these speculatively.

- 10-Q ingestion and the wider S&P 500 corpus
- Local JWT validation instead of calling Supabase Auth per request
- A repeatable multi-turn evaluation harness with scored regression runs
- Thread naming / summarisation, search across threads, export
- Streaming citations mid-answer refinement, resume-stream-after-reconnect
- Object storage for raw filings
- Any move off the two-service Railway shape