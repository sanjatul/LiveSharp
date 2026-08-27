# Phase 00 — Governance & Scaffolding

| | |
|---|---|
| **Version at exit** | `0.1.0-alpha` (git tag `v0.1.0-alpha.1`) |
| **Depends on** | nothing |
| **Branch** | `feat/phase-00-governance` |
| **Tasks** | `P00.T1` … `P00.T6` |
| **Read before starting** | `EXECUTION_PLAN.md` §5–§11 |

---

## Goal

Turn an empty repository into a professional, automated, self-documenting open-source .NET
repository. At the end of this phase there is **no product code at all** — but every guard rail
that keeps the next 18 phases honest is in place and proven in CI.

This phase exists because Spec §5 forbids starting feature work before governance and architecture
are written down.

---

## Non-goals — DO NOT implement in this phase

- No `src/LiveSharp.*` projects with real types. Phase 1 creates them.
- No abstractions, interfaces, services, hubs, or DI extension methods.
- No SignalR. No WebRTC. No chat.
- No NuGet publishing. The release workflow is created but gated and never runs automatically.
- No documentation site build (DocFX comes in Phase 17); only the `docs/` folder skeleton and
  hand-written Markdown.

---

## Tasks

### P00.T1 — Repository skeleton, build props, solution

**Deliverables**

```
global.json
Directory.Build.props
Directory.Build.targets
Directory.Packages.props
LiveSharp.slnx
.editorconfig
.gitattributes
.gitignore
NuGet.config
.config/dotnet-tools.json
```

**`global.json`** — exactly as the Spec's closing section mandates:

```json
{
  "sdk": {
    "version": "10.0.302",
    "rollForward": "latestFeature"
  }
}
```

**`Directory.Build.props`** — applies to everything. Key properties:

| Property | Value | Reason |
|---|---|---|
| `TargetFramework` | `net10.0` | ADR-001 |
| `LangVersion` | `latest` | |
| `Nullable` | `enable` | Spec §36 |
| `ImplicitUsings` | `enable` | |
| `TreatWarningsAsErrors` | `true` | |
| `WarningsNotAsErrors` | `NU1901;NU1902;NU1903;NU1904` | audit advisories must not block a local build; CI has a dedicated audit step |
| `NuGetAudit` / `NuGetAuditMode` | `true` / `all` | supply-chain surfacing |
| `EnforceCodeStyleInBuild` | `true` | `.editorconfig` rules become build errors |
| `EnableNETAnalyzers` | `true` | |
| `AnalysisLevel` | `latest-Recommended` | |
| `CodeAnalysisTreatWarningsAsErrors` | `true` | |
| `ManagePackageVersionsCentrally` | `true` | via `Directory.Packages.props` |
| `Deterministic` | `true` | |
| `ContinuousIntegrationBuild` | `$(TF_BUILD)`/`$(GITHUB_ACTIONS)` | Source Link correctness |
| `PublishRepositoryUrl` | `true` | Source Link (in-SDK, no package) |
| `EmbedUntrackedSources` | `true` | |
| `IncludeSymbols` | `true` | |
| `SymbolPackageFormat` | `snupkg` | |
| `GenerateDocumentationFile` | `true` for `src/**` | Spec §34 |
| `EnablePackageValidation` | `false` for now | turned on in Phase 18 once a baseline exists |

Split the file so that `src/**` gets packaging + doc-generation + public-API rules while
`tests/**`, `benchmarks/**`, `examples/**`, and `clients/**` do not. Use a directory-scoped
`Directory.Build.props` in `src/`, `tests/`, and `examples/` rather than a single file full of
`Condition="$(MSBuildProjectDirectory.Contains(...))"` hacks.

**`src/Directory.Build.props`** additionally sets: `IsPackable=true`, `Authors`, `Company`,
`Product=LiveSharp`, `PackageLicenseExpression=MIT`, `PackageProjectUrl`,
`RepositoryUrl`/`RepositoryType=git`, `PackageReadmeFile=README.md`,
`PackageTags=realtime;signalr;chat;webrtc;presence;aspnetcore`,
`PackageIcon=icon.png`, and includes the root `README.md` + `icon.png` in the package.

**`tests/Directory.Build.props`** sets `IsPackable=false`, `IsTestProject=true`,
`GenerateDocumentationFile=false`, `NoWarn=CS1591`, and adds shared test package references.

**`Directory.Packages.props`** — every version from `EXECUTION_PLAN.md` §11. Add only the ones
needed now (test stack, analyzers, MinVer); later phases append.

**`LiveSharp.slnx`** — the .NET 10 XML solution format. Contains only `src/**` and `tests/**`
projects. `examples/**` and `clients/dotnet/**` are **not** in this solution (ADR-010, D5) —
`examples/LiveSharp.Examples.slnx` is created in Phase 4, and `clients/dotnet/LiveSharp.Client`
joins `LiveSharp.slnx` in Phase 3 because it is a shipping package.

> **Risk gate.** Before finishing this task, prove all four of `dotnet build LiveSharp.slnx`,
> `dotnet test LiveSharp.slnx`, `dotnet pack LiveSharp.slnx`, and
> `dotnet format LiveSharp.slnx --verify-no-changes` work against the `.slnx` format on this SDK.
> If any fails, fall back to `LiveSharp.sln`, record the failure in `DEVELOPMENT_LOG.md` under
> **Known Issues**, and note it in `ADR-010`. Do not spend a session fighting the tooling.

**`.editorconfig`** — full C# style ruleset. Minimum required rules, all at `error` severity:
file-scoped namespaces, `var` usage consistency, expression-bodied member preferences, `this.`
never, braces required for multi-line, sorted usings with `System` first, sealed-by-default
guidance via analyzer, `CA2007` (ConfigureAwait) **disabled** for ASP.NET Core projects and
**enabled** for `LiveSharp.Abstractions`/`Core`/`Client` (libraries that may run under a sync
context). Also set `dotnet_diagnostic.CS1591.severity = error` for `src/`.

**`.gitattributes`** — `* text=auto eol=lf`, `*.ps1 text eol=crlf`, binary markers for images.
**This also fixes the existing UTF-16 `README.md`** — see `P00.T2`.

**`.config/dotnet-tools.json`** — local tool manifest. Add nothing yet beyond what is needed;
`docfx` and `reportgenerator` are added in Phase 17 and Phase 16 respectively. Creating the empty
manifest now means later phases just run `dotnet tool install`.

**Tests** — none (no code yet). The verification is that the four `dotnet` commands succeed.

---

### P00.T2 — Community and legal files

**Deliverables**

```
LICENSE
README.md            <- REWRITE, and re-encode as UTF-8 without BOM
CONTRIBUTING.md
CODE_OF_CONDUCT.md
SECURITY.md
CHANGELOG.md
ROADMAP.md
AGENTS.md
DEVELOPMENT_LOG.md
assets/icon.png      <- placeholder is acceptable; must be a valid PNG
```

> **Do not skip this:** the current `README.md` is **UTF-16 LE with a BOM** and contains one line.
> Delete and recreate it as UTF-8 (no BOM, LF endings). Verify with
> `Format-Hex -Path README.md -Count 4` — the first bytes must be `23 20 4C 69` (`# Li`), not
> `FF FE`.

**`LICENSE`** — MIT, current year, copyright holder `LiveSharp contributors`.

**`README.md`** — follow Spec §31's section list, but **honesty is mandatory** (Spec §61). At the
end of Phase 0 the only truthful status is "nothing is implemented yet". Structure:

- Title, one-line description, badge row (CI status, NuGet — the NuGet badge may be added but must
  point at a real package ID even before publish).
- **Status: pre-alpha. No public release yet.** A prominent callout.
- Project overview / Why this project exists / Goals / **Non-Goals** (copy
  `EXECUTION_PLAN.md` §13 verbatim).
- **Feature status table** with `Available` / `Preview` / `Experimental` / `Planned` columns. Every
  row is `Planned` in Phase 0. Each later phase flips exactly the rows it shipped.
- Architecture diagram (ASCII, matching `EXECUTION_PLAN.md` §3).
- Installation / Quick Start sections marked **"not yet available"** — do not write aspirational
  code samples that do not compile. A single "intended developer experience" snippet is acceptable
  if labelled as a design target.
- Roadmap link, Versioning, Contributing, Development, Testing, License.

**`CONTRIBUTING.md`** — prerequisites (.NET 10.0.302 SDK, Node 22+, pnpm), clone/build/test
commands, `scripts/verify.ps1` as the pre-PR gate, branch naming, conventional commit format,
the PR description template, how to add an ADR, how to add a dependency (points at Spec §37),
and a statement that examples and docs are part of a feature, not follow-up work.

**`CODE_OF_CONDUCT.md`** — Contributor Covenant 2.1 verbatim with a real contact address
placeholder clearly marked `TODO(maintainer)`.

**`SECURITY.md`** — supported versions table, private reporting via GitHub Security Advisories
(not public issues), response-time commitment, scope statement, and a "Security model" section
stating: LiveSharp authenticates via the host application's ASP.NET Core authentication; the server
never sees media; TURN credentials are ephemeral; message content is not encrypted end-to-end
pre-1.0 (say this plainly — do not imply E2EE). Expanded substantially in Phase 9 and Phase 18.

**`CHANGELOG.md`** — Keep a Changelog 1.1.0 format, SemVer, with an `## [Unreleased]` section
containing `### Added` for the scaffolding.

**`ROADMAP.md`** — the exact status markers from Spec §43 (`[x] [~] [ ] [-]`), organised into the
same 19 phases as `EXECUTION_PLAN.md` §9 with every task ID listed. This file is the
human-readable progress view; `EXECUTION_PLAN.md` §9 is the index. Keep them consistent.

**`AGENTS.md`** — the operating manual for every future AI session. Contents:

1. "Read `EXECUTION_PLAN.md` first" as line one.
2. The session-start and session-end checklists from `EXECUTION_PLAN.md` §7, inlined so an agent
   that only reads `AGENTS.md` still behaves correctly.
3. The non-negotiable code rules from `EXECUTION_PLAN.md` §5.2, condensed.
4. The hard stops from `EXECUTION_PLAN.md` §7.4.
5. Commands: build, test, format, verify, run an example.
6. Repository map.
7. "Never" list: never read future phase files, never delete a failing test, never add an
   undeclared dependency, never mark roadmap items done without docs+tests, never log secrets or
   message content, never call `DateTime.UtcNow`.

**`DEVELOPMENT_LOG.md`** — created with the exact schema in `EXECUTION_PLAN.md` §7.3, filled in
for Phase 0.

---

### P00.T3 — Architecture decision records

**Deliverables**

```
docs/architecture/decisions/README.md            <- index + ADR process + template
docs/architecture/decisions/ADR-000-project-name-and-scope.md
docs/architecture/decisions/ADR-001-target-framework-policy.md
docs/architecture/decisions/ADR-002-license.md
docs/architecture/decisions/ADR-003-testing-strategy.md
docs/architecture/decisions/ADR-004-package-boundaries.md
docs/architecture/decisions/ADR-005-wire-protocol.md
docs/architecture/decisions/ADR-006-webrtc-strategy.md
docs/architecture/decisions/ADR-007-error-model.md
docs/architecture/decisions/ADR-008-time-abstraction.md
docs/architecture/decisions/ADR-009-identity-representation.md
docs/architecture/decisions/ADR-010-repository-layout.md
docs/architecture/decisions/ADR-011-public-api-governance.md
docs/architecture/overview.md
```

Every ADR uses this structure (Spec §45): `Status` (Accepted, date), `Context`, `Decision`,
`Alternatives considered` (with why each was rejected), `Consequences` (positive **and**
negative — an ADR with no downsides listed is an incomplete ADR), `Revisit when`.

Source material is `EXECUTION_PLAN.md` §2, §3, §4, and §6. **These decisions are already made.**
Your job is to write them up properly, including the honest trade-offs. Specifically:

- `ADR-004` must state deviation **D1** (no `LiveSharp.SignalR` package) and its downside: if a
  second transport is ever added, types move and that is a breaking change.
- `ADR-005` must state deviation **D3** (single hub) and its downside: developers using raw
  `@microsoft/signalr` or `HubConnection` without the LiveSharp client lose method-level
  IntelliSense and must use string operation names.
- `ADR-006` must state deviation **D2** (`.Signaling` not `.WebRTC`) and note that the package
  name will surprise people searching NuGet for "webrtc" — mitigate via `PackageTags`.
- `ADR-009` must state the downside: `string` user IDs allow accidental parameter transposition
  (`SendAsync(recipientId, senderId)` compiles), mitigated by named arguments in examples and a
  Phase 17 analyzer.

**`docs/architecture/overview.md`** — the layering diagram, the package graph, the dependency
direction rules from `EXECUTION_PLAN.md` §3, and the request/event flow through the system. This
is the document a new contributor reads first.

---

### P00.T4 — Documentation skeleton and test-runner smoke proof

**Deliverables**

```
docs/README.md                         <- docs index
docs/getting-started/README.md
docs/architecture/README.md
docs/chat/README.md
docs/presence/README.md
docs/calls/README.md
docs/webrtc/README.md
docs/configuration/README.md
docs/security/README.md
docs/scaling/README.md
docs/troubleshooting/README.md
docs/api/README.md
docs/plan/                             <- already exists, contains this plan
tests/LiveSharp.BuildSmokeTests/LiveSharp.BuildSmokeTests.csproj
tests/LiveSharp.BuildSmokeTests/ToolchainSmokeTests.cs
```

Every `docs/**/README.md` is a real page with a heading, a one-paragraph statement of what will
live there, and an explicit `Status: not yet written — see ROADMAP.md` line. Empty placeholder
files are not acceptable; a reader must never hit a blank page.

**`ToolchainSmokeTests.cs`** exists purely to prove the test stack works end-to-end in CI before
any real tests are written (`EXECUTION_PLAN.md` §12). It must contain exactly these tests:

| Test | Asserts |
|---|---|
| `Xunit_v3_runner_executes_tests` | trivially true — proves discovery + execution |
| `Shouldly_assertions_work` | `1.ShouldBe(1)` and a failing assertion caught via `Should.Throw` |
| `NSubstitute_can_substitute_an_interface` | a local interface substitute returns a configured value |
| `FakeTimeProvider_advances_deterministically` | `FakeTimeProvider.Advance` moves `GetUtcNow()` |
| `Target_framework_is_net10` | `RuntimeInformation.FrameworkDescription` starts with `.NET 10.` |

Delete this project in Phase 1 once real test projects exist, and record the deletion in
`DEVELOPMENT_LOG.md`. Its only purpose is de-risking the toolchain.

---

### P00.T5 — Scripts and the verification gate

**Deliverables**

```
scripts/build.ps1
scripts/test.ps1
scripts/format.ps1
scripts/pack.ps1
scripts/verify.ps1
scripts/README.md
```

All scripts are PowerShell 7-compatible **and** must run on Linux (`pwsh`), because CI runs on
both. Use `$ErrorActionPreference = 'Stop'`, `$PSNativeCommandUseErrorActionPreference = $true`,
explicit `exit $LASTEXITCODE` propagation, and no `Write-Host` for machine-readable output.

**`scripts/verify.ps1`** is the Definition-of-Done gate (`EXECUTION_PLAN.md` §10). Parameters:
`-Configuration Release`, `-SkipJs`, `-SkipE2E`, `-SkipPack`, `-NoRestore`. Steps in order, each
printing a clear banner and failing fast:

1. `dotnet --version` vs `global.json` — warn on mismatch.
2. `dotnet restore LiveSharp.slnx --locked-mode` if a lock file exists, else plain restore.
3. `dotnet build LiveSharp.slnx -c $Configuration --no-restore -warnaserror`
4. `dotnet test LiveSharp.slnx -c $Configuration --no-build` with TRX + coverage output to
   `artifacts/test-results/`.
5. `dotnet format LiveSharp.slnx --verify-no-changes --severity error`
6. `dotnet pack LiveSharp.slnx -c $Configuration --no-build -o artifacts/packages` (unless
   `-SkipPack`).
7. Public API check: fail if `git diff --name-only` shows a modified `PublicAPI.Unshipped.txt`
   that is not staged, or if any `PublicAPI.Unshipped.txt` contains a line the analyzer flagged.
   In Phase 0 this step is a no-op stub that prints "no shipping projects yet".
8. JS gate — stub in Phase 0, activated in Phase 3.
9. E2E gate — stub in Phase 0, activated in Phase 12.

Print a summary table at the end: step, result, elapsed. Non-zero exit on any failure.

**`scripts/README.md`** documents each script, its parameters, and when to use which.

---

### P00.T6 — GitHub automation

**Deliverables**

```
.github/workflows/ci.yml
.github/workflows/codeql.yml
.github/workflows/release.yml
.github/dependabot.yml
.github/ISSUE_TEMPLATE/config.yml
.github/ISSUE_TEMPLATE/bug_report.yml
.github/ISSUE_TEMPLATE/feature_request.yml
.github/ISSUE_TEMPLATE/question.yml
.github/PULL_REQUEST_TEMPLATE.md
.github/CODEOWNERS
```

**`ci.yml`** — triggers on `push` to `main` and all `pull_request`. Jobs:

| Job | Runs on | Does |
|---|---|---|
| `verify` | matrix `ubuntu-latest`, `windows-latest` | `actions/setup-dotnet` from `global.json`, NuGet cache, `pwsh scripts/verify.ps1 -SkipJs -SkipE2E`, upload TRX + packages as artifacts, publish test results summary |
| `format` | `ubuntu-latest` | `dotnet format --verify-no-changes` as a standalone fast-fail job |
| `audit` | `ubuntu-latest` | `dotnet list package --vulnerable --include-transitive` and fail on any finding |

Pin all actions to a commit SHA, not a tag — supply-chain hygiene, and note why in a comment.
Set `permissions: contents: read` at workflow level and elevate per job only where needed.
Set `concurrency` to cancel superseded runs on the same ref.

**`codeql.yml`** — CodeQL for `csharp` on push/PR/weekly schedule.

**`release.yml`** — triggers on tags matching `v*`. **Must not publish in Phase 0.** Structure it
now so Phase 18 only has to remove the guard:

1. `verify` job (full gate).
2. `pack` job producing `.nupkg` + `.snupkg` artifacts.
3. `publish` job with `environment: nuget-release` (a protected GitHub environment requiring
   manual approval) and an `if: false` guard plus a comment
   `# ENABLED IN PHASE 18 — see docs/plan/phase-18-hardening.md`.
4. `github-release` job creating a draft release with notes extracted from `CHANGELOG.md`.

Use `--skip-duplicate` on push and `NuGet` API key from repository secrets, never inline.

**`dependabot.yml`** — ecosystems `nuget` (root, weekly, grouped by `Microsoft.*` /
`analyzers` / `test`), `github-actions` (weekly), and `npm` (`clients/js`, weekly — added now,
harmless until Phase 3).

**Issue templates** — GitHub Forms YAML, not Markdown. `bug_report.yml` must require: LiveSharp
version, .NET version, package(s) affected, transport (WebSocket/SSE/LongPolling), minimal repro,
expected vs actual, logs with secrets redacted. `config.yml` disables blank issues and links to
Discussions for questions.

**`PULL_REQUEST_TEMPLATE.md`** — the eight sections from Spec §40 plus the Definition-of-Done
checklist from `EXECUTION_PLAN.md` §5.1 as unchecked boxes.

**`CODEOWNERS`** — `* @sanjatul`.

---

## Dependency justification

| Package | Why | BCL alternative? | Licence | Notes |
|---|---|---|---|---|
| `MinVer` `7.0.0` | Derives package version from git tags, so version and tag can never disagree. | No. | MIT | Build-time only (`PrivateAssets=all`), not a runtime dependency. Alternative `Nerdbank.GitVersioning` rejected as heavier (needs a version.json and has more moving parts) for equivalent benefit. |
| `Microsoft.CodeAnalysis.PublicApiAnalyzers` `5.6.0` | Makes public API changes appear as reviewable diffs in `PublicAPI.Unshipped.txt`, which is what Spec §35 asks for. | No. | MIT | Analyzer only. |
| `Microsoft.CodeAnalysis.BannedApiAnalyzers` `5.x` | Mechanically enforces the `DateTime.UtcNow` ban (ADR-008) instead of relying on reviewer vigilance. | No. | MIT | Analyzer only. |
| `xunit.v3` `4.0.0` | Test framework (ADR-003). | No. | Apache-2.0 | |
| `Shouldly` `4.3.0` | Readable assertions. Chosen over FluentAssertions because FluentAssertions v8+ requires a paid commercial licence, which would be a trap for downstream contributors. | Partly — `Assert.*` works but reads worse. | BSD-3-Clause | |
| `NSubstitute` `6.2.0` | Mocking. Chosen over Moq following the 2024 SponsorLink incident and the resulting community trust damage. | No. | BSD-3-Clause | |
| `Microsoft.Extensions.TimeProvider.Testing` `10.9.0` | `FakeTimeProvider`, required to test presence TTL, typing expiry, ring timeouts, and reaping deterministically (ADR-008). | No. | MIT | |

---

## Documentation deltas

- `README.md` — complete rewrite (see `P00.T2`).
- `docs/README.md` + all eleven section indexes.
- `docs/architecture/overview.md` and twelve ADRs.
- `docs/plan/` — this plan (already present).
- `scripts/README.md`.

## Example deltas

None. Examples begin in Phase 4.

## CHANGELOG entry

```markdown
## [Unreleased]

### Added
- Repository scaffolding: solution, central build properties, central package management.
- MIT license, contribution guide, code of conduct, security policy.
- Architecture decision records ADR-000 through ADR-011.
- Documentation skeleton.
- Verification script (`scripts/verify.ps1`) as the single definition-of-done gate.
- GitHub Actions: CI (Windows + Linux), CodeQL, gated release workflow, Dependabot.
- Development log and roadmap for AI-assisted incremental development.
```

---

## Exit criteria

- [ ] `scripts/verify.ps1` exits `0` on Windows **and** Linux.
- [ ] CI is green on a real pull request, both matrix legs.
- [ ] `dotnet build`, `dotnet test`, `dotnet pack`, `dotnet format --verify-no-changes` all work
      against `LiveSharp.slnx` — or the `.sln` fallback is taken and documented.
- [ ] `README.md` is UTF-8 without BOM and claims no implemented features.
- [ ] Twelve ADRs exist, each with a populated **negative** Consequences section.
- [ ] No file under `docs/` is empty.
- [ ] `AGENTS.md` alone is sufficient for a fresh agent to start correctly.
- [ ] `DEVELOPMENT_LOG.md` and `ROADMAP.md` reflect Phase 0 complete, Phase 1 next.
- [ ] `git tag v0.1.0-alpha.1` produces a MinVer version of `0.1.0-alpha.1`.
- [ ] The `release.yml` publish job is guarded and cannot fire.

## Verify

```powershell
pwsh scripts/verify.ps1 -SkipJs -SkipE2E
dotnet build LiveSharp.slnx -c Release -warnaserror
Format-Hex -Path README.md -Count 4     # must be 23 20 4C 69, not FF FE
git tag v0.1.0-alpha.1; dotnet msbuild -getProperty:Version
```

## Next

`docs/plan/phase-01-abstractions.md`, task `P01.T1`.
