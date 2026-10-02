# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned

- Multi-provider profiles and capability detection (see Roadmap in
  [README.md](README.md#roadmap)).

## [0.1.0] - 2025-XX-XX

Initial MVP. This release reworks the original single-file AI Token consumption
tool into a token metering and usage verification tool.

### Added

- **OpenAI-compatible API client** (`src/js/api.js`) supporting Chat
  Completions, with SSE streaming and non-streaming requests.
- **Model discovery** via `GET /models`, with graceful degradation to manual
  model entry when the endpoint is unavailable, unauthorized, empty, or returns
  malformed JSON.
- **Token Usage Test mode** (`src/js/engine.js`) for measuring consumption,
  throughput, latency, and request stability, with configurable request count,
  `max_tokens`, concurrency, model, and prompt.
- **Small-scale Verification mode** (`src/js/verify.js`) built on the same
  engine, with deliberately conservative defaults: 3 requests, `max_tokens` 100,
  concurrency 1.
- **Full usage normalization** (`src/js/core/usage.js`) covering
  `prompt_tokens`, `completion_tokens`, `total_tokens`, `prompt_tokens_details`,
  `completion_tokens_details`, `cached_tokens`, and `reasoning_tokens`, with
  absent fields explicitly marked as unknown rather than estimated.
- **Balance / quota comparison**: automatic before-and-after reads where a
  provider exposes them, and manual entry where it does not.
- **Discrepancy analysis** (`src/js/core/diff.js`) computing
  `Account Usage − API-reported Usage` and the discrepancy rate, keeping
  token-level and cost-level figures strictly separate.
- **Repeat verification with aggregation** (`src/js/core/stats.js`): mean API
  tokens, mean account deduction, mean discrepancy rate, and min/max spread
  across 3, 5, or a user-defined number of iterations.
- **Neutral result wording** with three outcomes only: no significant usage
  discrepancy detected; a usage or billing discrepancy was detected with a
  recommendation to repeat testing; insufficient data for full verification.
- **Report export** (`src/js/report.js`) in JSON and CSV, containing per-request
  detail plus the session summary and a data-completeness note.
- **Complete zh-CN / en-US internationalization** (`src/js/i18n.js`,
  `locales/*.json`) with system-locale detection on first launch and runtime
  switching that preserves configuration, in-flight tests, and results.
- **Local configuration persistence** (`src/js/store.js`) backed by
  `localStorage`.
- **API key masking** (`src/js/core/mask.js`) applied to every log line,
  message, and exported report, rendering keys as `sk-1234************abcd`.
- **Error classification** (`src/js/core/errors.js`) covering `401`, `403`,
  `429`, `5xx`, network/DNS failures, timeouts, missing `/models`, absent
  `usage`, malformed JSON, and non-OpenAI-compatible responses, all surfaced as
  readable messages that never blank the page.
- **Stop control** that aborts in-flight requests while retaining partial
  results.
- **Locale build script** (`scripts/build-locales.mjs`) generating
  `locales/*.js` from `locales/*.json` so translations load offline over
  `file://`.
- **Static server** (`scripts/serve.mjs`) on `http://127.0.0.1:4173`.
- **Live API smoke check** (`scripts/live-check.mjs`), driven by the
  `ARM_BASE_URL`, `ARM_API_KEY`, and `ARM_MODEL` environment variables.
- **Unit and integration tests** (`tests/`) runnable with `npm test`.
- **Local mock endpoint** (`demo/`) for offline development without consuming
  real quota.
- **WebCatX Android packaging** configuration and documentation
  (`android/`, `appConfig.xlt`).
- **Documentation**: English `README.md`, Chinese `README.zh-CN.md`,
  `docs/ARCHITECTURE.md`, `LICENSE` (MIT), and this changelog.

### Changed

- **Project scope.** The original tool counted `usage.total_tokens` from
  concurrent Chat Completions requests to reach a target token volume. AI Relay
  Meter reframes the same request machinery as a measurement and verification
  instrument: it records every usage field, compares API-reported usage against
  account-level usage, and exports reviewable data.
- **Request engine.** Replaced the "consume until target reached" loop with a
  bounded, request-count-driven engine whose defaults (3 requests,
  `max_tokens` 100, concurrency 1) minimize accidental spend.
- **Usage recording.** Extended from `total_tokens` only to the full usage
  object, including details, cached, and reasoning token fields.
- **Result language.** Replaced the original "test station" framing with neutral
  developer-tool wording. The application measures, records, compares, and
  displays; it does not conclude that a provider is misreporting.
- **Interface.** Replaced the dark GitHub-style single-page layout with an
  Apple / macOS / iOS-inspired minimal design: generous whitespace, rounded
  corners, clear hierarchy, lightweight glass effects, mobile-first and
  responsive.
- **Architecture.** Split the single `index.html` into focused modules
  (`src/js/`, `src/js/core/`, `src/css/`). Modules are UMD-style classic scripts
  attached to a `window.ARM` namespace, so the same files serve the browser and
  `node --test` without a bundler.
- **Configuration.** API host, key, and model are now persisted in
  `localStorage` and restored on launch, rather than re-entered each session.
- **Model selection.** The model list now renders as selectable chips with
  explicit success and failure states.
- **Timing.** Retained millisecond-precision elapsed timing, now reported per
  request as latency alongside run-level totals.
- **Documentation.** Rewrote the project documentation in both English and
  Chinese; the WebCatX bridge references (`docs/webcat-api.md`,
  `docs/webcat-quick.md`) are carried over from the original project unchanged.

### Removed

- **Target-token consumption loop.** The "target token volume" input and the
  worker loop that ran until a token target was reached were removed. That
  behaviour encouraged large, costly runs, which conflicts with the
  conservative-defaults requirement. The underlying concurrent request
  machinery was retained and repurposed.
- **Long-form system prompt.** The original forced long-form system prompt,
  designed to maximize output length, was replaced by a short default prompt
  that limits consumption.
- **"Test station" naming and UI copy.** The original product name, title, and
  Chinese-only interface strings were removed in favour of the AI Relay Meter
  identity and a fully internationalized string set.
- **Hard-coded default host.** The pre-filled DeepSeek base URL was removed so
  that no single provider is presented as the default or only option. The field
  is left empty for the user to fill in.

### Notes

- This project does not fabricate token counts, balances, or test results. Where
  a provider does not return a value, the interface reports it as unknown.
- Verification outcomes are intentionally limited to neutral statements, and
  every run can be repeated and its raw data exported for independent review.

[Unreleased]: https://github.com/your-org/AI-Relay-Meter/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/your-org/AI-Relay-Meter/releases/tag/v0.1.0
