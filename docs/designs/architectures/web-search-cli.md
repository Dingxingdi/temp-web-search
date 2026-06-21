## Architecture: Web Search CLI

### 1. Scope & Assumptions

#### In Scope

- A first-version Python CLI and foreground daemon.
- Three user-facing workflows: `keyword-search`, `llm-search`, and `url-fetch`.
- In-memory URL state with the public five-field URL object: `url`, `raw_content`, `content`, `abstract`, `available`.
- Keyword search and web fetch provider adapters based on the behavior in `useful_mcps`.
- OpenAI-compatible chat-completions LLM calls using direct HTTP requests.
- Provider-level concurrency control and per-URL fetch singleflight.

#### Out of Scope

- Persistent URL state across daemon restarts.
- Automatic daemon startup.
- Runtime LLM provider failover.
- External plugin loading for arbitrary third-party providers.
- Anthropic, Gemini, and OpenAI Responses adapters in the first version.
- Query-specific abstracts. `abstract` is a URL field and follows first-write semantics.

#### Assumptions

- The user starts the daemon manually with `web-search daemon`.
- The daemon is the only owner of mutable URL state.
- Network failures are treated as execution failures, not as URL semantics.
- `available=false` is terminal for the daemon lifetime.
- A URL without a non-empty abstract is not useful to the user and is discarded during search.

---

### 2. Architecture Summary

The CLI is a thin client that sends one newline-delimited JSON request to a foreground daemon over a Unix domain socket. The daemon owns config, provider clients, concurrency controls, and in-memory URL state. Search commands call enabled providers, normalize and merge URL records, validate provider-supplied body fields before writing them, and emit a jsonl file containing only `url` and `abstract`. `url-fetch` only accepts URLs already admitted by search, prepares stable `content` through web fetch providers and LLM cleaning when needed, safety-checks final content, and returns either full content or a request-local focus summary.

---

### 3. Design Decisions

#### Runtime Model

##### Foreground Daemon With Thin CLI

- Description: Users run `web-search daemon` in one terminal. Other commands connect to `~/.cache/web-search-cli/daemon.sock`.

- Rationale: This keeps CLI arguments small while preserving shared in-memory state across multiple CLI invocations.

- Trade-offs: The user must keep the daemon terminal open. If it stops, all in-memory URL state is lost.

- Rejected Alternatives:
  - Pure one-shot CLI:
    - Description: Every command starts and exits independently.
    - Why Rejected: `url-fetch <url>` would not know prior search state without adding persistent storage or extra path arguments.
  - Auto-start daemon:
    - Description: CLI starts the daemon when the socket is missing.
    - Why Rejected: First version would need stale socket handling, duplicate startup prevention, daemon log routing, and version checks.

##### Python First Version

- Description: Implement the first version with Python, `uv`, `argparse`, and `httpx`.

- Rationale: The source project is Python, provider behavior can be reused quickly, and the first version is still exploring product semantics.

- Trade-offs: Python distribution and long-running daemon ergonomics are weaker than a single Rust binary.

- Rejected Alternatives:
  - Rust first version:
    - Description: Build the CLI/daemon in Rust from the start.
    - Why Rejected: Better long-term packaging, but slower for this exploratory version, especially around async shared state and provider adapters.

#### Interface / Protocol

##### Single CLI Entry Point

- Description: Use subcommands under one executable: `web-search daemon`, `web-search keyword-search`, `web-search llm-search`, and `web-search url-fetch`.

- Rationale: This matches common CLI shape and avoids globally generic command names such as `keyword-search`.

- Trade-offs: Slightly longer commands than standalone tool names.

##### Newline-Delimited JSON Over Unix Socket

- Description: Each request and response is one compact JSON object followed by a newline.

- Rationale: Unix sockets provide byte streams, not object boundaries. A newline gives a simple and reliable message boundary while keeping the protocol easy to inspect.

- Trade-offs: This protocol is less general than HTTP and less rigorous than length-prefixed framing for arbitrary binary payloads.

- Rejected Alternatives:
  - HTTP over Unix socket:
    - Description: Use HTTP request/response framing on the local socket.
    - Why Rejected: More machinery than the first version needs.
  - Length-prefixed JSON:
    - Description: Prefix each JSON payload with its byte length.
    - Why Rejected: More exact, but harder to read and not necessary for one-request/one-response CLI calls.

##### Plain Text Command Output

- Description: Search commands print only the result jsonl path. `url-fetch` prints content or focus summary. Errors go to stderr with a non-zero exit code.

- Rationale: The successful output is easy to pipe into other tools and easy to read.

- Trade-offs: Structured error details are not available on stdout in the first version.

#### State Management

##### Five-Field URL Object

- Description: The URL object has exactly `url`, `raw_content`, `content`, `abstract`, and `available`.

- Rationale: The domain model stays small and matches the user's desired mental model.

- Trade-offs: The daemon does not cache metadata such as safety-check hashes, provenance, failure reasons, or per-query abstracts. This means safety can run again for repeated `url-fetch` calls.

##### First Non-Empty Wins

- Description: `abstract`, `raw_content`, and `content` are written only when empty. Existing non-empty values are never replaced.

- Rationale: This gives deterministic merge behavior across providers and prevents later provider output from silently changing a URL object.

- Trade-offs: A later provider may have a better abstract or cleaner content, but it will not replace the first accepted value.

- Rejected Alternatives:
  - Prefer richer provider output:
    - Description: Replace fields when a later provider returns more complete data.
    - Why Rejected: Adds provider ranking and replacement semantics that are not needed for the first version.

##### Search-Admitted URLs Only

- Description: `url-fetch` only accepts URLs that were kept by `keyword-search` or `llm-search`. Search hits without non-empty abstract are discarded and never enter URL state.

- Rationale: This reduces the risk of fetching hallucinated or low-signal URLs.

- Trade-offs: A real URL returned without an abstract cannot be fetched in the first version.

##### Repeat Searches Always Execute

- Description: Repeating the same normalized query calls providers again and creates a new jsonl file. Existing URL objects are reused by normalized URL, but query results are not cached.

- Rationale: Search results may change, while already accepted URL fields should remain stable.

- Trade-offs: Repeated searches consume provider quota even when the previous result would have been sufficient.

##### Unavailable URLs Remain Visible In Search Output

- Description: If a later search returns an existing URL whose `available` field is already `false`, the URL may still be written to the new jsonl result with its stored abstract.

- Rationale: Search output remains faithful to provider discoveries, while `url-fetch` continues to enforce the cached unavailable state.

- Trade-offs: A jsonl result may contain a URL that cannot be fetched.

#### Storage / Persistence

##### In-Memory URL State

- Description: The daemon stores URL objects in memory only.

- Rationale: The first version assumes no terminal restart and avoids database schema, migrations, cleanup policy, and disk privacy issues.

- Trade-offs: Daemon restart loses all search and fetch state.

##### Result Jsonl Files

- Description: Search results are written to `~/.cache/web-search-cli/results/` with simple unique names such as `keyword-8f3a2c.jsonl`.

- Rationale: Search output stays easy for humans and tools to inspect, while internal body state remains in daemon memory.

- Trade-offs: Jsonl files contain only `url` and `abstract`; they cannot restore full daemon state.

#### Provider Integration

##### Adapter Protocols And Registry

- Description: Core orchestration depends on `KeywordSearchProvider`, `WebFetchProvider`, and `LLMClient` interfaces. Provider-specific request and response mapping lives in adapters registered by name.

- Rationale: Keyword search providers and web fetch providers can change without changing orchestration logic.

- Trade-offs: First version still ships provider adapters in the repository; users cannot add a completely new provider by config alone.

- Rejected Alternatives:
  - Fully config-driven HTTP providers:
    - Description: Let users define arbitrary provider request/response mapping in TOML.
    - Why Rejected: Response parsing would become brittle and hard to validate.
  - External plugin loading:
    - Description: Load adapters from Python entry points or module paths.
    - Why Rejected: Good long-term extension point, but unnecessary for the first version.

##### Direct HTTP Instead Of LiteLLM Or SDKs

- Description: Web and LLM providers are called with explicit `httpx` requests.

- Rationale: The request shape is transparent, provider-specific coupling is confined to adapters, and there is no dependency on LiteLLM.

- Trade-offs: The project must maintain request/response parsing and retry behavior itself.

##### OpenAI-Compatible Chat Completions First

- Description: The first LLM adapter accepts a configured base URL and supports OpenAI-compatible `/v1/chat/completions`, with the adapter appending the endpoint path.

- Rationale: It covers many hosted and proxy LLM endpoints and is enough to implement judge, safety, content-clean, focus-summary, and LLM search.

- Trade-offs: Anthropic, Gemini, and OpenAI Responses need later adapters.

##### Configuration Inheritance For LLM Fallback

- Description: LLM fallback means resolving missing stage config from a broader config scope, not switching provider after runtime failure. LLM search inherits from global LLM config. Judge, safety, content-clean, and focus-summary inherit from shared fetch-LLM config and then global LLM config.

- Rationale: Configuration remains predictable and errors are visible.

- Trade-offs: If a configured LLM provider fails after retries, the command fails instead of automatically trying another provider.

##### Multiple Named LLM Providers

- Description: Users may configure multiple named LLM providers, each with its own protocol, base URL, API-key environment variable, and concurrency limit. `[llm_search].providers` selects all LLM providers called concurrently for LLM search.

- Rationale: Provider definitions stay separate from stage selection, so stages can change providers without changing adapter code.

- Trade-offs: Configuration validation must resolve provider references and reject missing or incompatible definitions.

##### First-Version Adapter Set

- Description: Ship keyword search adapters for Tavily, Firecrawl, Exa, Linkup, Brave, AnySearch, and TinyFish. Ship web fetch adapters for Tavily, Firecrawl, Exa, Linkup, and TinyFish.

- Rationale: This matches the useful provider coverage already present in `useful_mcps`.

- Trade-offs: More adapters must be tested and maintained in the first release.

##### Provider Stage Enablement

- Description: Each web provider config has independent `enable_search` and `enable_fetch` flags. Enabling an unsupported stage, such as Brave fetch, fails daemon startup.

- Rationale: Users can control provider participation without duplicating provider credentials or concurrency config.

- Trade-offs: Startup validation must understand each registered adapter's capabilities.

#### Concurrency / Scheduling

##### Provider Quotas

- Description: Each web provider has one concurrency quota shared by its search and fetch stages. Each LLM provider has a separate quota.

- Rationale: Providers often enforce account-level rate limits, and search/fetch for the same web provider should not exceed the same allowance.

- Trade-offs: A long fetch can reduce search capacity for the same web provider.

##### Per-URL Singleflight

- Description: Concurrent `url-fetch` requests for the same normalized URL are serialized across the complete workflow, including body preparation, safety, and optional focus summary. Different URLs may run concurrently.

- Rationale: This prevents duplicate web fetch provider calls, duplicate content cleaning, and conflicting safety mutations for the same URL.

- Trade-offs: Different focus requests for the same URL wait for each other even though their summaries are request-local.

##### Fetch Provider Scheduler

- Description: A URL fetch job is attempted by available web fetch providers one at a time. A provider candidate must pass cheap check and judge before fields are written. The first successful provider completes the job.

- Rationale: This uses provider capacity efficiently without firing all web fetch providers for the same URL at once.

- Trade-offs: The scheduler is more complex than a fixed provider order.

#### Security

##### Fetch-Gated URL Admission

- Description: `url-fetch` rejects URLs that were not admitted by search.

- Rationale: This reduces exposure to model-hallucinated URLs and keeps the fetch surface tied to provider-discovered results.

- Trade-offs: Users cannot fetch an arbitrary URL directly in the first version.

##### Safety On Final Content

- Description: `url-fetch` runs safety on final `content` before returning it or using it for focus summary.

- Rationale: The returned content is the data that matters to downstream users and agents.

- Trade-offs: Raw content may be passed through content-clean before safety. This follows the chosen first-version flow rather than doing a pre-clean safety pass.

##### Secret Handling

- Description: Config stores `api_key_env` names, not plaintext API keys. Enabled providers fail daemon startup if the referenced environment variable is missing.

- Rationale: Secrets stay out of config files and missing credentials are explicit.

- Trade-offs: Users must manage environment variables before starting the daemon.

#### Observability

##### Daemon-Terminal Logs

- Description: Progress and debug logs are printed by the foreground daemon. CLI requests do not stream progress in the first version.

- Rationale: Foreground logs keep implementation simple while still making provider activity visible.

- Trade-offs: A CLI command appears idle while waiting for a long daemon request.

#### Future Migration

##### Stable Protocol And Config Boundaries

- Description: Treat socket request/response JSON, TOML config shape, provider result contracts, and the five-field URL object as migration-stable.

- Rationale: A future Rust daemon can replace the Python daemon without changing user commands or behavior.

- Trade-offs: Early mistakes in these contracts are more costly to change later.

---

### 4. Component Catalog

| Component | Purpose | Key Responsibilities | Public Interfaces | Dependencies | Owns State? | Data-Flow Role |
|---|---|---|---|---|---|---|
| CLI Entrypoint | Give users one command surface | Parse subcommands, encode requests, connect to socket, render success/errors | `web-search <subcommand>` | `argparse`, socket protocol | No | Source / renderer |
| Socket Protocol | Define daemon boundary | Read/write newline-delimited JSON, validate envelope shape | `send_request`, `read_response` | JSON, Unix socket | No | Boundary |
| Foreground Daemon | Own runtime process | Create socket, load config, initialize providers, dispatch requests | NDJSON request handler | Config loader, orchestrators, stores | Yes, process lifecycle | Coordinator |
| Config Loader | Turn TOML and env into runtime config | Validate provider support, resolve API key envs, concurrency defaults, LLM stage inheritance | `load_config()` | TOML parser, environment | No | Validator / transformer |
| URL Store | Hold admitted URL objects | Normalize keys, merge first non-empty fields, track `available` | `get`, `admit`, `merge`, `mark_unavailable` | URL normalizer | Yes, URL objects | Store |
| Search Orchestrator | Implement keyword and LLM search workflows | Call providers, validate optional body fields, aggregate, write jsonl | `keyword_search(query)`, `llm_search(prompt)` | Provider registry, URL store, LLM stages, result writer | No | Coordinator |
| Fetch Orchestrator | Implement URL fetch workflow | Enforce admitted URL rule, prepare content, run safety, run focus summary | `url_fetch(url, focus=None)` | URL store, fetch scheduler, LLM stages | No | Coordinator |
| Fetch Scheduler | Efficiently fetch missing page bodies | Singleflight by URL, dispatch provider attempts, classify semantic vs execution results | `fetch_until_accepted(url)` | Fetch providers, provider quotas, cheap check, judge | Yes, in-flight jobs | Scheduler |
| Provider Quota Manager | Enforce provider max concurrency | Provide shared web-provider semaphores and separate LLM semaphores | `acquire(provider_name)` | Config | Yes, semaphores | Gate |
| Web Provider Adapters | Isolate provider APIs | Map provider-specific keyword search and web fetch endpoints into common hit/result objects | `search(query)`, `fetch(url)` | `httpx`, provider config | No | Adapter |
| LLM Adapter | Isolate OpenAI-compatible API | Build chat-completions POST body, merge `extra_body`, parse text/JSON | `complete_text`, `complete_json` | `httpx`, LLM config, quota manager | No | Adapter |
| LLM Stages | Provide prompt-level behavior | Judge crawl success, safety-check content, clean content, summarize focus, parse LLM search Markdown | stage functions | LLM adapter, prompts | No | Validator / transformer |
| Result Writer | Produce user-visible search files | Write jsonl lines with `url` and `abstract` only | `write_results(kind, records)` | File system | No | Sink |

Important ownership boundary: provider adapters must not mutate `URL Store` directly. They return normalized candidate objects; orchestrators decide what can be written.

---

### 5. Data Flow

#### 5.1 High-Level Flow

```mermaid
flowchart TD
    A[web-search CLI] --> B[NDJSON over Unix Socket]
    B --> C[Foreground Daemon]
    C --> D[Search Orchestrator]
    C --> E[Fetch Orchestrator]
    D --> F[Web Search Providers]
    D --> G[LLM Search Providers]
    E --> H[Fetch Scheduler]
    H --> I[Web Fetch Providers]
    D --> J[LLM Stages]
    E --> J
    J --> K[OpenAI-Compatible LLM Adapter]
    D --> L[URL Store]
    E --> L
    D --> M[Result Jsonl Writer]
    M --> N[~/.cache/web-search-cli/results]
    E --> O[CLI Text Output]
```

#### 5.2 Keyword Search Flow

1. CLI sends `{ "type": "keyword_search", "query": "..." }`.
2. Daemon normalizes non-empty query text and dispatches to the search orchestrator.
3. Enabled keyword search providers run under their provider quotas.
4. Each provider returns hits with `url`, optional `title`, optional `snippet`, and optional body fields.
5. The orchestrator derives `abstract = snippet or title`; if empty, the hit is discarded.
6. Optional `content/raw_content` is validated before writing:
   - prefer `content` when present;
   - otherwise validate `raw_content`;
   - if both are present and `content` passes, both may be written.
7. URL records are admitted or merged in the URL store with first non-empty wins.
8. The result writer writes one jsonl line per kept URL: `{ "url": "...", "abstract": "..." }`. An existing record with `available=false` is still written when returned by the current search.
9. Daemon returns the result path; CLI prints it.

#### 5.3 LLM Search Flow

1. CLI sends `{ "type": "llm_search", "prompt": "..." }`.
2. The search orchestrator calls all providers listed in `[llm_search].providers`.
3. Each provider uses the OpenAI-compatible LLM adapter and may receive stage `extra_body`.
4. The model is prompted to return restricted Markdown blocks:

   ```markdown
   ## Result

   URL: https://example.com/page

   Abstract:
   Page summary.
   ```

5. The parser reads only `## Result`, `URL:`, and `Abstract:`. Arbitrary Markdown links are ignored.
6. Parsed records go through the same URL admission, merge, and jsonl writing path as keyword search.

#### 5.4 URL Fetch Flow

1. CLI sends `{ "type": "url_fetch", "url": "...", "focus": null }`.
2. Daemon normalizes the URL and checks that it exists in the URL store. A missing URL is an error, not a semantic unavailable response.
3. If `available=false`, the daemon returns the standard unavailable message.
4. If `content` exists, the daemon runs safety on `content`.
5. If `content` is empty and `raw_content` exists, the daemon runs content-clean and writes `content`.
6. If both body fields are empty, the fetch orchestrator creates or joins a singleflight fetch job.
7. The fetch scheduler dispatches one provider attempt at a time as provider quota is available.
8. A provider candidate must pass cheap check and judge before body fields are written.
9. After `content` exists, safety runs on final content.
10. Without focus, content is returned. With focus, focus-summary runs and its result is returned without caching.

#### 5.5 Failure Flow

- Invalid input: The daemon rejects empty queries, malformed URLs, unknown request types, and URLs not admitted by search with an error response.
- Provider timeout or malformed response: This is an execution failure for that provider pipeline. Other keyword search providers or web fetch providers may continue.
- Search provider partial failure: If at least one provider pipeline completes, the command can produce jsonl. If all fail by execution error, the command errors.
- Body validation rejection: Cheap check or judge rejection is semantic failure. In search, the URL can still be kept with abstract only. In fetch, if no provider succeeds and at least one semantic failure occurred, mark `available=false`.
- Safety rejection: Mark `available=false` and return unavailable.
- Safety/content-clean/focus execution failure: Return an error and do not mark `available=false`.
- Concurrent fetch for same URL: Later requests join the existing singleflight job and then read the resulting URL state.
- Missing daemon: CLI prints an instruction to start `web-search daemon` and exits non-zero.

---

### 6. Interfaces & Contracts

#### Request / Response Contract

Public, migration-stable socket requests:

```json
{"type":"keyword_search","query":"python async"}
```

```json
{"type":"llm_search","prompt":"find recent docs about asyncio cancellation"}
```

```json
{"type":"url_fetch","url":"https://example.com/page","focus":null}
```

```json
{"type":"url_fetch","url":"https://example.com/page","focus":"pricing details"}
```

Public, migration-stable socket responses:

```json
{"ok":true,"text":"/home/user/.cache/web-search-cli/results/keyword-8f3a2c.jsonl"}
```

```json
{"ok":true,"text":"returned content or focus summary"}
```

```json
{"ok":false,"error":"daemon_not_ready","message":"Start the daemon with: web-search daemon"}
```

#### Domain Object Contract

Public URL object shape:

```json
{
  "url": "https://example.com/page",
  "raw_content": "",
  "content": "",
  "abstract": "Short search-derived summary.",
  "available": true
}
```

Rules:

- `available` starts as `true`.
- `available=false` is terminal for the daemon lifetime.
- `raw_content`, `content`, and `abstract` use first non-empty wins.
- `focus` summaries are not URL object fields and are not cached.

#### Provider Contracts

Internal `KeywordSearchProvider` result:

```json
{
  "url": "https://example.com/page",
  "title": "Optional title",
  "snippet": "Optional snippet",
  "raw_content": "",
  "content": ""
}
```

Internal `WebFetchProvider` result:

```json
{
  "raw_content": "Raw page text or markup",
  "content": "Optional provider-cleaned content"
}
```

`raw_content` is required and non-empty for a successful `WebFetchProvider` result. `content` is optional. If both are returned, `content` is the validation candidate; when it passes, both fields may be written.

Internal LLM stage contract:

```json
{
  "provider": "openai_main",
  "model": "gpt-4o-mini",
  "extra_body": {}
}
```

Provider-specific fields must stay inside adapter config or `extra_body`; they must not leak into URL objects.

#### Config Contract

The first-version config is stored at `~/.config/web-search-cli/config.toml`:

```toml
[web_providers]
default_max_concurrency = 3

[web_providers.tavily]
enable_search = true
enable_fetch = true
api_url = "https://api.tavily.com"
api_key_env = "TAVILY_API_KEY"
max_concurrency = 5

[llm_providers]
default_max_concurrency = 2

[llm_providers.openai_main]
protocol = "openai"
api_style = "chat_completions"
api_url = "https://api.openai.com"
api_key_env = "OPENAI_API_KEY"
max_concurrency = 2

[llm]
provider = "openai_main"
model = "gpt-4o-mini"

[llm_search]
providers = ["openai_main"]
model = "gpt-4o-search-preview"
extra_body = { web_search_options = { search_context_size = "high" } }

[fetch_llm]
provider = "openai_main"
model = "gpt-4o-mini"

[fetch_llm.judge]
extra_body = {}

[fetch_llm.safety]
extra_body = {}

[fetch_llm.content_clean]
extra_body = {}

[fetch_llm.focus_summary]
extra_body = {}
```

Rules:

- API keys are read only from the environment variable named by `api_key_env`.
- An enabled provider with a missing API-key environment variable fails daemon startup.
- Web-provider `max_concurrency` falls back to `[web_providers].default_max_concurrency`.
- LLM-provider `max_concurrency` falls back to `[llm_providers].default_max_concurrency`.
- Every LLM stage may override provider, model, protocol options, and `extra_body`.
- LLM search resolves stage settings from `[llm_search]` and then `[llm]`.
- Judge, safety, content-clean, and focus-summary resolve stage settings from their stage table, then `[fetch_llm]`, then `[llm]`.

#### Result Jsonl Contract

Public search output file:

```json
{"url":"https://example.com/page","abstract":"Short search-derived summary."}
```

Only records with a non-empty abstract are written.

---

### 7. Open Question

- If `available=false` is cached and a later keyword search provider returns the same URL with new body fields, should the daemon skip all body validation, validate without changing state, or support manual recovery? The current architecture treats `available=false` as terminal, but this edge case should be reviewed with the full state machine.
