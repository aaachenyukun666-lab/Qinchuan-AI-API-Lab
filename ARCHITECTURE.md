# Architecture

This document describes how AI Relay Meter is put together: what each module is
responsible for, how data moves through the system, the shapes of the core data
structures, the internationalization mechanism, the offline constraints that
shape the codebase, the error taxonomy, and the extension points for adding
providers.

For user-facing documentation, see [../README.md](../README.md) and
[../README.zh-CN.md](../README.zh-CN.md).

---

## Table of Contents

- [Design Principles](#design-principles)
- [High-Level Structure](#high-level-structure)
- [Module Responsibilities](#module-responsibilities)
- [Data Flow](#data-flow)
- [Core Data Structures](#core-data-structures)
- [Internationalization Mechanism](#internationalization-mechanism)
- [Offline / `file://` Constraints](#offline--file-constraints)
- [Error Classification](#error-classification)
- [Extension Points](#extension-points)
- [Testing Strategy](#testing-strategy)

---

## Design Principles

Five constraints drive nearly every structural decision in this codebase.

1. **No bundler, no framework.** `src/js/` is loaded directly by the browser.
   There is no transpilation step and no build artifact between source and
   runtime. What you read is what executes.
2. **One codebase, two environments.** The same source files run in the browser
   (via `<script src>`) and under `node --test` (via `require`). No duplicate
   test build.
3. **Offline-first.** The app must work when loaded from `file://` inside an
   Android WebView, with no network access beyond the configured API endpoint.
4. **Never fabricate data.** Absent values are represented as absent. A missing
   `usage` object is recorded as "no usage returned", never as an estimate.
5. **Pure logic is isolated.** All computation lives in `src/js/core/`, which has
   no DOM and no network dependency, and is therefore directly unit-testable.

## High-Level Structure

```text
                       ┌──────────────────────────┐
                       │        index.html        │
                       │  loads CSS, locales, JS  │
                       └────────────┬─────────────┘
                                    │
                       ┌────────────▼─────────────┐
                       │        app.js            │  UI assembly,
                       │  (flow orchestration)    │  event wiring
                       └──┬────────┬────────┬─────┘
                          │        │        │
              ┌───────────▼──┐ ┌───▼─────┐ ┌▼──────────┐
              │  engine.js   │ │verify.js│ │ report.js │
              │ usage test   │ │ verif.  │ │ build/exp │
              └───────┬──────┘ └────┬────┘ └─────┬─────┘
                      │             │            │
              ┌───────▼─────────────▼────────────▼─────┐
              │               api.js                   │
              │  OpenAI-compatible client (SSE, HTTP)  │
              └───────────────────┬────────────────────┘
                                  │
              ┌───────────────────▼────────────────────┐
              │           src/js/core/*.js             │
              │  usage · diff · errors · mask · stats  │
              │        (pure, DOM-free, testable)      │
              └────────────────────────────────────────┘

              i18n.js ── locales/*.json ── locales/*.js (generated)
              store.js ── localStorage
```

The dependency direction is strictly one-way. `core/` depends on nothing.
`api.js` depends only on `core/`. `engine.js`, `verify.js`, and `report.js`
depend on `api.js` and `core/`. `app.js` sits on top and owns all DOM access.

## Module Responsibilities

### Module loading convention

Every file under `src/js/` is a **UMD-style classic script**, not an ES module.
Each file attaches itself to the global `window.ARM` namespace while remaining
`require()`-able from Node:

```js
(function (root, factory) {
  if (typeof module === 'object' && module.exports) { module.exports = factory(); }
  else { root.ARM = root.ARM || {}; root.ARM.usage = factory(); }
})(typeof globalThis !== 'undefined' ? globalThis : this, function () {
  'use strict';
  /* ... */
  return { normalizeUsage: normalizeUsage };
});
```

The namespace assignments are:

| File | Namespace |
| --- | --- |
| `src/js/core/usage.js` | `ARM.usage` |
| `src/js/core/diff.js` | `ARM.diff` |
| `src/js/core/errors.js` | `ARM.errors` |
| `src/js/core/mask.js` | `ARM.mask` |
| `src/js/core/stats.js` | `ARM.stats` |
| `src/js/api.js` | `ARM.api` |
| `src/js/engine.js` | `ARM.engine` |
| `src/js/verify.js` | `ARM.verify` |
| `src/js/report.js` | `ARM.report` |
| `src/js/i18n.js` | `ARM.i18n` |
| `src/js/store.js` | `ARM.store` |

Consequences:

- `package.json` does **not** set `"type": "module"`. The package stays
  CommonJS by default, so `tests/` can `require()` the exact files the browser
  loads.
- Script load order in `index.html` matters. `core/` files must be included
  before the modules that consume them, and all of them before `app.js`.
- There are no `import` / `export` statements anywhere in `src/js/`.

The one deliberate exception is `scripts/`, whose files are `.mjs` and use ESM.
Those are Node-only tooling and never reach the browser.

### `index.html`

The single-page application entry. It declares the DOM skeleton, links
`src/css/style.css`, loads `locales/*.js` followed by `src/js/**/*.js` in
dependency order, and contains no application logic. This is also the
`启动页面` (start page) referenced by `appConfig.xlt` for Android packaging.

### `src/js/app.js`

UI assembly and flow orchestration. Owns every DOM reference and every event
listener. Responsibilities:

- Read and write form inputs; render live results, verification output, and the
  report view.
- Validate user input before a run starts.
- Instantiate a test run, delegate execution to `engine.js` or `verify.js`,
  and stream progress callbacks into the DOM.
- Wire the start and stop controls, including aborting in-flight requests.
- Trigger report building and export via `report.js`.
- Coordinate language switching through `i18n.js` and re-render affected text
  without touching test state.

`app.js` contains no token arithmetic and no HTTP logic. Anything computational
belongs in `core/`.

### `src/js/api.js`

The OpenAI-compatible client. Responsibilities:

- Build request URLs from the configured `Base URL` (normalizing trailing
  slashes) and append `/chat/completions` or `/models`.
- Attach the `Authorization: Bearer <key>` header and `Content-Type`.
- Issue Chat Completions requests in both streaming (SSE) and non-streaming
  modes.
- Parse the SSE frame stream and extract the final `usage` object when the
  provider emits one.
- Enforce per-request timeouts and support `AbortController` cancellation.
- Surface raw HTTP status codes and response bodies to the caller, without
  swallowing them.
- Fetch the model list and report the specific failure mode on error.

`api.js` performs no token computation. It returns raw responses; `core/usage.js`
normalizes them.

### `src/js/engine.js`

The Token Usage Test engine. Responsibilities:

- Accept a validated test configuration (request count, `max_tokens`,
  concurrency, model, prompt).
- Schedule requests according to the concurrency limit, never exceeding it.
- Record per-request results, including latency measured around the request.
- Emit progress callbacks as each request settles so the UI can update live.
- Honour cancellation promptly and retain partial results.
- Classify failures per request via `core/errors.js` rather than aborting the
  whole run.

The engine is bounded by **request count**, not by a token target. This is a
deliberate constraint: it guarantees a predictable upper bound on cost.

### `src/js/verify.js`

Small-scale Verification and discrepancy analysis. Responsibilities:

- Run the same request engine with conservative defaults (3 requests,
  `max_tokens` 100, concurrency 1).
- Capture the account balance/quota reading before and after the run when the
  provider exposes such an endpoint, and accept manually entered values when it
  does not.
- Feed the observations into `core/diff.js` to compute token-level and
  cost-level differences.
- Aggregate repeated iterations through `core/stats.js`.
- Select the neutral outcome statement based on data completeness and observed
  differences, and record which fields could not be obtained.

`verify.js` never asserts that a provider is misreporting. It selects among
three neutral statements and attaches the raw evidence.

### `src/js/report.js`

Report construction and export. Responsibilities:

- Assemble the report object from run records and summary statistics.
- Apply key masking to every string that reaches the report.
- Serialize to JSON.
- Serialize to CSV: one row per request plus a summary row.
- Render the report in the active language.

### `src/js/i18n.js`

Translation lookup, locale detection, and runtime language switching. See
[Internationalization Mechanism](#internationalization-mechanism).

### `src/js/store.js`

Configuration persistence over `localStorage`, under three versioned keys:

| Key | Contents |
| --- | --- |
| `arm.settings.v1` | Base URL, model, and test parameters |
| `arm.secret.v1` | The API key, stored separately from general settings |
| `arm.lastReport.v1` | The most recent report, so a reload does not lose it |

Responsibilities:

- Persist and restore configuration and the last report.
- **Keep the API key in its own key** (`arm.secret.v1`), separate from general
  settings, so that clearing or rewriting settings cannot accidentally leak the
  key into a non-secret blob, and so `clear()` can remove it deliberately.
- **Keep the language selection out of this module entirely.** It lives under
  `arm.locale`, owned by `i18n.js`, so switching language can never clear
  configuration, run history, token data, or results.
- **Probe `localStorage` once at load time.** A write/read/remove probe
  determines availability up front; if storage is unusable, the module falls
  back to in-memory state rather than throwing. Every read and write is also
  wrapped defensively, because restricted WebView origins can fail at any time.
- Expose `KEYS` so callers and tests refer to the same key names.

### `src/js/core/usage.js`

Usage normalization. Converts a raw provider `usage` object into the internal
canonical shape, preserving absent fields as `null` rather than `0` or an
estimate. Handles the common variations in how providers nest `details` and
`cached_tokens`, and records *why* usage is absent via a `reason` field:
`missing` (no usage object), `not_an_object` (present but not an object),
`no_token_fields` (object with no recognized counter), or `ok`.

Exports: `normalizeUsage`, `accountableTokens`, `describeUsage`, `emptyUsage`.
`accountableTokens` selects the figure used for accounting — `total_tokens` when
the API reported it, otherwise the sum of `prompt_tokens` and
`completion_tokens` when both are present. That fallback is the only derivation
the project performs, and it is flagged on the record as `usageDerived`.

### `src/js/core/diff.js`

Token and cost difference calculation. Computes the signed difference, the
absolute difference, and the discrepancy rate, and keeps token-level and
cost-level results in separate fields. Guards against division by zero when the
baseline is zero, returning a `null` rate rather than infinity.

Exports: `computeTokenDifference`, `computeCostDifference`, `computeAccountUsage`,
`classify`, `analyze`, plus the `STATUS` constants and
`DEFAULT_TOLERANCE_PERCENT` (1.0%).

`computeAccountUsage(before, after, unit)` turns a balance delta into an account
usage figure and records whether the unit is `currency` or `tokens`, so a report
can never silently compare CNY against tokens. If the balance rose rather than
fell, the signed delta is reported honestly instead of being forced positive.

`classify(diff, tolerancePercent)` maps a computed difference onto one of three
statuses — `normal`, `discrepancy`, or `insufficient` — and returns the matching
`conclusionKey`. `analyze()` is the one-call entry point the UI uses.

### `src/js/core/errors.js`

Error classification. Maps HTTP status codes, network failures, timeouts, and
payload problems onto a stable set of error codes, each with a corresponding
`errors.*` i18n key and `fatal` / `retryable` flags. `shouldAbortRun()` uses the
`fatal` flag to decide whether a failure should end the whole run. See
[Error Classification](#error-classification).

### `src/js/core/mask.js`

API key masking. Produces `sk-1234************abcd`-style output, retaining only
a short prefix and suffix. Exports `maskKey`, plus `scrubText` and
`safeHeaders` for scrubbing free-form log text and header objects. Applied at
every logging and reporting boundary so a key cannot leak through a code path
that forgets to mask.

### `src/js/core/stats.js`

Repeat-test aggregation. Exports `describe` (count, sum, mean, median, min,
max, standard deviation, range), `summarizeRequests` (per-run request and token
statistics), `summarizeRounds` (across repeats), `weightedDiscrepancyRate`,
`countBy`, and `classifyStability`.

The purpose of repetition is to show whether a difference is *stable*, so the
module reports the spread and deliberately does not convert it into a final
accusation. `describe` returns `null` — not `0` — for statistics that cannot be
computed, and `stdev` is `null` for fewer than two samples. Contains no I/O.

## Data Flow

The end-to-end path from configuration to exported report:

```text
 1. Configuration
    User enters Base URL / API Key / Model (and parameters).
    store.js persists them to localStorage.
             │
             ▼
 2. Request construction
    app.js validates input and builds a test configuration.
    api.js composes the endpoint URL and headers.
             │
             ▼
 3. Execution
    engine.js schedules requests under the concurrency limit.
    Each request is a real HTTP call (streaming or not).
             │
             ▼
 4. Raw response
    api.js returns HTTP status, body or SSE frames, and measured latency.
    Nothing is interpreted at this stage.
             │
             ▼
 5. Usage normalization
    core/usage.js maps the raw usage object onto the canonical shape.
    Missing fields stay null and are flagged as unknown.
             │
             ▼
 6. Account observation  (Small-scale Verification only)
    Balance/quota is read before and after the run, or entered manually.
             │
             ▼
 7. Difference calculation
    core/diff.js computes Difference and Discrepancy Rate,
    keeping token and cost figures separate.
             │
             ▼
 8. Aggregation  (repeat runs)
    core/stats.js produces means and min/max across iterations.
             │
             ▼
 9. Outcome selection
    verify.js chooses one of three neutral statements based on
    data completeness and the observed difference.
             │
             ▼
10. Report
    report.js assembles the report, masks keys, renders it,
    and exports JSON or CSV.
```

A key property: **errors do not short-circuit the pipeline**. A failed request
becomes a record with an error classification and null usage fields, and the
run continues. This is what prevents a single upstream failure from blanking the
page.

## Core Data Structures

These are the canonical shapes. Field names are stable; the JSON export is
derived from them.

### Request record

One entry per API request issued.

Records are flat — the token fields sit directly on the record rather than
inside a nested `usage` object, because the table and CSV views render one row
per record.

```js
{
  index: 1,                      // 1-based position within the run
  startedAt: "2025-01-01T00:00:00.000Z",  // ISO 8601, when the request was sent
  model: "example-model",
  requestId: "",                 // provider request id; '' when not returned
  status: 200,                   // HTTP status; 0 when the request never landed
  latencyMs: 842,                // measured round-trip time in ms; null if unmeasured
  ok: true,                      // HTTP 2xx and a parseable body
  errorCode: null,               // e.g. 'RATE_LIMITED'; see Error Classification
  errorMessage: "",              // provider-supplied message, truncated

  promptTokens: 21,              // null when the API did not report it
  completionTokens: 100,
  totalTokens: 121,
  cachedTokens: null,
  reasoningTokens: null,
  usageSource: "api",            // 'api' | 'derived' | 'unavailable'
  usageDerived: false,           // true when totalTokens was summed from parts
  hasUsage: true,                // false => API returned no usage object at all

  textPreview: ""                // short excerpt of the completion, for diagnosis
}
```

Notes:

- `hasUsage` is the flag that distinguishes "the API said zero" from "the API
  said nothing". Only when it is `false` does the UI show
  `API 未返回 Token Usage` / `No Token Usage returned by the API`.
- `usageDerived` is what keeps derived figures honest. When a provider returns
  only `prompt_tokens` and `completion_tokens`, `core/usage.js` sums them for
  `totalTokens` and sets `usageDerived: true`, so the report can say the total
  was derived rather than reported. Nothing is ever estimated beyond that sum,
  and `usageSource` records where the number came from.
- Unknown fields are `null`, never `0`. A `0` is a real measurement.
- `latencyMs` is measured locally around the request and is an observation of the
  client, not a value reported by the provider.
- The API key is never placed on a record.

### Test run

One entry per run started by the user.

A run started by the user. The shape differs by mode: `engine.run()` returns a
flat run result, while `verify.runRepeated()` returns a result containing
`rounds`.

```js
// Returned by engine.run() — Token Usage Test
{
  ok: true,
  summary: {
    mode: "usage",               // or "verification"
    baseUrl: "https://api.example.com/v1",
    model: "example-model",
    requests: 3,                 // planned request count
    maxTokens: 100,
    concurrency: 1,
    stream: false,
    prompt: "...",               // the exact prompt text sent
    startedAt: "2025-01-01T00:00:00.000Z",
    endedAt: "2025-01-01T00:00:05.000Z",
    durationMs: 5000,
    completedRequests: 3,
    stoppedEarly: false,
    stopReason: null,            // 'stopped' when the user aborted
    fatalCode: null,             // error code that ended the run early
    requestStats: { /* see Summary statistics */ },
    totalTokens: 363,
    dataComplete: true           // see the note below
  },
  records: [ /* Request record, ... */ ]
}
```

```js
// Returned by verify.runRepeated() — Small-scale Verification
{
  ok: true,
  repeat: 5,                     // clamped to 1..20
  rounds: [
    {
      ok: true,
      config: { baseUrl, model, requests, maxTokens, concurrency, stream },
      summary: { /* engine summary for this round */ },
      records: [ /* Request record, ... */ ],
      analysis: {                // output of core/diff.js analyze()
        token:  { status, apiTokens, accountTokens, difference,
                  absoluteDifference, discrepancyRate, reason },
        cost:   { status, theoreticalCost, actualCost, difference,
                  absoluteDifference, discrepancyRate, reason },
        status: "normal" | "discrepancy" | "insufficient",
        conclusionKey: "result.normal" | "result.discrepancy" | "result.insufficient",
        tolerancePercent: 1.0,
        dataCompleteness: { apiTokens, accountTokens, theoreticalCost,
                            actualCost, unit }
      },
      balance: {
        before: 12.3456,         // null when unavailable
        after: 12.3400,
        deltaAvailable: true,
        unit: "currency"         // 'currency' | 'tokens'
      }
    }
  ],
  roundRecords: [ /* one analysis-bearing record per round, for stats */ ],
  summary: { /* aggregated summary */ },
  stats: { /* core/stats.js summarizeRounds() output */ },
  overallStatus: "normal" | "discrepancy" | "insufficient",
  conclusionKey: "result.normal" | "result.discrepancy" | "result.insufficient"
}
```

Notes:

- **`dataComplete`** is `true` only when every planned request finished **and**
  the API reported usage for all of them, with no early stop. A run that
  succeeded but returned no usage is *not* complete, and is reported as
  incomplete rather than quietly averaged.
- **Manual balance values apply to one round only.** When `repeat > 1`, a
  manually entered before/after pair is cleared for subsequent rounds, because
  one manually observed delta describes a single run and cannot be reused as if
  it were measured per round.
- **The overall verdict follows the weakest complete layer.** If any complete
  layer reports a discrepancy, the overall status is `discrepancy`; if every
  layer is `insufficient`, so is the overall status.
- The API key is never copied onto a run or round object. Reports are built
  from these structures, so this keeps keys out of exports by construction, in
  addition to the explicit masking pass.

### Summary statistics

Produced by `core/stats.js` for a run, and used as the per-round input when
aggregating repeats. Numeric fields are grouped by metric, each carrying the
full descriptive set from `describe()`.

```js
// stats.summarizeRequests(records)
{
  totalRequests: 3,
  successCount: 3,
  failureCount: 0,
  requestsWithUsage: 3,          // API actually reported usage
  requestsWithoutUsage: 0,
  errorCodeCounts: {},           // { RATE_LIMITED: 1, ... }

  totalTokens:      { count: 3, sum: 363, mean: 121, median: 121,
                      min: 118, max: 124, stdev: 2.5, range: 6 },
  promptTokens:     { /* same shape */ },
  completionTokens: { /* same shape */ },
  cachedTokens:     { /* same shape */ },
  reasoningTokens:  { /* same shape */ },
  latencyMs:        { /* same shape */ }
}

// stats.summarizeRounds(roundRecords)
{
  rounds: 5,
  token:  { meanApiTokens, meanAccountTokens, meanDifference,
            minDifference, maxDifference, /* ... */ },
  rate:   { mean, median, min, max, stdev },
  cost:   { /* same shape as token */ },
  stability: "stable" | "variable" | "insufficient"
}

// stats.weightedDiscrepancyRate(roundRecords)
0.55   // sum(|account - api|) / sum(api) * 100, or null when unusable
```

`describe()` returns `null` for `sum` / `mean` / `min` / `max` / `stdev` when
there are no samples, and `stdev` is `null` for fewer than two samples rather
than reporting a meaningless zero.

Every numeric field that could not be derived is `null`. The UI renders `null`
as an explicit "unknown" marker with the corresponding localized label, never as
zero.

`weightedDiscrepancyRate` weights each round by its API token count, so a round
that consumed more tokens contributes proportionally more to the aggregate rate.
It returns `null` when no round has a usable positive API token count.

### Report object

The report is a serialization wrapper built by `report.build(session, options)`.
It branches on mode: a plain run produces a usage report, a repeated run
produces a verification report.

```js
{
  schema: "ai-relay-meter/report",
  reportVersion: "1.0",          // format version, for future migrations
  generatedAt: "2025-01-01T00:00:10.000Z",
  locale: "zh-CN",               // language the report was rendered in
  mode: "usage",                 // or "verification"
  tool: { name: "AI Relay Meter", version: "0.1.0" },

  configuration: {
    baseUrl: "https://api.example.com/v1",
    model: "example-model",
    requests: 3,
    maxTokens: 100,
    concurrency: 1,
    stream: false,
    prompt: "...",
    apiKey: "[not included]"     // literal placeholder, by construction
  },

  timing: { startedAt, endedAt, durationMs },

  totals: {
    totalRequests, successCount, failureCount,
    requestsWithUsage, requestsWithoutUsage,
    promptTokens, completionTokens, totalTokens,
    cachedTokens, reasoningTokens
  },

  statistics: { totalTokens, latencyMs, promptTokens, completionTokens },
  errors: { /* error code -> count */ },

  completeness: {
    complete: true,
    stoppedEarly: false,
    stopReason: null,
    fatalCode: null,
    usageReportedForAllRequests: true,
    notes: [ /* human-readable statements of what could not be measured */ ]
  },

  requests: [ /* flattened per-request rows, as exported */ ],

  // mode === "verification" only:
  verification: {
    repeat: 5,
    rounds: [ /* per-round analysis */ ],
    stats: { /* aggregated statistics */ },
    overallStatus: "normal" | "discrepancy" | "insufficient",
    conclusionKey: "result.normal" | "result.discrepancy" | "result.insufficient",
    tolerancePercent: 1.0
  }
}
```

Notes:

- **`apiKey` is the literal string `[not included]`.** The report does not carry
  a key, a key fragment, or a hash of one. Combined with the masking pass, a
  report is safe to attach to a public issue.
- **`reportVersion` is `"1.0"`** and `schema` is a fixed identifier, so v1.0 can
  introduce a migration path for previously exported reports without silently
  changing their meaning.
- **`tolerancePercent` defaults to 1.0%.** A discrepancy rate at or below the
  tolerance is classified as `normal`. Exact zero is not the bar, because API
  accounting is integer-based and relays round differently — treating every
  rounding difference as a discrepancy would produce false positives.
- **`conclusionKey` is an i18n key, not a sentence.** The report stores the key
  and renders the sentence in `locale`, so the same report object can be
  re-rendered in either language without changing the data.

## Internationalization Mechanism

Two locales, one codebase. There are no parallel forks per language.

### Resource layout

```text
locales/zh-CN.json    # canonical source of truth
locales/en-US.json
locales/zh-CN.js      # generated: window.ARM_LOCALES = { "zh-CN": {...} }
locales/en-US.js      # generated
```

`zh-CN.json` is canonical. A new string is added there first, mirrored into
`en-US.json`, and then both are regenerated into `.js` bundles by
`scripts/build-locales.mjs`.

### Why `.json` and `.js` both exist

The JSON files are the authoring format: readable, diffable, and easy to
validate. The generated `.js` files exist purely for offline loading — see
[Offline / `file://` Constraints](#offline--file-constraints).

### Runtime lookup

`src/js/i18n.js` exposes:

- **`detectLocale(navLang, stored)`** — a stored preference wins; otherwise the
  candidate list from `navigator.languages` / `navigator.language` is consulted.
  A tag beginning with `zh` selects `zh-CN`; anything unsupported falls back to
  `FALLBACK` (`en-US`).
- **`t(key, params)`** — resolves a dot-delimited key against the active bundle,
  with `{placeholder}` interpolation via `interpolate()`. A missing key falls
  back to `FALLBACK` and then to the key itself, so a missing translation is
  visible rather than blank.
- **`setLocale(tag, options)`** — normalizes the tag, swaps the active bundle,
  persists it to `localStorage` under `arm.locale` (unless
  `options.persist === false`), and notifies subscribers. Persistence is
  best-effort: a storage failure leaves the language working, just not
  remembered.
- **`registerLocale(tag, bundle)`** — installs a bundle. `init()` calls this for
  each entry in `window.ARM_LOCALES`.
- **`subscribe(fn)` / `applyTo(root)`** — re-render hooks used by `app.js` to
  update the DOM after a language change.
- **`SUPPORTED`** — `['zh-CN', 'en-US']`; **`STORAGE_KEY`** — `'arm.locale'`.

### Rules the implementation must uphold

- **Switching language must not clear anything.** The language is stored under
  `arm.locale`, a different key from the settings and secret keys used by
  `store.js`. `setLocale` only swaps the active bundle and writes that one key,
  so configuration, run records, token data, and results are untouched.
- **Switching during a running test is allowed.** Test data lives in the run
  result objects and the DOM is re-rendered from that state, so an in-flight run
  continues and its numbers are unchanged. Only presentation changes.
- **No hard-coded user-visible strings.** All UI text, button labels, error
  messages, help text, settings, and report headings come from the bundles.
- **Technical terms stay in English.** API, Token, OpenAI, DeepSeek, OpenRouter,
  JSON, HTTP, HTTPS, REST, and SSE are not translated, by convention.

## Offline / `file://` Constraints

The Android build loads the page from `file://` inside a WebView. This single
fact determines two structural decisions.

### Constraint 1 — classic scripts, not ES modules

When a page is loaded from `file://`, the WebView treats it as an opaque origin
and applies CORS rules to module fetches. A `<script type="module">` triggers a
cross-origin module load that is blocked, so the application would fail to start.

A classic `<script src="...">` has no such restriction and loads normally from
the local filesystem.

Therefore:

- Every file in `src/js/` is a **UMD-style classic script**, wrapped in the
  `(function (root, factory) { ... })` pattern shown earlier.
- Scripts are included with plain `<script src="...">` tags, in dependency
  order.
- `package.json` does **not** set `"type": "module"`.
- No `import` or `export` appears in `src/js/`.

The UMD wrapper is what makes this workable without a bundler. The same file
that assigns `root.ARM.usage = factory()` in the browser also satisfies
`module.exports = factory()` under Node. Tests therefore `require()` the
identical source the browser executes — there is no separate build, and no
possibility of the tested code diverging from the shipped code.

### Constraint 2 — generated locale bundles

The same origin restriction applies to data fetches. A `fetch("locales/zh-CN.json")`
from a `file://` page is a cross-origin request and is blocked.

A classic `<script src>` is not blocked, so the locale data is delivered as
executable JavaScript that assigns a global:

```js
// locales/zh-CN.js — generated, do not edit by hand
window.ARM_LOCALES = window.ARM_LOCALES || {};
window.ARM_LOCALES["zh-CN"] = { /* ... */ };
```

`index.html` loads the locale `.js` files before `i18n.js`, so the bundles are
present as globals by the time the app initializes. Over HTTP the same files
work identically.

`scripts/build-locales.mjs` performs the JSON → JS conversion. It is an `.mjs`
file and uses ESM, because it runs only under Node and never reaches the
browser.

**Practical consequence:** `npm run build:locales` must be run after cloning and
after any change to `locales/*.json`. If the generated `.js` files are missing,
the app still starts but shows raw translation keys instead of text.

### Other offline considerations

- **No external assets.** No CDN links, web fonts, or third-party scripts. The
  icon ships in `assets/`.
- **`localStorage` availability.** The `file://` origin can have restricted
  storage in some WebViews. `store.js` treats persistence as best-effort and
  falls back to in-memory state.
- **Network reachability.** Requests go to the user-configured `Base URL` from
  the `file://` origin. Providers that reject browser or opaque origins will
  fail; WebCatX's `webcat.network.request` is the native, CORS-free alternative
  and is the intended extension point for that case.

## Error Classification

Errors are classified by `src/js/core/errors.js` into a stable set of codes.
Each code maps to an i18n key under `errors.*`, so the UI renders a localized
sentence rather than a raw exception. Classification is per request, so one
failure never aborts a run unless the condition is genuinely fatal.

Codes, as defined in the `CODES` map:

| Code | Trigger | i18n key | Fatal | Retryable |
| --- | --- | --- | --- | --- |
| `OK` | Request succeeded | `errors.ok` | no | — |
| `INVALID_KEY` | HTTP 401 | `errors.invalidKey` | **yes** | no |
| `FORBIDDEN` | HTTP 403 | `errors.forbidden` | **yes** | no |
| `NOT_FOUND` | HTTP 404 — usually a wrong model name or path | `errors.notFound` | **yes** | no |
| `BAD_REQUEST` | HTTP 400 | `errors.badRequest` | **yes** | no |
| `INSUFFICIENT_QUOTA` | HTTP 402 — account out of credit | `errors.insufficientQuota` | **yes** | no |
| `RATE_LIMITED` | HTTP 429 | `errors.rateLimited` | no | **yes** |
| `SERVER_ERROR` | HTTP 5xx (other than 502) | `errors.serverError` | no | **yes** |
| `BAD_GATEWAY` | HTTP 502 | `errors.badGateway` | no | **yes** |
| `TIMEOUT` | HTTP 408 / 504, or a client-side timeout | `errors.timeout` | no | **yes** |
| `ABORTED` | The run was stopped by the user | `errors.aborted` | **yes** | no |
| `NETWORK` | DNS or connection failure | `errors.network` | no | **yes** |
| `CORS` | `Failed to fetch` / `NetworkError` — browser-opaque failures | `errors.cors` | no | **yes** |
| `BAD_JSON` | The response body did not parse as JSON | `errors.badJson` | no | no |
| `MODELS_UNSUPPORTED` | `GET /models` returned 404/405 or an empty list | `errors.modelsUnsupported` | no | no |
| `NO_USAGE` | HTTP 200 with no `usage` object | `errors.noUsage` | no | no |
| `UNKNOWN` | Anything unclassified | `errors.unknown` | no | **yes** |

Two design decisions are worth calling out.

**Fatal codes stop the run.** A bad key, an unknown model, or a missing
permission will not fix itself by trying again, so `shouldAbortRun()` ends the
run instead of burning through the remaining request budget on calls that cannot
succeed. Rate limits and 5xx responses are retryable and do **not** stop the run.

**CORS failures are reported as `CORS`, not `NETWORK`.** Browsers deliberately
give almost nothing away for cross-origin failures, so a bare `Failed to fetch`
is ambiguous. It is classified separately and the UI surfaces CORS as the likely
cause, because that distinction changes what the user should do next — the
native bridge path rather than a network fix.

Two invariants hold for all error codes:

1. **No error blanks the page.** Every failure renders as a message in the UI
   and, where a request was issued, as a record in the run.
2. **No error is hidden.** HTTP status codes and provider messages are surfaced
   to the caller and shown in truncated form, rather than being replaced by a
   generic failure message.

## Extension Points

### Adding a provider

The architecture treats "provider" as configuration, not as a code branch. A new
OpenAI-compatible provider normally requires **no code change at all** — the user
supplies a `Base URL`, key, and model.

When a provider deviates from the protocol, extend in this order:

1. **Response normalization — `src/js/core/usage.js`.**
   If the provider nests usage differently, or names cached/reasoning fields
   differently, add a mapping there. This is the correct place for shape
   differences, and it keeps the divergence in one pure, testable function.

2. **Capability probing — `src/js/api.js`.**
   If the provider exposes optional endpoints (a balance or quota route), add a
   probe that attempts the call and reports unavailability cleanly. The
   verification flow must continue when the probe fails; a missing endpoint is
   the normal case, not an error condition.

3. **Error mapping — `src/js/core/errors.js`.**
   If the provider uses non-standard status codes or error envelopes, map them
   onto the existing kinds rather than adding a parallel error path.

4. **Request shape — `src/js/api.js`.**
   If a provider requires extra parameters or a different body shape, add them
   as an opt-in configuration field. Do not change the default request shape.

5. **Pricing rules — `src/js/core/diff.js`.**
   Provider-specific pricing (input/output tiers, cache pricing, reasoning token
   pricing, multipliers, minimum billable units) belongs in the cost layer, kept
   separate from the token layer. Record theoretical and actual cost as distinct
   fields.

Guidelines that apply throughout:

- **Never hard-code a provider as the only option.** The default configuration
  ships with an empty `Base URL`.
- **Keep new logic in `core/` where possible**, so it is unit-testable without
  a network.
- **Preserve the UMD wrapper** in any new `src/js/` file, and add its namespace
  assignment to the table in
  [Module Responsibilities](#module-responsibilities).
- **Preserve the neutrality rule.** A new provider integration must not
  introduce wording that characterizes a platform as fraudulent.

### Other extension points

| Goal | Where to extend |
| --- | --- |
| New test mode | Add a module alongside `engine.js` / `verify.js`, reusing `api.js` |
| New export format | Add a serializer in `report.js`; keep the report object unchanged |
| New aggregation metric | Add to `core/stats.js`; keep it pure |
| New UI language | Add `locales/<code>.json`, regenerate, extend detection in `i18n.js` |
| Scheduled / interval verification | Wrap `engine.js` in a scheduler; do not add timers inside `core/` |
| Charts and trends | Consume exported report objects; do not couple the chart layer to the DOM of the main view |

## Testing Strategy

The module layout is designed so that most logic is testable without a network.

- **Pure modules (`src/js/core/`)** are the primary unit-test target: usage
  normalization across provider shapes, difference and discrepancy-rate
  arithmetic (including division-by-zero guards), error classification, key
  masking, and repeat aggregation. They have no DOM and no I/O.
- **UMD reuse.** Because modules are UMD, tests `require()` the same files the
  browser loads. There is no separate test build, so tested code and shipped
  code cannot diverge.
- **Integration tests** cover the client and engine against a local mock
  endpoint in `demo/`, including deliberately missing `usage` fields and
  injected error statuses.
- **Live smoke check** (`scripts/live-check.mjs`, run via `npm run test:live`)
  issues a real request against a real endpoint. It is excluded from `npm test`
  because it needs credentials and consumes real quota.
- **Manual verification** covers behaviours that require a real endpoint or a
  device: authentication failures, rate limiting, streaming completion, runtime
  language switching during a run, and Android launch.

No test result figures are recorded in this document. Run `npm test` for the
current state; the project does not assert pass counts it has not observed.

See [../README.md](../README.md#testing) for the commands.
