## Testing: Web Search CLI

### 1. Test Strategy

The first version should treat tests as executable confirmation of the design, not as provider-integration smoke tests. Most tests should use fake `KeywordSearchProvider`, fake `WebFetchProvider`, fake LLM clients, and in-memory daemon state.

Primary goals:

- Protect the five-field URL object semantics.
- Prove provider failures are isolated correctly.
- Prove semantic rejection and execution failure produce different state changes.
- Prove concurrency controls prevent duplicate URL work.
- Prove config inheritance and provider capability validation.
- Prove the CLI/daemon socket protocol is stable enough for a future Rust daemon.

---

### 2. Test Layers

| Layer | Purpose | Uses real network? | Examples |
|---|---|---|---|
| Unit tests | Validate pure functions and small components | No | URL normalization, Markdown parsing, config resolution |
| Orchestrator tests | Validate search/fetch state transitions | No | Duplicate URL merge, `available=false`, body validation |
| Scheduler tests | Validate concurrency and provider worker behavior | No | Same URL singleflight, different URL concurrency |
| Protocol tests | Validate CLI/daemon JSON contract | No | NDJSON framing, response rendering |
| Adapter parser tests | Validate provider response mapping | No | Tavily/Exa/Firecrawl sample JSON parsing |
| Optional integration tests | Manual or opt-in real provider checks | Yes, opt-in only | Real Tavily search, real OpenAI-compatible LLM call |

Real provider tests should be skipped by default and enabled only with explicit environment variables.

---

### 3. Unit Test Coverage

#### URL Normalization

Cover:

- Trims leading/trailing whitespace.
- Accepts only `http://` and `https://`.
- Lowercases host.
- Preserves path, query, and fragment.
- Rejects empty string, `ftp://`, `mailto:`, and malformed URLs.

#### URL Object Merge Rules

Cover:

- `abstract`, `raw_content`, and `content` use first non-empty wins.
- Empty fields can be filled later.
- Non-empty fields are never replaced.
- `available=false` is terminal for `url-fetch`.
- Focus summary is never written to the URL object.

#### Restricted Markdown Parser For LLM Search

Cover:

- Parses repeated `## Result` blocks.
- Extracts only `URL:` and `Abstract:`.
- Ignores arbitrary Markdown links outside result blocks.
- Drops results with empty abstract.
- Fails the LLM search provider pipeline on malformed required fields.

#### Config Resolution

Cover:

- `api_key_env` is required for enabled providers.
- Missing API key environment variables fail daemon config validation.
- Web-provider concurrency defaults and provider overrides resolve correctly.
- LLM-provider concurrency defaults and provider overrides resolve correctly.
- LLM search resolves from `[llm_search]` then `[llm]`.
- Judge/safety/content-clean/focus-summary resolve from stage table, then `[fetch_llm]`, then `[llm]`.
- `extra_body` is allowed on every LLM stage and passed through without core interpretation.
- Enabling an unsupported provider stage fails startup validation.

---

### 4. Search Orchestrator Tests

#### Keyword Search

Cover:

- Empty query returns `empty_query`.
- No enabled keyword search provider returns `no_keyword_search_providers`.
- Each search execution creates a new jsonl path; normalized query cache is not reused.
- Providers are called under their quota manager.
- Partial provider execution failure still writes results from successful provider pipelines.
- All provider pipelines failing by execution error fails the command.
- Search hits with empty abstract are discarded and do not enter URL state.
- `abstract = snippet or title`.
- Duplicate normalized URLs merge with first non-empty wins.
- Existing `available=false` URL can still appear in a later jsonl result.
- Provider-supplied `content` is cheap-checked and judged before writing.
- Provider-supplied `raw_content` is cheap-checked and judged before writing when no `content` exists.
- If both `content` and `raw_content` exist and `content` passes validation, both may be written.
- Cheap-check or judge semantic rejection prevents body-field writes but may keep URL/abstract.
- Judge execution failure fails only that provider pipeline.

#### LLM Search

Cover:

- No configured LLM search providers returns `no_llm_search_providers`.
- All configured `[llm_search].providers` are called.
- Partial LLM search provider execution failure still writes results from successful pipelines.
- A successful provider that parses zero results still counts as completed.
- All LLM search provider pipelines failing produces `all_providers_failed`.
- Parsed results fill `url` and `abstract` only.

---

### 5. URL Fetch Tests

#### Admission And Basic State

Cover:

- Invalid URL returns `invalid_url`.
- URL not emitted by search returns `url_not_admitted` and exits as error at CLI layer.
- `available=false` returns the stable unavailable message without provider or LLM calls.
- Existing `content` skips web fetch and content-clean, but still runs safety before returning.
- Existing `raw_content` with empty `content` runs content-clean, writes `content`, then safety.
- Focus request runs safety first, then focus-summary, and does not cache focus output.

#### Web Fetch Provider Attempts

Cover:

- A successful provider writes fields only after cheap_check and judge pass.
- If provider returns both `content` and `raw_content`, validate `content`; if it passes, write both.
- If provider `content` fails validation, do not retry that provider's `raw_content` fallback.
- Pure execution failure across all providers errors and does not set `available=false`.
- Any semantic failure and no success sets `available=false`.
- A later provider can succeed after earlier execution or semantic failures.
- `available=false` prevents future provider attempts.

#### LLM Stages

Cover:

- Safety `ok=false` sets `available=false` and returns unavailable.
- Safety execution failure errors and does not set `available=false`.
- Content-clean execution failure errors and does not set `available=false`.
- Focus-summary execution failure errors and does not return full content as fallback.
- Retry exhaustion is classified as execution failure.

---

### 6. Scheduler And Concurrency Tests

#### Per-URL Singleflight

Cover:

- Two concurrent `url-fetch` calls for the same URL trigger one web fetch provider attempt.
- Waiting callers receive the same resulting content or unavailable/error result.
- Same-URL requests serialize safety/content-clean/focus workflow.
- Different URLs can run concurrently.

#### Provider Quotas

Cover:

- A web provider's keyword search and web fetch share one semaphore.
- LLM providers use separate semaphores from web providers.
- Provider `max_concurrency` is respected under concurrent requests.
- Scheduler never runs the same URL through two web fetch providers at the same time.
- When one provider is saturated, another available provider can process a different URL job.

Use controllable fake providers with async events to prove ordering without sleeping.

---

### 7. Socket And CLI Tests

#### NDJSON Framing

Cover:

- One request object followed by `\n` is decoded as one request.
- Multiple requests arriving in one byte read are split correctly.
- Partial request bytes are buffered until newline.
- Invalid JSON produces `bad_request`.
- Response objects are encoded as one line.

#### CLI Rendering

Cover:

- Successful search prints only jsonl path to stdout.
- Successful `url-fetch` prints only content, summary, or unavailable message to stdout.
- Error responses print message to stderr, stdout is empty, exit code is non-zero.
- Missing daemon prints instruction to run `web-search daemon`.

---

### 8. Adapter Tests

For each shipped `KeywordSearchProvider` adapter, test response parsing with representative sample payloads:

- Tavily
- Firecrawl
- Exa
- Linkup
- Brave
- AnySearch
- TinyFish

For each shipped `WebFetchProvider` adapter, test response parsing with representative sample payloads:

- Tavily
- Firecrawl
- Exa
- Linkup
- TinyFish

Adapter tests should verify:

- URL extraction.
- Title/snippet extraction for keyword search.
- `raw_content/content` mapping.
- Empty or malformed provider responses return empty provider results or provider execution failure according to adapter contract.
- Adapter never mutates URL state.

---

### 9. Fixture Design

Use these fake components:

- `FakeKeywordSearchProvider`: returns configured hits or raises configured exception.
- `FakeWebFetchProvider`: returns configured fetch result, semantic candidate, or raises configured exception.
- `FakeLLMClient`: records stage calls and returns configured text/JSON.
- `ControlledProvider`: blocks on async events to test semaphore and scheduler ordering.
- Temporary cache/config directories: isolate `~/.cache` and `~/.config` behavior.

Avoid wall-clock sleeps. Use events, semaphores, and explicit task coordination.

---

### 10. Opt-In Integration Tests

Integration tests may be added but must be skipped unless explicit variables are set, such as:

```text
WEB_SEARCH_RUN_INTEGRATION=1
TAVILY_API_KEY=...
OPENAI_API_KEY=...
```

They should verify only basic connectivity and response shape. They should not be required for normal CI or local development.

---

### 11. Acceptance Criteria

Before implementation is considered complete:

- Unit and orchestrator tests pass without network access.
- Provider adapter parser tests cover every first-version adapter.
- Scheduler tests prove same-URL singleflight and provider quota behavior.
- CLI/protocol tests prove stdout/stderr/exit-code behavior.
- Error handling tests cover execution failure vs semantic rejection.
- Search commands create jsonl files with only `url` and `abstract`.
- No test relies on real provider credentials unless explicitly opt-in.
