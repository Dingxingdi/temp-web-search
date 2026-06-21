## Error Handling: Web Search CLI

### 1. Error Model

The system separates errors into three categories:

| Category | Meaning | User-visible result | State mutation |
|---|---|---|---|
| Input error | CLI request is invalid or cannot be served by current daemon state | stderr + non-zero exit | No URL field changes |
| Execution failure | A dependency, protocol, parser, or LLM stage failed to execute correctly after retries | stderr + non-zero exit, unless isolated to one provider pipeline | Usually no URL field changes |
| Semantic rejection | A dependency executed successfully and produced a meaningful negative judgment | Successful unavailable response or provider-level rejection | May set `available=false` or skip body-field writes |

Primary rule:

- Execution failure means "the system could not complete the operation."
- Semantic rejection means "the URL/content was judged unsuitable."
- Only semantic rejection may make a URL unavailable.

---

### 2. CLI And Socket Errors

#### Missing Daemon For Workflow Commands

- Applies to: `keyword-search`, `llm-search`, `url-fetch`.
- Condition: CLI cannot connect to `~/.cache/web-search-cli/daemon.sock`.
- Handling: Print an instruction to start `web-search daemon` to stderr and exit non-zero.
- State: No state changes.

#### Missing Daemon For Stop

- Applies to: `stop`.
- Condition: CLI cannot connect to `~/.cache/web-search-cli/daemon.sock`.
- Handling: Treat as success. Print `Daemon is not running.` to stdout and exit zero.
- State: No state changes.

#### Malformed Socket Response

- Condition: CLI receives invalid JSON, no newline-delimited response, or a response missing required fields.
- Handling: Print a protocol error to stderr and exit non-zero.
- State: No state changes.

#### Daemon Request Decode Failure

- Condition: Daemon receives invalid newline-delimited JSON or an unknown request type.
- Handling: Return `{ "ok": false, "error": "bad_request", "message": "..." }`.
- State: No state changes.

#### Daemon Is Shutting Down

- Condition: Daemon has accepted a shutdown request and receives a new workflow request before the socket is removed.
- Handling: Return `{ "ok": false, "error": "daemon_shutting_down", "message": "Daemon is shutting down" }`; CLI writes the message to stderr and exits non-zero.
- State: No URL field changes.

#### Repeated Stop During Shutdown

- Condition: Daemon has accepted a shutdown request and receives another `shutdown` request before the socket is removed.
- Handling: Treat as success. Return a concise shutdown-in-progress response.
- State: No URL field changes.

---

### 3. Input Validation Errors

#### Search Query Is Empty

- Applies to: `keyword-search`, `llm-search`.
- Handling: Reject request as input error.
- State: No state changes.

#### URL Is Invalid

- Applies to: `url-fetch`.
- Rule: URL must normalize to `http://` or `https://`; host is lowercased; path/query/fragment are preserved.
- Handling: Reject request as input error.
- State: No state changes.

#### URL Was Not Search-Admitted

- Applies to: `url-fetch`.
- Condition: Normalized URL is not in daemon URL state because it was never emitted in a search jsonl result.
- Handling: Return `url_not_admitted` as an input/state error; CLI writes the message to stderr and exits non-zero.
- State: No state changes.

#### URL Is Already Unavailable

- Applies to: `url-fetch`.
- Condition: URL object exists and `available=false`.
- Handling: Return the standard unavailable message.
- State: No state changes.

---

### 4. Provider Pipeline Errors

Provider pipelines are isolated units. One provider failing should not automatically fail the whole command if other provider pipelines can still produce a valid result.

#### Keyword Search Provider Pipeline

Pipeline:

```text
keyword provider HTTP call
-> provider response parsing
-> URL/abstract extraction
-> optional body validation through cheap_check + judge
```

Handling:

- HTTP timeout, transport error, retry exhaustion, malformed provider response, or judge execution failure marks this provider pipeline as failed.
- Other keyword search providers continue.
- If at least one keyword search provider pipeline completes, the command writes jsonl, even if the result set is empty.
- If all keyword search provider pipelines fail by execution failure, the command errors.

Semantic body rejection:

- `cheap_check` rejection or judge `ok=false` only prevents `raw_content/content` from being written.
- The URL can still be kept if it has a non-empty abstract.

#### LLM Search Provider Pipeline

Pipeline:

```text
LLM chat-completions call
-> restricted Markdown parsing
-> URL/abstract extraction
```

Handling:

- LLM execution failure, invalid response shape, or unparseable restricted Markdown marks this LLM search provider pipeline as failed.
- Other LLM search providers continue.
- If at least one LLM search provider pipeline completes, the command writes jsonl, even if no results were parsed.
- If all configured LLM search provider pipelines fail, the command errors.

#### URL Fetch Provider Pipeline

Pipeline:

```text
URL fetch provider HTTP call
-> provider response parsing
-> candidate selection
-> cheap_check
-> judge
```

Handling:

- HTTP timeout, transport error, retry exhaustion, malformed provider response, empty execution result, or judge execution failure marks this URL fetch provider attempt as execution failure.
- `cheap_check` rejection or judge `ok=false` marks this provider attempt as semantic failure.
- The fetch scheduler can try another URL fetch provider after either kind of failed attempt.

Final no-success rule:

- Pure execution failure across all attempted providers: `url-fetch` errors and does not mark `available=false`.
- At least one semantic failure and no successful provider: mark `available=false` and return the unavailable message.
- First successful provider candidate writes fields and completes the fetch job.

---

### 5. LLM Stage Errors

All external HTTP and LLM calls use configurable retry with exponential backoff. Retryable transport/status errors and invalid transient LLM responses are retried. Retry exhaustion is an execution failure unless explicitly listed as semantic rejection.

#### Judge

- Execution failure: Treat as provider pipeline failure, not URL semantic failure.
- Semantic rejection: `ok=false`.
- Search behavior: Do not write provider-supplied body fields; URL may still be kept with abstract.
- Fetch behavior: Count as semantic failure for that provider attempt only when judge returns `ok=false`; try other providers if available.

#### Safety

- Execution failure: `url-fetch` errors and does not mark `available=false`.
- Semantic rejection: `ok=false`; mark `available=false` and return unavailable.
- Safety runs on final `content` before returning content or before focus summary.

#### Content Clean

- Execution failure: `url-fetch` errors and does not mark `available=false`.
- Invalid output after retries: Treat as execution failure.
- Successful output: Write `content` only if `content` is still empty.

#### Focus Summary

- Execution failure: `url-fetch` errors and does not return full content as fallback.
- Successful output: Return summary text only; do not cache it.

#### LLM Search

- Execution failure: Fails only that LLM search provider pipeline.
- Semantic no-results: A successfully parsed response with zero results is not an error.

---

### 6. URL State Mutation Rules

#### Allowed Mutations

- Create URL object with `available=true` only when a search result has non-empty abstract.
- Fill `abstract` only if empty.
- Fill `raw_content` only if empty and candidate body validation passed.
- Fill `content` only if empty and candidate body validation or content-clean succeeded.
- Set `available=false` only for semantic rejection:
  - safety `ok=false`;
  - no URL fetch provider succeeds and at least one provider attempt semantically rejected the candidate.

#### Forbidden Mutations

- Do not replace non-empty `abstract`, `raw_content`, or `content`.
- Do not mark `available=false` for pure execution failure.
- Do not create URL objects for search hits with empty abstract.
- Do not cache focus summaries.
- Do not let provider adapters mutate URL state directly.

---

### 7. User-Facing Output

#### Success

- `web-search keyword-search ...`: stdout contains only the jsonl path.
- `web-search llm-search ...`: stdout contains only the jsonl path.
- `web-search url-fetch URL`: stdout contains content or unavailable message.
- `web-search url-fetch URL FOCUS`: stdout contains focus summary or unavailable message.
- `web-search stop`: stdout contains `Daemon stopped.` when it stopped a running daemon, or `Daemon is not running.` when no daemon was running.

#### Error

- stderr contains a concise human-readable error.
- Exit code is non-zero.
- stdout should be empty on errors.

#### Unavailable Message

Use one stable user-facing message for unavailable URLs:

```text
URL unavailable: it may have failed content validation or been flagged as unsafe.
```

This message is a successful command response only when a known URL is semantically unavailable. A URL that was never admitted by search is an error instead.

---

### 8. Logging

- Daemon logs provider execution failures, semantic rejections, retries, and successful provider completions to the daemon terminal.
- CLI commands do not stream logs in the first version.
- Logs may include provider name, stage, normalized URL or query, and short reason.
- Logs must not include API keys or full request headers.
- Full page content should not be logged by default.

---

### 9. Error Codes

Socket responses should use stable internal error codes even if CLI renders human text:

| Code | Meaning |
|---|---|
| `bad_request` | Invalid JSON request, unknown type, or malformed fields |
| `empty_query` | Search query or prompt normalized to empty |
| `invalid_url` | URL could not be normalized to HTTP/HTTPS |
| `url_not_admitted` | URL was not emitted by prior search |
| `no_keyword_search_providers` | No enabled keyword search providers |
| `no_llm_search_providers` | No configured LLM search providers |
| `no_url_fetch_providers` | No enabled URL fetch providers and no stored body |
| `all_providers_failed` | All relevant provider pipelines failed by execution error |
| `llm_stage_failed` | A required LLM stage failed after retries |
| `protocol_error` | Socket or provider protocol response was malformed |
| `config_error` | Daemon config is invalid |
| `daemon_shutting_down` | Daemon is rejecting new workflow requests because shutdown has started |

Unavailable semantic responses should not use an error code in successful responses unless the CLI later gains structured output.
