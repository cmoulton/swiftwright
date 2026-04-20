# Swiftwright — Design Spec

**Date:** 2026-04-17
**Status:** Approved for implementation planning
**Owner:** Christina Moulton

## Summary

Swiftwright is a Claude Code plugin that packages opinionated Swift 6 / iOS 18+ development practices into reusable skills, commands, templates, and MCP configurations. It enables an agentic workflow for native iOS app development: the agent has durable context about architecture, concurrency, dependency injection, and testing, and can build, run, and validate its own changes.

The plugin is general-purpose across iOS projects. The typing iPad app is the first validation target but does not drive design decisions.

## Goals

1. Agent-first iOS development. Sessions start with useful context instead of re-deriving conventions.
2. Centralized, versioned updates: improvements to the toolkit propagate to all projects on `/reload-plugins`.
3. Stable project outputs: files copied into a project during bootstrap stay put; the project doesn't drift when the toolkit moves.
4. Strong defaults for concurrency, testability, and long-term maintenance — the three areas the user flagged as highest-stakes.
5. Descriptive architecture guidance — SwiftUI is fixed, MVVM is the leaning default, but alternatives are documented and selectable.
6. Feedback loop: lessons learned during real project work can be reflected back into the toolkit via a dedicated skill.

## Non-goals

- Server-side Swift, macOS-only apps, or cross-platform frameworks.
- Replacing Xcode for project authoring. The plugin uses Xcode's tooling (xcodebuild, xcodegen) but does not reimplement it.
- Forking every upstream skill. The plugin curates; it does not own the world.
- Supporting Swift versions prior to 6, or iOS below 18. Conventions assume strict concurrency.

## Known risks and mitigations

Three areas where AI-assisted iOS development historically underperforms. Swiftwright must counter each deliberately — they are not "known issues to live with," they are first-class design constraints.

### Risk 1: AI-written SwiftUI tends to be slow

SwiftUI performance failure modes AI commonly produces: excessive view re-evaluation, overly coarse `@Observable` granularity, expensive work in `body`, missing `Equatable` on view inputs, incorrect `List`/`ForEach` usage, computed properties in view hierarchies that trigger recomputation, unnecessary `@State` coupling. These don't fail tests; they ship as perceptible jank.

**Mitigation:**
- The `swift-architecture` skill includes a dedicated SwiftUI performance section: the common pitfalls above, each with the correct pattern and a rationale rooted in how SwiftUI's diffing actually works.
- The skill instructs agents to profile meaningful view hierarchies with Instruments (via XcodeBuildMCP) before declaring UI work complete — performance is a verification step, not a vibe.
- The `swift-testing-patterns` skill covers performance *assertions* where tractable: e.g., measuring view body invocation counts in tests, bounding allocation growth.
- Future evolution: a dedicated `swiftui-performance` skill if the section outgrows `swift-architecture`. Also: evaluating the upstream twostraws Performance skill during implementation and deciding embed-vs-reference.

### Risk 2: MVVM's async testability is weaker than TCA's

We chose SwiftUI + MVVM over TCA due to verbosity, but TCA's reducer model makes async flows deterministic and trivially testable. Plain MVVM with async view model methods is harder to test well: race conditions, hard-to-drive state transitions, flaky tests that cheat with sleeps.

**Mitigation:**
- The `swift-testing-patterns` skill treats async MVVM testing as a top-level topic, not a footnote. Covers: protocol-first boundaries for every external dependency, injected clocks for deterministic scheduling, state-machine modeling of view model flows, polling-for-state assertions (no arbitrary sleeps — enforced by SwiftLint rule), decoupling the work from the trigger so the work can be tested in isolation.
- The `swift-dependency-injection` skill requires that every side-effect boundary (network, persistence, notifications, analytics, etc.) be Knit-registered. View models take protocol-typed dependencies, never concrete types.
- The `swift-architecture` skill documents the "testability tax" explicitly: what MVVM gives up vs. TCA, and what patterns pay it back.
- Future evolution: if the tax proves too high on a real project, revisit TCA for parts of the app where async correctness dominates (e.g., complex forms, sync flows). The reflection skill captures this.

### Risk 3: AI + DI misses ambient state that should be injected

Things that look constant but aren't: current date/time, locale, time zone, calendar, currency, random sources, UUIDs, device region. AI-generated code routinely calls `Date()`, `Locale.current`, `Calendar.current`, `UUID()` directly in business logic. This compiles, passes tests that also use the real values, and ships bugs that surface only when a user travels, changes region, or a date-boundary edge case hits.

**Mitigation:**
- The `swift-dependency-injection` skill has a first-class "ambient state" section naming the recurring offenders. Pattern: a small `SystemContext` (or similar) injected via Knit containing `Clock`, `Locale`, `TimeZone`, `Calendar`, `CurrencyCode`, `RandomNumberGenerator`, `UUIDProvider`. Business logic never references the global static accessors; agents are told so explicitly.
- The `swift-testing-patterns` skill includes patterns for exercising these: frozen clocks, test locales, non-Gregorian calendars, non-USD currencies. Tests must run under at least one non-default configuration for date/locale-sensitive logic.
- The zadr SwiftLint set is extended with custom rules banning direct use of `Date()`, `Locale.current`, `Calendar.current`, `TimeZone.current`, `UUID()` (standalone) in non-test targets. Violations steer the agent to the injected alternative.
- Future evolution: the ambient-state list grows with experience. Reflection skill captures newly-discovered offenders.

## Target stack

- **Swift 6** with strict concurrency checking enabled.
- **iOS 18+** minimum deployment target (iPadOS 18+ implied).
- **SwiftUI** as the view layer.
- **MVVM** as the default view/presenter pattern, with documented alternatives.
- **Knit** (Cash App) for dependency injection, built on Swinject with code-gen type safety.
- **Swift Testing** (Apple's new framework) for tests. No XCTest compatibility shims.
- **SwiftLint** with zadr's custom rule set as the enforced baseline.
- **Xcode 16.2+** (or whatever version supports the target Swift/iOS combo at install time).

## High-level architecture

Swiftwright has three layers:

1. **Authored content** (owned by Swiftwright): skills, templates, slash commands. This is the stable core.
2. **Referenced upstream content** (pinned to tags/commits): AvdLee's concurrency skill and a curated subset of `twostraws/Swift-Agent-Skills`.
3. **MCP servers** (external services wired up via config): Sosumi (Apple docs), XcodeBuildMCP (build/run/simulator).

Skills and templates are consumed differently:
- **Skills and commands** live in the plugin and update centrally.
- **Templates** (SwiftLint config, CLAUDE.md, xcodegen `project.yml`) are *copied* into each project during bootstrap. Projects do not auto-update from template changes — a deliberate stability choice. A later subcommand can offer to refresh templates on demand.

## Repository layout

```
swiftwright/
├── .claude-plugin/
│   ├── plugin.json              # Plugin manifest (name, version, entry points)
│   └── marketplace.json         # Optional, for later publishing
├── skills/
│   ├── swift-architecture/
│   │   └── SKILL.md             # SwiftUI + MVVM, layer model, performance, build-time budget
│   ├── swift-navigation/
│   │   └── SKILL.md             # NavigationStack + enum Route + Router pattern
│   ├── swift-dependency-injection/
│   │   └── SKILL.md             # Knit patterns, preview DI, ambient state, exit ramp
│   ├── swift-error-handling/
│   │   └── SKILL.md             # Domain vs infrastructure errors, toast UX, logger handoff
│   ├── swift-logging/
│   │   └── SKILL.md             # OSLog facade with future backend hook
│   ├── swift6-concurrency-conventions/
│   │   └── SKILL.md             # House rules complementing AvdLee's skill
│   ├── swift-testing-patterns/
│   │   └── SKILL.md             # Swift Testing, async MVVM testing, snapshot tests, UI test policy
│   ├── swift-project-bootstrap/
│   │   └── SKILL.md             # New-project scaffolding structure and conventions
│   ├── swift-project-retrofit/
│   │   └── SKILL.md             # Migrate existing projects to Swiftwright one concern at a time
│   ├── swift-persistence/
│   │   └── SKILL.md             # SwiftData vs Core Data vs GRDB vs SQLite vs CloudKit decision guide
│   └── swift-toolkit-reflection/
│       └── SKILL.md             # Analyze session work for toolkit-worthy patterns (with noise filter)
├── commands/
│   ├── swift-init.md            # Bootstrap a new project
│   ├── swift-retrofit.md        # Retrofit an existing project (one concern at a time)
│   ├── swift-skill-check.md     # Report which skills should be consulted for the current task
│   ├── swift-validate-skills.md # Skill staleness audit
│   └── toolkit-review.md        # Reflection command
├── templates/
│   ├── .swiftlint.yml           # zadr rules + Swiftwright additions
│   ├── CLAUDE.md                # Project-level agent context template (with skill routing table)
│   ├── project.yml              # xcodegen project template
│   ├── ci/
│   │   └── github-actions.yml   # CI workflow for private repos
│   ├── docs/
│   │   ├── architecture.md      # Pointer file loaded on demand
│   │   ├── navigation.md
│   │   └── testing.md
│   └── project-structure/       # Folder skeleton (App/, Features/, Core/, etc.)
├── mcp/
│   └── servers.json             # Sosumi + XcodeBuildMCP snippets for settings.json
├── references/
│   └── upstream-skills.md       # AvdLee + curated twostraws skills, with pinned refs
├── CHANGELOG.md
└── README.md
```

Skills use frontmatter fields `lastVerified: YYYY-MM-DD` and `targetSDK: iOS 18.x / Swift 6.x / Xcode 16.x` to support staleness tracking (see Maintenance section).

## Authored skills — content scope

Each skill is a `SKILL.md` with a tight `description` frontmatter field so Claude selects it accurately. Supporting files go alongside for deeper reference where useful.

### swift-architecture

Primary pattern: SwiftUI + MVVM, using `@Observable` for view models.
Documents alternatives:
- TCA (The Composable Architecture) — when it fits, when it doesn't. Explicit note on TCA's superior async testability as a factor to weigh.
- Vanilla SwiftUI with view-local `@State` + environment — for small apps.
- Coordinator pattern for complex navigation — when screen count and flow branching justify it.

For each pattern: when to pick, when to avoid, example file layout, interaction with DI, testability implications.

**Explicit layer model (prescribed, not optional):**
Prevents fat view models and inconsistent business-logic placement.

- **View** — SwiftUI view. No logic beyond layout and binding. Takes a view model via init.
- **ViewModel** — presentation adapter. Exposes state for the view (`@Observable`), forwards user intents to Services. Contains no business rules, no data fetching logic, no validation rules — only translation between view state and service calls. Takes services and ambient context via init (resolved by Knit).
- **Service** — use-case orchestration. Coordinates domain logic and infrastructure. Returns Domain types (or DomainError). Owns "what the app does." Protocol-typed so view models and tests consume an abstraction.
- **Domain** — pure Swift types and rules. No frameworks, no I/O, no Foundation dependencies beyond what's unavoidable. Value types. Fully testable in isolation.
- **Infrastructure** — network, persistence, platform APIs. Hidden behind protocols in the Service layer. Knit-registered.

A file's layer is visible from its folder (`Features/<Feature>/View`, `ViewModel`; `Core/Services`, `Core/Domain`, `Core/Infrastructure`). Cross-layer imports flow one direction: View → ViewModel → Service → Domain; Infrastructure implements Service-owned protocols.

**Default package strategy: one target.** Do not split into Swift Packages until measured cause exists (build-time pressure, reuse across apps, clear ownership boundary). Premature modularization is a productivity drag for solo developers.

**Build-time budget:**
- Target: clean build ≤ 60s, incremental build ≤ 5s on the developer's machine.
- When exceeded: `/swift-init` and `/toolkit-review` surface a warning. Common culprits: excessive Knit Assembly size, SwiftLint rule set bloat, `@Observable` overuse in hot paths, premature package splitting.
- Agents should notify the user when their changes push incremental build over budget.

**SwiftUI performance section (first-class, not a footnote):**
- Common AI-produced failure modes: expensive work in `body`, missing `Equatable` on view inputs, coarse `@Observable` granularity, computed properties that trigger recomputation, misuse of `List`/`ForEach`, unnecessary `@State` at the wrong level.
- The correct pattern for each, with rationale rooted in SwiftUI's diffing model.
- Profiling requirements: meaningful view hierarchies must be profiled (Instruments via command-line `xctrace`, or `SwiftUI` debug tools) before declaring UI work complete. Performance is a verification step.
- Testable performance assertions where tractable: body invocation counts, allocation bounds.

Explicitly calls out the MVVM async testability tax and links to the testing skill for how to pay it.

Also covers: project folder structure (`App/`, `Features/`, `Core/`, `Services/`), module boundaries, the (rare) conditions that justify splitting into Swift packages.

### swift-navigation

SwiftUI navigation has been the most volatile part of the framework across iOS versions. This skill prescribes a single coherent pattern for iOS 18+.

**Core pattern:**
- Per-feature (or app-wide, for small apps) `Route` enum listing every destination the stack can contain.
- `NavigationStack(path: $router.path)` at the feature root, with `.navigationDestination(for: Route.self) { route in ... }` handling all destinations.
- A `Router` / `Navigator` `@Observable` class holds the `path: [Route]` as single source of truth. Injected into view models via Knit.
- View models call `router.push(.someDestination)`, `router.pop()`, `router.popToRoot()`. Views never manipulate navigation state directly.

**Modals:**
- Sheets: `presentedSheet: SheetDestination?` on the router, driven by `.sheet(item:)`.
- Full-screen covers: analogous `presentedFullScreen: FullScreenDestination?`.
- Same single-source-of-truth rule.

**Deep linking:**
- URL → parser → `[Route]` → `router.path = routes`.
- The parser is a pure function, trivially testable.
- iPad universal links, custom URL schemes handled identically.

**iPad specifics:**
- `NavigationSplitView` for list-detail patterns on large-screen sizes; same `Route` model applies, with a sidebar selection driving detail.
- Multitasking / split view: no special-casing needed beyond standard SwiftUI responsiveness.

**Anti-patterns documented:**
- `NavigationLink(destination:)` without path binding (loses state, no programmatic nav).
- Sheet state scattered across multiple `@State` properties.
- Nested `NavigationStack`s without clear boundary rationale.
- Deep linking via `onOpenURL` scattered across views instead of routed through the router.

### swift-dependency-injection

Knit-first. Covers:
- Assembly file structure and naming
- `@knit` annotation usage
- Generated type-safe resolvers vs. string-based Swinject
- Duplicate registration detection (compile-time and runtime)
- Integration with SwiftUI: view models take dependencies via init (not `@Environment`); parent resolves from the Knit container
- Testability framing: every boundary worth mocking goes through Knit
- Test assemblies: separate configurations for unit, integration, UI tests

**Preview DI pattern (first-class section — previews are a solo-dev productivity gate):**
- Every project ships a `PreviewAssembly: Knit.Assembly` that registers lightweight in-memory stubs for all services. Loads fast, no async setup, no network.
- Every view model with dependencies ships a static `.preview()` factory that resolves from the preview container. Convention: if a view model has dependencies, it has `.preview()`.
- `#Preview` blocks use the factory: `#Preview { SomeView(viewModel: .preview()) }`.
- Previews that need specific state (loading, error, populated) use factory variants: `.preview(state: .loading)`.
- Rule: views must never fail to preview. If a view can't preview, its dependency shape is wrong (likely concrete types leaking in).

**Knit exit ramp:**
Knit is Cash App scale. If it becomes a drag (build-time, debugging complexity, maintenance), migrate out. The skill documents the path:

- Service access is already protocol-typed, so callers don't know or care about Knit.
- Migration target: either **Factory** (similar ergonomics, no codegen) or plain init-based DI with a small app-scoped registry.
- Steps: replace Knit Assemblies with equivalent Factory registrations (or constructor-chains); delete codegen build phase; update bootstrap skill. Migration is a swap at the composition root, not a rewrite of consumers.
- A deprecation signal for Knit (e.g., Cash App stops maintaining it) triggers an open issue in Swiftwright to revisit the default.

**Ambient state injection (first-class section):**
The recurring AI failure mode of calling global statics directly in business logic is called out by name, with required alternatives.

Required-to-inject list (non-test code must not reference the global form):
- `Date()` → injected `DateProvider` (protocol returning current `Date`)
- Scheduling / delays → Swift's `Clock` protocol (`ContinuousClock` / `SuspendingClock` in prod, custom test clock in tests) — not `Task.sleep(nanoseconds:)` with the default clock
- `Locale.current` → injected `Locale`
- `TimeZone.current` → injected `TimeZone`
- `Calendar.current` → injected `Calendar`
- `UUID()` standalone → injected `UUIDProvider`
- Currency, region, random sources — same pattern

Pattern: a `SystemContext` (or similar) struct/protocol registered in Knit's main assembly. View models and services take it (or specific pieces of it) via constructor injection. Test and preview assemblies provide frozen/controlled variants — a `DateProvider` fixed at a known instant, a custom `Clock` that lets tests advance time deterministically, non-default `Locale`/`Calendar`, etc.

Custom SwiftLint rules (beyond the zadr set) enforce this: direct use of `Date()`, `Locale.current`, `Calendar.current`, `TimeZone.current`, `UUID()` in non-test targets is banned at error severity.

The ambient-state list is expected to grow; the `swift-toolkit-reflection` skill surfaces new offenders.

### swift-error-handling

Prescribes a single consistent error model across layers, wired into UI and logging.

**Error type hierarchy:**
- **DomainError** — expected, recoverable failures originating from business rules (validation, user-visible conditions). Conforms to `LocalizedError`. Each Service defines its own domain errors.
- **InfrastructureError** — unexpected failures from the outside world (network, persistence, platform APIs). Wraps the underlying error for logging, exposes a generic user-visible surface.
- No generic `AppError` bucket. Every thrown error has a specific, named type.

**Surfacing model:**
- **Toasts** are the default customer-facing surface for recoverable errors — non-modal, dismissible, accessible, auto-expire with longer duration for errors than for info.
- A `ToastCenter` (or equivalent) `@Observable` singleton is Knit-registered and injected wherever errors cross into view models.
- Inline validation errors stay inline; toasts are for runtime failures (save failed, network unavailable, etc.).
- Modals (alerts, sheets) only for blocking confirmations or destructive-action undo flows, not for routine errors.

**Propagation rules:**
- Services throw typed errors.
- View models catch, translate to toast/user-facing message, and dispatch.
- View models never rethrow infrastructure errors to the view layer.
- All caught errors flow through the logger before the user sees them.

**Logging handoff:**
- Every catch site calls `logger.error(...)` with the original error attached.
- Toasts display user-friendly messages; full error details go only to the logger.

### swift-logging

Vanilla OSLog, wrapped in a thin facade to keep the door open for a future backend service (Bugsnag, Datadog, Sentry).

**Facade:**
```swift
protocol Logger {
    func debug(_ message: String, metadata: [String: String]?)
    func info(_ message: String, metadata: [String: String]?)
    func warning(_ message: String, metadata: [String: String]?)
    func error(_ message: String, error: Error?, metadata: [String: String]?)
}
```

- Knit-registered. Every caller takes `Logger` by protocol.
- Default implementation wraps `os.Logger` (OSLog), one instance per subsystem/category.
- Subsystem convention: reverse-DNS bundle id. Category convention: logical concern (e.g., `"Network"`, `"Persistence"`, `"Feature.Onboarding"`).
- Privacy: OSLog's `OSLogPrivacy` markers used consistently; skill documents when to mark `.private` vs `.public`.

**Future backend swap:**
- To add Bugsnag/Datadog/Sentry, implement a new `Logger` conformance that forwards to the backend SDK (keeping OSLog calls too, since local logs remain useful).
- Swap the Knit registration to the composite logger.
- No caller code changes. This is the whole reason for the facade.

**Test and preview loggers:**
- `RecordingLogger` captures calls for test assertions.
- `NullLogger` for previews — no console noise during preview rendering.

### swift6-concurrency-conventions

Complements (does not duplicate) AvdLee's concurrency skill. Focus is on the user's house rules.

Covers:
- The six zadr SwiftLint rules and rationale: no DispatchQueue, no `@unchecked Sendable`, no deprecated locks, no arbitrary test sleeps, no `#expect(true/false)`, no `DispatchQueue.main.async`
- MainActor discipline in SwiftUI — what to mark, when default actor inference is sufficient
- Sendable-first data modeling
- Actor reentrancy pitfalls
- Replacements for banned patterns (e.g., `Task { @MainActor in }` instead of `DispatchQueue.main.async`)

### swift-testing-patterns

- `@Test`, `@Suite`, parameterized tests, traits
- `#expect` vs. `Issue.record()`
- DI-driven test setup with Knit test assemblies
- Concurrency testing: polling for state instead of arbitrary sleeps
- Decision guide: unit vs. integration vs. UI tests vs. snapshot tests
- Running tests from the agent via XcodeBuildMCP

**Async MVVM testing (first-class section — pays the MVVM-vs-TCA testability tax):**
- Protocol-first boundaries for every external dependency so view models can be tested in isolation.
- Injected clocks and schedulers for deterministic timing — never real time, never `Task.sleep`.
- State-machine modeling of view model flows: enumerate states, assert transitions, drive inputs.
- Polling-for-state assertions with bounded timeouts (the zadr `no_arbitrary_sleeps` rule applies here).
- Decoupling triggers from work: make the async work a separately-testable function, not an inline closure in a button handler.
- Cancellation and reentrancy testing: ensure a view model method behaves correctly when invoked twice concurrently.
- Testing against the ambient-state injections from the DI skill: frozen clocks, non-default locales, non-Gregorian calendars, non-USD currencies. Date/locale-sensitive logic must have at least one test under a non-default configuration.

**Snapshot testing (first-class — SwiftUI regression guard):**
- Uses `swift-snapshot-testing` (Point-Free) or equivalent Swift-Testing-compatible lib. During implementation, verify a viable Swift Testing integration exists; if not, author a thin wrapper.
- Coverage policy — prescribed test counts per screen type:
  - **Simple screens** (single state): 1 snapshot (default state).
  - **Screens with state machines** (loading/loaded/error/empty): one snapshot per state.
  - **Screens with variable content**: one snapshot for short content + one for long/overflow content.
  - **Dark mode**: all snapshots run in both light and dark for screens with custom styling.
  - **Dynamic Type**: at least one snapshot at `accessibilityLarge` for any screen with text.
  - **iPad-specific layouts**: separate snapshots at iPad size classes when layout differs from iPhone.
- Snapshots live alongside their test file. Reference images under `Tests/__Snapshots__/`. Regenerating snapshots requires explicit test env var to prevent accidents.
- Snapshot diffs are reviewed as part of PR/commit review — not auto-accepted.

**UI test policy (minimal, high-leverage):**
- Scope: 3–5 UI tests per app covering the critical customer journeys only (e.g., for the typing app: launch → pick lesson → complete lesson → see results). Not per-screen coverage.
- Each UI test asserts the story completes and a key end-state element is visible; no pixel-level assertions.
- UI tests run against in-memory stubs (via a launch-argument switch that picks a UI-test Knit assembly). No real network, no persisted data.
- Run on every PR/local pre-commit, but allowed to be slower than unit tests.
- Flaky UI tests get quarantined within one session; the skill documents the quarantine-then-fix workflow so they don't accumulate.

**Performance assertions (where tractable):**
- Body invocation counts for critical views.
- Allocation bounds for hot paths.
- Cross-reference to the `swift-architecture` performance section for when to write these tests.

**Swift Testing ecosystem gaps:**
- Some libraries still ship XCTest-only. Skill lists known gaps (snapshot libs, UI test helpers) and Swiftwright's position: author thin wrappers where needed, accept small dev-time tax.

### swift-project-bootstrap

Documents the structure `/swift-init` produces. Stands alone so it can be read when onboarding an existing project or when a bootstrap step needs to be applied manually.

Covers:
- Target layout, folder structure (matches the layer model from `swift-architecture`)
- SwiftLint build phase integration
- Knit codegen build phase
- Logger facade registration in the root Assembly
- Router registration in the root Assembly
- CLAUDE.md conventions with progressive disclosure (see below)
- `project.yml` (xcodegen) structure and regeneration
- GitHub Actions CI workflow template (works with private repos on free plans within the included minutes budget)
- Build-time budget and diagnostics

**CLAUDE.md progressive disclosure convention:**
The project's root `CLAUDE.md` contains only what's always needed:
- Target stack pinning (iOS version, Swift version, Xcode version, Swiftwright version used at bootstrap)
- Skill routing table: "for navigation work → see `swift-navigation`; for DI work → see `swift-dependency-injection`; etc."
- Pointer list to on-demand docs in `docs/`:
  - `docs/architecture.md` — detailed layer model and folder structure
  - `docs/navigation.md` — project-specific navigation conventions
  - `docs/testing.md` — project-specific testing conventions
  - `docs/errors.md` — project-specific error catalogue
- Bootstrap decision record (short) — which options were chosen from the interactive flow.
- Rule: root CLAUDE.md stays under ~150 lines. Overflow goes to `docs/`.

**CI template (GitHub Actions):**
- Runs on PR and pushes to main.
- Steps: checkout → resolve SPM deps → xcodegen generate → xcodebuild test → SwiftLint strict → upload snapshot diffs as artifact on failure.
- Works with private repos within GitHub Free's 2000 minute/month allotment for typical solo workloads.
- Template references `xcodebuild` directly (not XcodeBuildMCP; MCP is for the agent, CI is for GitHub Actions).

### swift-project-retrofit

For bringing an existing iOS project under Swiftwright, one concern at a time. Explicit about what can be applied independently.

**Ordered menu of retrofit concerns** (each can be applied alone):
1. **Logging** — introduce `Logger` protocol + OSLog impl, swap existing `print`/`NSLog` call sites. Lowest risk, highest immediate value for debugging.
2. **Ambient state injection** — add `SystemContext`, replace `Date()`, `Locale.current`, etc. with injected variants. Independent of DI framework.
3. **SwiftLint rules** — drop in the Swiftwright SwiftLint config; fix violations incrementally.
4. **Error handling + toasts** — introduce `ToastCenter` and typed errors. Can run in parallel with existing patterns during migration.
5. **DI (Knit)** — stand up Knit Assemblies, route new code through them, migrate existing services as touched.
6. **Navigation** — convert to `NavigationStack` + `Route` enum + `Router`. Usually a per-feature migration.
7. **Testing conventions** — introduce Swift Testing alongside existing XCTest (they coexist). New tests go in Swift Testing; migrate on touch.
8. **SwiftUI performance + layer model** — most invasive. Refactor a feature at a time.

**Per-concern workflow:**
- Agent reads the relevant skill(s).
- Scopes to a single file, feature, or module.
- Verifies build + tests after each step.
- Records progress in a `swiftwright-retrofit.md` file at project root — tracks which concerns are applied, to what scope.

**Anti-patterns explicitly banned during retrofit:**
- Big-bang rewrites.
- Mixing multiple concerns in one PR/commit.
- Leaving a feature half-migrated across sessions.

### swift-persistence

Decision guide, not a how-to. Covers:
- SwiftData: current known issues (migration bugs, sync instability, performance cliffs), version-specific caveats — agent should not default to it
- Core Data: still the most stable choice for complex local storage
- GRDB: direct SQLite with Swift ergonomics — best for predictable performance
- Raw SQLite: when you need full control
- CloudKit: when sync is required, including CloudKit-only and CloudKit-backed options
- How each integrates with SwiftUI, Knit, and Swift 6 concurrency

### swift-toolkit-reflection

Triggered when user asks to "check if there's anything for the toolkit" or similar.

Analyzes the current session's work for:
- Recurring patterns applied multiple times within the session (candidate for a new skill entry)
- Surprises or gotchas encountered (candidate for concurrency conventions or a new skill)
- Decisions made against the default (candidate for architecture skill alternatives section)
- Template drift: CLAUDE.md or .swiftlint.yml changes that should propagate upstream
- Upstream skill gaps where AvdLee/twostraws content was insufficient
- **Skill staleness flags**: cases where user corrected agent advice that came from an existing skill — surfaces the skill for re-review

**Noise filter (mandatory):**
A proposal only reaches the output if it passes both:
1. **Generalizability test** — would this apply to a second, distinct iOS project, not just this one? If not, it's project-specific and belongs in the project's own docs, not Swiftwright.
2. **Frequency or severity gate** — either the pattern recurred 3+ times in the session/across projects, OR it caused a concrete bug / wasted time on something the toolkit should have warned about.

Proposals that don't clear the filter are dropped silently to avoid noise.

**Output format:**
Each proposal includes: the observation, which skill/template it updates (or "new skill: X"), the specific change proposed, why it clears the noise filter, and an explicit "[ ] approve [ ] reject" marker.

**Human approval required for every generalization:**
No silent edits. Each proposal is rejected by default unless the user explicitly approves. On approval, agent edits the Swiftwright repo and updates the skill's `lastVerified` date. Rejected proposals are logged (not deleted) in a `docs/rejected-reflections.md` file so patterns that recur can still surface later.

## Referenced upstream skills

Pinned in `references/upstream-skills.md` to specific commits or tags. Not forked.

**Included:**
- **AvdLee/Swift-Concurrency-Agent-Skill** — the broad educational surface for async/await, actors, Sendable, Swift 6 migration. Complemented by our own `swift6-concurrency-conventions`.
- **Curated `twostraws/Swift-Agent-Skills` subset** — to be selected during implementation, filtered by: active maintenance, Swift 6 / iOS 18 compatibility, non-overlap with authored skills, alignment with architecture and DI choices. Likely candidates: Accessibility, Performance (SwiftUI), iOS Simulator tooling, App Store Connect CLI. Explicitly excluded: SwiftData skill (pending user assessment of SwiftData stability).

**Update policy:** pins are bumped intentionally. User reviews the diff before updating.

## MCP servers

Configured in `mcp/servers.json` as snippets. The `/swift-init` command emits these for the user to add to their Claude Code `settings.json` (or a project-local `.mcp.json`).

- **Sosumi** — Apple docs access. Required.
- **XcodeBuildMCP** — build, test, simulator control, log inspection. Required; the agent validates its own changes through this.

Explicitly excluded: `claude-code-lsp-enforcement-kit` — designed for TypeScript/JS; SourceKit-LSP integration would be non-trivial and unproven. Reconsider if navigation becomes a measured problem.

## /swift-init slash command

Located at `commands/swift-init.md`. Thin orchestrator — most logic lives in the `swift-project-bootstrap` skill.

### Flow

1. **Detect context.** Check for `.xcodeproj`, `Package.swift`, `CLAUDE.md`. Classify as "new project" vs. "retrofit existing."

2. **Verify Xcode.** Run `xcodebuild -version` and `xcode-select -p`. Check against minimum version declared by the toolkit. If mismatched or missing, print clear remediation (point to `xcodes` CLI or direct download). Do not silently proceed.

3. **Ask scope questions interactively**, one at a time:
   - iOS minimum target (default 18)
   - SwiftUI + MVVM default, or alternative architecture?
   - Knit for DI (default yes)
   - Persistence choice — defers to `swift-persistence` skill for decision support
   - Networking required? If yes: URLSession + async/await with a small protocol-based client, or a dedicated layer?
   - CloudKit sync required?

4. **Invoke `swift-project-bootstrap` skill** for structural work.

5. **Create the Xcode project.** Primary path: xcodegen with a copied, parameterized `project.yml`. Fallback: `xcodebuild` with a template `.xcodeproj` if xcodegen is unavailable. User should not need to open Xcode for bootstrap.

6. **Copy templates** into the project, parameterized by scope answers:
   - `.swiftlint.yml`
   - `CLAUDE.md` — including recorded bootstrap choices
   - Knit Assembly starter
   - Folder skeleton

7. **Emit MCP configuration.** Print the Sosumi + XcodeBuildMCP JSON and ask the user to add to `settings.json` or confirm it's already present.

8. **Verify.** Run `xcodebuild` via XcodeBuildMCP to confirm the scaffold compiles. Run SwiftLint to confirm config is valid.

9. **Summarize.** Created files, active skills, next steps.

### What /swift-init does not do

- Modify existing Xcode project files without explicit retrofit mode.
- Auto-update templates in previously bootstrapped projects.

## /toolkit-review slash command

Thin wrapper over `swift-toolkit-reflection`. Invokes the skill against the current session and project.

Outputs a proposal list. No writes without user approval per item.

## /swift-retrofit slash command

Located at `commands/swift-retrofit.md`. Operationalizes the `swift-project-retrofit` skill for existing iOS projects.

### Flow
1. **Detect context.** Confirm `.xcodeproj` / `Package.swift` exists; read bundle id, target iOS, Swift version; read any existing `CLAUDE.md` or `swiftwright-retrofit.md`.
2. **Present concern menu.** Show the ordered list of retrofit concerns (logging → ambient state → SwiftLint → errors → DI → navigation → testing → layer model) with each marked "not started", "in progress", or "applied" based on the retrofit record.
3. **Pick one concern** (user choice). Command refuses to start a second concern if one is in progress.
4. **Invoke the skill** for that concern with project-specific parameters.
5. **Scope**: default to a single feature / module / file set. User can expand.
6. **Verify** — build + tests + lint pass — before marking the concern applied.
7. **Update retrofit record.** Write to `swiftwright-retrofit.md` with what was done, remaining scope, date.

### What /swift-retrofit does not do
- Run multiple concerns in parallel.
- Bulk-migrate the whole project in one invocation.
- Modify files outside the declared scope.

## /swift-skill-check slash command

Located at `commands/swift-skill-check.md`. Given a task description (or the current session context), reports which Swiftwright skills should be consulted and why. Forces an explicit skill-selection check when the user isn't sure, and helps verify skill descriptions are discriminating enough.

Output: ranked list of relevant skills with one-line rationale each.

## /swift-validate-skills slash command

Located at `commands/swift-validate-skills.md`. Skill staleness audit.

### Flow
1. Read every skill's frontmatter (`lastVerified`, `targetSDK`).
2. Flag skills older than a threshold (default 90 days) or targeting an obsolete SDK.
3. For each flagged skill, offer options:
   - **Quick re-verify**: user attests the skill is still correct; `lastVerified` bumps.
   - **Deep re-verify**: agent reads the skill, cross-checks claims against current Apple docs via Sosumi MCP, surfaces divergences for user review, applies updates with user approval, bumps `lastVerified`.
   - **Skip**: defer; no change.
4. Cross-reference the staleness-flag queue from `swift-toolkit-reflection`: skills where user recently corrected agent advice jump to the front regardless of date.

## Maintenance & update flow

### Editing Swiftwright

1. During real project work, identify a toolkit-worthy pattern.
2. Ask the agent to update Swiftwright. Agent edits the appropriate skill or template in the Swiftwright repo (cloned at `~/Developer/swiftwright/`).
3. Commit and push.
4. `/reload-plugins` in any project picks up the new version.

### Version discipline

- Semver-ish tags on the Swiftwright repo.
- Breaking changes get a minor bump, with migration notes in `CHANGELOG.md`.
- Project-level `CLAUDE.md` records the Swiftwright version used at bootstrap — supports reproducibility for older projects.

### Upstream skill updates

- Check pinned refs periodically (or when a new upstream release is announced).
- Review diffs before bumping pins.

### Template refresh

- Templates are copied, not symlinked. Projects are stable by default.
- Future subcommand (post-v1): `/swift-init refresh-templates` to opt into pulling updated templates into an existing project.

### Skill staleness tracking

- Every skill's frontmatter records `lastVerified: YYYY-MM-DD` and `targetSDK: iOS 18.x, Swift 6.x, Xcode 16.x`.
- `/swift-validate-skills` audits these on demand. Default threshold 90 days; tunable per skill (e.g., concurrency conventions may age faster than architecture).
- A skill is considered verified when: (a) user attests, or (b) agent completed a deep cross-check against current Apple docs via Sosumi and user approved resulting updates.
- Reflection skill surfaces suspected-stale skills when agent advice is corrected by the user. Those skills jump to the front of the audit queue.
- Moving to a new major iOS / Xcode version triggers a full audit pass.

### Structural rot check

- Every few projects, review whether skills are being invoked appropriately, `/swift-init` matches current practice, templates are current.
- Findings feed back into the toolkit — this is what `swift-toolkit-reflection` routinizes.

## Error handling & verification

- `/swift-init` verifies Xcode version before doing anything destructive. Aborts with clear guidance if unsupported.
- Scaffold is validated by `xcodebuild` post-creation. If the build fails, the command reports the failure and does not claim success.
- SwiftLint config is validated on a scaffold file.
- MCP configuration is presented for user action, not silently written to `settings.json`.
- `/swift-retrofit` refuses to start a new concern while another is in progress; refuses to modify files outside declared scope.
- `swift-toolkit-reflection` never edits Swiftwright silently. All changes are proposals requiring user approval.
- `/swift-validate-skills` updates skill frontmatter only after user attestation or explicit approval of agent-proposed changes.
- Build time: agents warn the user when their changes push incremental build above the declared budget.

## Open questions

### Structure and scope

- Exact curated list of `twostraws/Swift-Agent-Skills` — determined during implementation after audit.
- Whether `swift-persistence` should be a standalone skill or a section within `swift-architecture`. Current plan: standalone, because it's often the biggest risk/uncertainty area in a new iOS project.
- Whether to add a dedicated `swiftui-performance` skill in v1, or keep the performance content inside `swift-architecture`. Leaning: start inside architecture; split out if the section outgrows the skill or if a real project surfaces enough performance patterns to justify a dedicated surface.
- Whether to maintain a dedicated `swift-mvvm-async-testing` skill, or keep that content inside `swift-testing-patterns`. Leaning: keep in testing for v1; split if the testing skill balloons.
- Extent of custom SwiftLint rules beyond zadr's set (ambient-state bans). Baseline set defined during implementation; list grows via reflection.
- Snapshot testing library choice: Point-Free's `swift-snapshot-testing` is the default candidate, but its Swift Testing support may require a wrapper. Decide during implementation; defer to Point-Free's current state.
- Whether `ToastCenter` belongs in the error-handling skill's prescribed code or in a dedicated `swift-ui-components` skill for recurring custom UI primitives. Leaning: start in error-handling; extract if more prescribed primitives accumulate.

### Licensing and public release

- **Swiftwright's own license** — MIT vs. Apache 2.0. Leaning MIT (community default for tooling, minimal friction for contributors). Apache 2.0 adds an explicit patent grant, which matters for larger corporate-adjacent projects but is likely overkill here. Decide before first public tag.
- **zadr SwiftLint rules ownership** — three options: (a) ask zadr to add a license to their dotfiles repo and attribute, (b) attribute with assumed-permissive intent (legally ambiguous — dotfiles without explicit licenses are not clearly open-source), (c) rewrite the rules independently in Swiftwright's own words and rationale. Recommended path: ask zadr for clarity, *and* rewrite so Swiftwright owns its SwiftLint config outright regardless of their response. Blocks public release.
- **Timing of public release** — publish from day 1 as a private-but-visible repo? Wait until the typing app validates the toolkit? Wait until v1.0 with written articles? Each trigger has tradeoffs around polish vs. momentum. Decide when the first real project reaches a meaningful state.
- **Upstream fork policy** — current design references upstream skills only (no forking). If an upstream skill becomes unmaintained or diverges from Swiftwright's conventions, what's the fork-and-own threshold? Decide case-by-case; document the decision in the reflection skill's patterns.
- **Article/blog content** — okay to describe Swiftwright's design, share snippets of authored skills, and reference upstream work. Not okay to republish substantial upstream content without compliance. If writing about the toolkit becomes regular, consider a dedicated `writing-about-swiftwright.md` doc codifying what's safe to share.

## Pre-publication checklist

Before making the Swiftwright repo public or writing about it externally:

- [ ] Add `LICENSE` to the repo (MIT recommended for tooling; Apache 2.0 if explicit patent grant matters).
- [ ] Resolve zadr's SwiftLint rules IP status: ask zadr to add a license, attribute with permission, or rewrite rules independently with original rationale. Preferred: ask then rewrite so Swiftwright owns its SwiftLint config outright.
- [ ] Verify licenses on every embedded upstream artifact (none in v1 — all upstream content is referenced, not forked; re-verify if that changes).
- [ ] Add `CREDITS.md` or equivalent README section listing every upstream dependency (Knit, Sosumi, XcodeBuildMCP, AvdLee's skill, twostraws skills) with links and licenses.
- [ ] Confirm usage of "Claude Code" and Apple marks (Swift, iOS, iPadOS, Xcode) is descriptive/nominative only — no logos, no implied endorsement.
- [ ] Review article drafts before publishing to ensure no upstream content is republished without permission/compliance.

## Success criteria

Swiftwright v1 is successful when:

1. Running `/swift-init` in an empty directory produces a building, linting, testing iOS 18+ Swift 6 project with Knit, logger facade, Router, toast center, CI workflow, and SwiftUI previews all wired in — in under 5 minutes, without opening Xcode. Incremental builds from this scaffold are within budget (≤ 5s).
2. The agent consistently applies the concurrency conventions, DI patterns (including ambient-state injection), navigation pattern, error handling + logging pattern, and testing idioms without prompting. No agent-written code calls `Date()`, `Locale.current`, `DispatchQueue`, `print`/`NSLog`, etc. in non-test targets.
3. Agent-written SwiftUI passes a performance verification step (profile or body-invocation assertion) before UI work is declared complete, and previews work out of the box for every new view.
4. Agent-written async view model code is deterministically testable — no `Task.sleep`, injected clocks used consistently, tests run without flakes across repeated invocations.
5. Screens have snapshot coverage that matches the coverage policy (state machines covered, dark mode + Dynamic Type covered where specified).
6. Running `/swift-retrofit` on an existing project lets the user migrate one concern at a time without breaking the build or tests.
7. Running `/toolkit-review` after a work session produces proposals only when they clear the noise filter — no false positives from project-specific patterns.
8. Running `/swift-validate-skills` surfaces stale skills and walks the user through re-verification.
9. Updates to Swiftwright propagate to all projects via `/reload-plugins`, and existing projects remain stable (templates unchanged unless user opts into refresh).

## Next step

Invoke the `superpowers:writing-plans` skill to create the implementation plan based on this spec.
