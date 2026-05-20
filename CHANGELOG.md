# Changelog

All notable changes to this project will be documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project does not yet follow Semantic Versioning — version numbers will
be assigned once a stable public API is established.

---

## [0.2.0] — 2026-05-19

Second release of `gradle-deps-monitor`. Two themes — one additive
feature phase, one breaking default change that pre-1.0 SemVer
permits without a new MAJOR.

### Highlights

- **Phase 8 — Analytics & insights v1**: the first Claude Code
  skill in the project, `/analyze-freeze`, backed by a DuckDB-based
  canonical query library that consumes the RFC-0017 CSVs and
  produces a Markdown insight summary. Eight canonical queries
  cover every CSV dimension, including the compound
  "duplicate-with-CVE" exposure that the narrative `freeze.md`
  doesn't surface. Analytics dependencies are opt-in via a new
  `[analytics]` extra (`pip install -e ".[analytics]"`); the
  default install path is untouched.
- **Slack output is now opt-in** (BREAKING default change).
  `gradle-deps-monitor check` and `diff` no longer write
  `freeze-slack.json` / `freeze-diff-slack.json` by default —
  pass `--slack` (or set `[output] slack = true` in
  `gradle-deps-monitor.toml`) to re-enable. See RFC-0034
  for rationale and migration.
- **Architecture documented**: ADR-0010 pins the analytics stack
  (DuckDB query layer; `tabulate` as the only presentation library;
  no pandas in v1); RFC-0033 + RFC-0034 land as the two feature
  RFCs for this release.

### Added

#### Analytics & the `/analyze-freeze` skill (RFC-0033, ADR-0010)

- **`/analyze-freeze <report-dir>`** — first project-level Claude
  Code skill. Lives at `.claude/skills/analyze-freeze/SKILL.md`.
  Loads the two RFC-0017 CSVs into in-memory DuckDB tables, runs
  every canonical query in `tools/analytics/queries/` in numeric
  order, and emits a Markdown summary with one `## <Title>`
  section per query.
- **Canonical query library** at `tools/analytics/queries/`. Eight
  queries cover every distinct CSV dimension: `top_risk`,
  `drift_by_severity`, `compound_security_duplicates` (the
  RFC-0017 issue-#13 compound exposure), `unstable_prerelease_in_prod`,
  `inactive_or_unhealthy`, `license_risk`,
  `finding_severity_breakdown`, `bom_coverage`. `queries/INDEX.md`
  codifies what counts as "canonical" and how to add a new one.
- **Analytics runner** at `tools/analytics/runner.py`. Verifies
  CSV headers against the RFC-0017 contract (fail-fast on drift)
  before loading. Detects scanner-not-run cases (e.g. `--risk-score`
  off, no CVE credentials) and surfaces them as actionable hints
  instead of generic "no rows" footers.
- **`[analytics]` optional extra** in `pyproject.toml`
  (`duckdb>=1.1,<2.0`, `tabulate>=0.9`). Install via
  `pip install -e ".[analytics]"`. Default install path remains
  unchanged; users who only run `gradle-deps-monitor check` pay
  no extra cost.
- **`Tip: run /analyze-freeze …` line** in the post-`check`
  console summary, shown after the file listing when both CSVs
  land (the default).
- **User Guide chapter**
  [`docs/user-guide/analyzing-a-freeze-report.md`](docs/user-guide/analyzing-a-freeze-report.md):
  skill-vs-sub-agent primer (the first skill in the project),
  worked example against the bundled Sunflower freeze, and the
  procedure for adding a new canonical query.
- **CONTRIBUTING reading order** gains a "Skills" step pointing
  at `.claude/skills/` with cross-links to ADR-0010 and the new
  User Guide chapter.

#### Output configuration (RFC-0034)

- **`--slack` / `--no-slack` flag** on both `check` and `diff`.
  Tri-state Typer option: `--slack` enables, `--no-slack`
  disables, unset defers to the config file.
- **`[output]` config section** in `gradle-deps-monitor.toml`.
  Previously reserved-but-ignored; now consumed and validated.
  Today's only knob is `slack = true|false`; future opt-in
  writers (e.g. the planned RFC-0010 HTML export) follow the
  same per-writer boolean pattern.

#### Decisions and design docs

- [ADR-0010](docs/adr/0010-analytics-stack-duckdb.md) — Analytics
  stack: DuckDB as the query layer for downstream insights;
  `tabulate` as the only presentation library (no pandas in v1).
  Pandas reconsideration is explicitly deferred to RFC-0010
  (HTML export) time, where build-time data shaping may earn it
  back. Documents the empirical pandas-3.x finding that pivoted
  the original stack design mid-tracer.
- [RFC-0033](docs/proposals/0033-analyze-freeze-skill.md) —
  `/analyze-freeze` skill + canonical query library.
- [RFC-0034](docs/proposals/0034-output-slack-opt-in.md) — Slack
  output becomes opt-in.

### Changed

- **BREAKING — Slack output is opt-in.** `gradle-deps-monitor check`
  default output set drops from 5 files to 4 (no `freeze-slack.json`);
  `diff` drops from 3 to 2 (no `freeze-diff-slack.json`). The four
  load-bearing outputs (`freeze.md` canonical for humans;
  `freeze.json` consumed by `diff`; the two CSVs consumed by
  `/analyze-freeze`) are unchanged. **Migration:**
  ```bash
  gradle-deps-monitor check ./gradle --slack
  gradle-deps-monitor diff old.json new.json --slack
  ```
  or:
  ```toml
  [output]
  slack = true
  ```
  Pre-1.0 SemVer (0.x.y) permits this default change without a
  MAJOR bump. See [RFC-0034](docs/proposals/0034-output-slack-opt-in.md).
- **Phase 8 follow-ups header** added to `docs/roadmap.md` for
  small breaking-default changes between Phase 8 v1 and v0.2.0.
  RFC-0034 was the only entry; the cluster ships closed in this
  release.

### Notes for upgraders from v0.1.0

If you POST `freeze-slack.json` to a Slack incoming webhook from
CI, add `--slack` to your `gradle-deps-monitor check` invocation
(or set `[output] slack = true` in `gradle-deps-monitor.toml`).
Same shape for `diff`. Nothing else changed by default — the
canonical Markdown / JSON / CSV outputs are unaffected.

To use the new `/analyze-freeze` skill, install the optional
analytics extra:

```bash
pip install -e ".[analytics]"
```

Then invoke `/analyze-freeze <report-dir>` from Claude Code, or
call `python tools/analytics/runner.py --dir <report-dir>` directly.

---

## [0.1.0] — 2026-05-18

First public release of `gradle-deps-monitor` — a freeze-time
due-diligence CLI for Android projects that read their dependencies
from a Gradle version catalog (`libs.versions.toml`).

### Highlights

- **One command, five outputs:** `gradle-deps-monitor check ./gradle`
  produces a human-readable Markdown report, a SemVer-versioned JSON
  snapshot, a Slack Block Kit summary, and two CSV files (one row per
  library, one row per finding) in 5–10 seconds for a typical 200-
  library catalog.
- **Eleven dimensions checked** out of the box: version drift, catalog
  health (9 rules), CVE scanning (GHSA + OSS Index), Play Store
  compliance, toolchain compatibility (Kotlin / Compose / KSP / AGP),
  library health (curated KB + POM `<relocation>` + inactivity),
  BoM resolution, license tier, changelog scraping with breaking-change
  heuristic, plus opt-in module-usage scanning and a composite Risk
  Score.
- **CI-friendly:** `--fail-on-errors` (exit 1 on critical CVE /
  compliance violation / toolchain error / strong-copyleft license),
  `--warn-on <categories>` for opt-in surfacing, and automatic
  GitHub Actions workflow annotations when `GITHUB_ACTIONS=true`.
  Exit codes follow `sysexits.h` (0 / 1 / 2 / 3).
- **Operationally robust:** resilient HTTP transport with exponential
  backoff + jitter + `Retry-After` honoring, namespaced disk cache
  with per-source TTL and CLI bypass / purge / override, atomic
  report writes via temp-file + rename so a process killed mid-write
  never leaves a half-rendered file behind.
- **Pragmatic clean architecture:** six layers enforced by
  `import-linter` (5 contracts), 1219 tests across all layers, CI
  matrix on Python 3.11 / 3.12 / 3.13 / 3.14.
- **Documented:** five-chapter User Guide at `docs/user-guide/`
  covering getting started, configuration, every feature, CI
  integration, and troubleshooting.

### Added

#### CLI & gatekeeper

- `check` command with `--out` / `-o` for output directory,
  `--module-usage` / `-m` and `--risk-score` / `-r` for opt-in
  heavier checks, `--no-cache` / `--clear-cache` / `--cache-ttl`
  for per-invocation cache control, and `--fail-on-errors` /
  `--warn-on` for CI policy enforcement (RFC-0018 v1).
- `diff` command compares two `freeze.json` snapshots and writes
  Markdown / JSON / Slack diff reports.
- `--version` flag.
- `GITHUB_ACTIONS=true` auto-detection emits one `::error
  file=…::…` or `::warning file=…::…` workflow annotation per
  policy hit so reviewers see violations inline in PR file diffs.
  Special characters (`%`, CR, LF, `:`) escaped per the workflow-
  commands spec.

#### Reports & outputs

- **Markdown** report (`freeze.md`) with sections for Outdated
  summary, BoMs, Libraries, Plugins, Bundles, Catalog Health,
  Security, Play Store Compliance, Active Rejections, Toolchain
  Compatibility, Library Health, Major Upgrades, Module Usage Map,
  License Audit, Risk Score. Each section renders an explicit
  placeholder when its adapter ran with no findings, so readers can
  distinguish "scanned, clean" from "didn't scan".
- **JSON** report (`freeze.json`), schema 1.7.0 per ADR-0008 SemVer.
- **Slack** report (`freeze-slack.json`) as Block Kit JSON for
  incoming-webhook posting.
- **Inventory CSV** (`freeze-inventory.csv`) — one row per catalog
  library, 15 columns joining version / drift / risk / vulnerability
  / compliance / license / health / BoM-parent / `duplicate_of`.
  RFC-0017.
- **Findings CSV** (`freeze-findings.csv`) — one row per finding
  across every section, with cross-section `common_severity`
  vocabulary. RFC-0017.
- Console executive summary via Rich, with a unified severity
  vocabulary (`CommonSeverity`) so each severity renders with
  consistent emoji / colour across Markdown, Slack, and console.
  RFC-0016 / RFC-0016b.

#### Checks & domain signals

- **Catalog Health** — nine rules: duplicate library, unresolved
  `version.ref`, inconsistent alias naming, no `[plugins]` block,
  orphan version keys, inline version literals, no `[bundles]`
  block, duplicate version values.
- **CVE scanning** — GHSA (via GraphQL) and OSS Index (REST)
  scanners, composed when both credential sources are present;
  composite scanner deduplicates by `(coordinate, advisory-id)`.
- **Play Store compliance** — bundled knowledge base flags
  deprecated APIs (SafetyNet, Play Core Splitcompat, …) and
  `targetSdk` drift against Google's current minimum.
- **Toolchain compatibility** — Kotlin ↔ Compose Compiler,
  Kotlin ↔ KSP, AGP ↔ Gradle wrapper matrices.
- **Library health** — curated knowledge base (26 entries) +
  Maven POM `<relocation>` + inactivity heuristic via
  `<lastUpdated>`. JSR / Jakarta libraries are exempted from the
  inactivity heuristic since they are frozen by design.
- **BoM resolution** — detects `*-bom` / `*-platform` artifacts
  (including release-line-suffixed variants like
  `compose-bom-alpha`), fetches their `<dependencyManagement>`,
  enriches catalog children with the BoM-resolved version.
- **License audit** — POM `<licenses>` classified into Permissive
  / Weak copyleft / Strong copyleft / Unknown. GPL-with-Classpath-
  Exception correctly downgrades to Permissive (RFC-0023).
- **Changelog scraping** — GitHub Releases + `CHANGELOG.md` for
  major upgrades, with a `LIKELY` / `CLEAN` / `UNKNOWN` breaking-
  change heuristic. `ChangelogFetchStats` surfaces silent
  degradation under rate limits (RFC-0024 PR #2).
- **Active Rejections** Markdown section listing libraries with
  catalog-level `reject` constraints — intentional negative pins.
- **Module Usage map** (opt-in, `--module-usage`) — static scan of
  `build.gradle(.kts)` files counting `impl` / `api` / `test only`
  references per catalog alias. Recognises every Gradle catalog
  accessor form: dotted, camelCase, bundle expansion, BoM wrappers
  (`platform(...)`, `enforcedPlatform(...)`, `testFixtures(...)`),
  and underscore-only alias variants (RFC-0019 + RFC-0022).
- **Risk Score** (opt-in, `--risk-score`) — composite 0-100 per
  library across six weighted dimensions (outdatedness 25, cve 30,
  abandonment 15, blast_radius 15, compliance 10, license 5).
  Bucketed into LOW / MEDIUM / HIGH / CRITICAL via three
  configurable thresholds. Experimental — most informative as a
  trend across freezes (ADR-0004).
- **`Stability.PRE_1_0`** classification for naked `0.x.y` versions
  per SemVer §4, distinct from `STABLE` (RFC-0026).
- **`RichVersion`** domain value object for Gradle rich-version
  constraints (`strictly` / `require` / `prefer` / `reject`); parser
  accepts rich blocks in both `[versions]` and per-library entries
  (RFC-0020).

#### Operational features

- **Cache controls** (RFC-0029) — `--no-cache` for ephemeral tempdir
  cache (atexit-cleaned), `--clear-cache` for persistent-cache
  purge, `--cache-ttl SECONDS` for per-run TTL override.
  `GRADLE_DEPS_MONITOR_CACHE_ROOT` env var redirects the cache
  root for CI runners with read-only `$HOME`. `[cache]` section in
  `gradle-deps-monitor.toml` configures `root`,
  `ttl_seconds_maven`, `ttl_seconds_advisory`. Negative-cache
  entries namespaced separately (`:404:`) from positives (`:ok:`)
  so they can be purged independently.
- **Shared HTTP resilience layer** (RFC-0030) — `HttpPolicy` +
  `ResilientTransport` (stateless `httpx.AsyncBaseTransport`
  wrapper with retry-on-transient-failure, exponential backoff with
  full jitter, `Retry-After` honoring) + `make_resilient_client`
  factory. Adopted by every outbound adapter (GHSA, OSS Index,
  Maven Central, Google Maven, changelog scraper, POM license,
  library health, BoM resolver). Per-adapter timeouts consolidated
  through `HttpPolicy`.
- **Concurrency caps** — `asyncio.Semaphore(20)` bounds the
  per-library `asyncio.gather` calls in the GHSA scanner, changelog
  fetcher, and library-health checker. A 170-library catalog no
  longer fans out 170 simultaneous outbound connections.
- **Atomic report writes** (RFC-0032) — all eight writers (Markdown
  / JSON / Slack for both `check` and `diff`, plus inventory and
  findings CSV) serialise through a shared `atomic_write` context
  manager that buffers output to a sibling temp file and renames it
  into place via `os.replace`. A killed process never leaves a
  half-rendered file behind.

#### Architecture, testing, docs

- **Pragmatic clean architecture** (ADR-0006) — six layers
  (`domain` / `application` / `checks` / `infrastructure` /
  `presentation` / `bootstrap`), with 5 `import-linter` contracts
  enforcing layer dependencies in CI.
- **CI matrix** — GitHub Actions runs the five-stage suite
  (`ruff check`, `ruff format --check`, `mypy`, `lint-imports`,
  `pytest`) on Python 3.11 / 3.12 / 3.13 / 3.14.
- **1 219 tests** across domain / application / infrastructure /
  presentation / wiring layers, including composition-root
  unit tests for `bootstrap.py` (RFC-0031, 18 tests, closes audit
  risk R9).
- **Architecture diagrams** at `docs/diagrams/` — system context,
  layered dependencies, port↔adapter map, async use-case pipeline.
- **User Guide** at `docs/user-guide/` (RFC-0021) — five chapters:
  Getting Started, Configuration, Feature Deep-Dives, CI
  Integration, Troubleshooting.

### Changed

- **Exit-code semantics** follow `sysexits.h`: `0` success,
  `1` policy violation (`--fail-on-errors`), `2` usage error
  (e.g. unknown `--warn-on` category), `3` configuration / parse
  error (TOML missing, unreadable, or malformed).
- **HTTP-resilience policy consolidated** — every adapter's
  per-module `_HTTP_TIMEOUT` constant retired; call sites construct
  `HttpPolicy(timeout_seconds=…)` inline. `PomLicenseChecker.__init__`
  drops the unused `http_timeout` parameter.
- **Markdown report no longer silently elides empty sections** —
  Catalog Health, Compliance, Toolchain, Library Health, and Major
  Upgrades render `✅ no findings` rather than disappearing.
  Security distinguishes `⊘ scan not configured` from `✅ no known
  advisories` via a new authoritative `security_scanned` flag on
  `FreezeReport` (RFC-0028).
- **Console Risk Score and Security summaries** enumerate every
  populated severity bucket (`N critical, M high, K medium,
  L low`) instead of collapsing non-CRITICAL/non-HIGH entries into
  `N other`.
- **`GenerateFreezeReport.execute` orchestrates adapters in
  parallel** — the six adapters that consume the enriched catalog
  without depending on each other (vulnerability scanner, library
  health, changelog fetcher, module-usage scanner, license checker,
  version-status resolver) run concurrently via `asyncio.gather`
  (RFC-0025). Risk score still runs after them as it consumes
  their output.
- **`GitHubAdvisoryScanner._scan_with`** runs per-library lookups
  in parallel with `asyncio.gather`, bounded by an
  `asyncio.Semaphore(20)` (RFC-0024 PR #1).
- **`GradleModuleScanner.scan` is now `async`** — per-module file
  reads + regex parsing run in parallel via `asyncio.to_thread` +
  `asyncio.gather`. `ModuleUsageScanner` port signature changed to
  `async def scan(...)` matching the other async adapter ports.
- **`GenerateFreezeReport.execute` is now `async`** — the previous
  pattern of seven independent `asyncio.run(...)` calls inside the
  use case is replaced by a single `asyncio.run(...)` at the CLI
  entry.
- **`ToolchainCompatibilityChecker`** no longer re-parses the
  catalog TOML — it consumes `Catalog.versions` directly per the
  RFC-0020 checker contract.

### Fixed

- **`_CACHE_ROOT` wiring bug** for GHSA + OSS Index scanners —
  both previously fell back to relative `Path(".cache/ghsa")` /
  `Path(".cache/ossindex")` defaults while the Maven resolver was
  correctly wired to `~/.cache/gradle-deps-monitor/maven`. All
  three sites now share the resolved cache root (RFC-0029).
- **Pre-release versions reported as "latest"** —
  `MavenMetadataRegistry._parse_release` now scans
  `<versioning><versions>` in reverse document order for the
  latest stable entry when the publisher set `<release>` to a
  pre-release tag. Live-validated against
  `com.google.protobuf:protoc` which had `<release>21.0-rc-1`
  while the actual stable line continued at 4.x.y (RFC-0027).
- **JSR / Jakarta EE libraries falsely flagged as abandoned** —
  the inactivity heuristic now skips groups under `javax` or
  `jakarta`. A 5 780-day-old `javax.inject:javax.inject` is a
  feature, not a signal.
- **`PomLicenseChecker` echoed placeholder license names** like
  literal `LICENSE` / `LICENSE.txt` / `License` in the License
  Audit. Such values are now normalised to `None` at
  finding-construction time.
- **`Library.is_bom_candidate` missed release-line-suffixed BoM
  artifacts** like `compose-bom-alpha` / `foo-platform-rc1`.
- **Console `Outdated (N)` summary** now matches the Markdown
  total — pre-fix the console count excluded libraries whose
  drift resolved to `UNKNOWN`.
- **Markdown `Outdated summary`** carries a one-line note
  explaining that the Libraries table's Drift column targets the
  latest available version (including pre-releases) whereas the
  Major Upgrades section targets the latest stable.
- **`PomLicenseChecker` false-positive on GPL with Classpath
  Exception** — classifier now detects the qualifier before the
  GPL keyword cascade and downgrades to PERMISSIVE. Closes the
  `com.android.tools:desugar_jdk_libs` mis-classification (RFC-0023).
- **`GradleModuleScanner`** now matches catalog aliases that use
  only underscores, detects BoM applications wrapped in
  `platform()` / `enforcedPlatform()` / `testFixtures()`,
  recognises camelCase accessors, survives a single corrupt
  `build.gradle(.kts)` file via a new `MOD-001` finding, and
  credits bundle members declared via `libs.bundles.<name>`
  (RFC-0019 / RFC-0022).
- **Parser no longer crashes on Gradle rich-version blocks** in
  either `[versions]` or per-library `[libraries]` / `[plugins]`
  entries (RFC-0020).

### Removed

- Legacy Bash wrapper (`check-dependencies.sh`) and original
  Python script (`version-stats.py`) — superseded by the
  Typer-based CLI.

---

## History

Pre-release internal milestones (never published to PyPI; no git
tag) shaped the codebase before this `v0.1.0` cut:

- **Phase 1 close (designated `0.1.0` in an earlier CHANGELOG)** —
  2026-05-04. Initial six-layer architecture, TOML parser, Maven
  cache, Markdown / JSON / Slack writers, eight Catalog Health
  rules, Rich console summary.
- **Phase 2–4** — version drift, BoM resolution, license audit,
  Play Store compliance, toolchain matrices, library health.
- **Phase 5** — CSV exports (RFC-0017), Module Scanner
  (RFC-0019), Rich Versions (RFC-0020), CI Gatekeeper v1
  (RFC-0018), User Guide (RFC-0021).
- **Phase 6** — stress-test follow-ups (RFC-0022 through -0028)
  surfaced by running the tool end-to-end against real Android
  catalogs.
- **Phase 7** — stability hardening (RFC-0029 cache controls,
  RFC-0030 HTTP resilience, RFC-0031 bootstrap composition tests,
  RFC-0032 atomic writes).

This `v0.1.0` consolidates all of the above into the first
publishable artifact. The next release will start the post-launch
`0.x` series.

---

[0.1.0]: https://github.com/gustavo-pedreros/experiment-toml-deps-checker/releases/tag/v0.1.0
