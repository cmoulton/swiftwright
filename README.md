# Swiftwright

A Claude Code plugin that packages opinionated Swift 6 / iOS 18+ development practices into reusable skills, slash commands, project templates, and MCP server configurations — so agentic iOS work starts with durable context instead of re-deriving conventions every session.

## Status

**Pre-v1 — design complete, implementation in planning.**

The full design is in [`docs/superpowers/specs/2026-04-17-swiftwright-design.md`](docs/superpowers/specs/2026-04-17-swiftwright-design.md). No code yet. No public release yet.

## Why Swiftwright exists

Working on a native iOS app with an AI agent surfaces recurring failure modes:

- Agents default to `Date()`, `Locale.current`, `DispatchQueue.main.async` in business logic — code that compiles, passes naive tests, and ships bugs.
- AI-written SwiftUI tends to be slow: body work, coarse `@Observable`, missing `Equatable`, mishandled `List`/`ForEach`.
- MVVM's async testability is weaker than TCA's by default — without deliberate patterns, view model tests become flaky sleep-based brittle messes.
- Every new project re-negotiates the same architecture, DI, testing, and concurrency decisions.

Swiftwright answers all of these in one installable plugin that any iOS project can opt into.

## What's inside (planned)

### Skills

- **swift-architecture** — SwiftUI + MVVM default, layer model (View → ViewModel → Service → Domain → Infrastructure), SwiftUI performance pitfalls and their fixes, build-time budget.
- **swift-navigation** — `NavigationStack` + enum `Route` + `Router` pattern for iOS 18+.
- **swift-dependency-injection** — [Knit](https://github.com/cashapp/knit)-first DI, preview `.preview()` factory convention, ambient-state injection (`DateProvider`, `Clock`, `Locale`, `UUIDProvider`, etc.), documented exit ramp if Knit becomes a drag.
- **swift-error-handling** — typed `DomainError` / `InfrastructureError`, toast-based customer surface, logger handoff at every catch site.
- **swift-logging** — thin `Logger` protocol over OSLog, swap-in path for Bugsnag/Datadog/Sentry with no caller changes.
- **swift6-concurrency-conventions** — house rules complementing [AvdLee's Swift Concurrency skill](https://github.com/AvdLee/Swift-Concurrency-Agent-Skill).
- **swift-testing-patterns** — Swift Testing idioms, deterministic async MVVM testing, snapshot test coverage policy, minimal-but-high-leverage UI test policy.
- **swift-project-bootstrap** — what a well-formed Swiftwright project looks like.
- **swift-project-retrofit** — migrate existing projects one concern at a time (logging → ambient state → SwiftLint → errors → DI → navigation → testing → layer model).
- **swift-persistence** — decision guide across SwiftData, Core Data, GRDB, raw SQLite, CloudKit (SwiftData explicitly not the default due to stability history).
- **swift-toolkit-reflection** — analyzes real project work for toolkit-worthy generalizations, with a mandatory noise filter (generalizability + frequency/severity gates) and per-item approval.

### Slash commands

- **`/swift-init`** — bootstrap a new iOS project (xcodegen → templates → verify build → emit MCP config).
- **`/swift-retrofit`** — migrate an existing project, one concern per invocation.
- **`/swift-skill-check`** — report which Swiftwright skills should be consulted for the current task.
- **`/swift-validate-skills`** — staleness audit for skills; flags any past threshold or targeting an obsolete SDK.
- **`/toolkit-review`** — run the reflection skill against the current session.

### Templates (copied into each project)

- `.swiftlint.yml` — [zadr's strict rule set](https://github.com/zadr/dotfiles/blob/main/swiftlint.yml) plus Swiftwright's ambient-state bans (rewrite pending, see [licensing below](#licensing)).
- `CLAUDE.md` — progressive-disclosure template with a skill routing table.
- `project.yml` — xcodegen project template.
- `ci/github-actions.yml` — CI workflow that works within GitHub Free's minutes budget for private repos.
- Folder skeleton matching the layer model.

### MCP server configurations

- **[Sosumi](https://sosumi.ai)** — Apple developer documentation accessible to LLMs.
- **XcodeBuildMCP** — build, test, simulator control, log inspection. Lets the agent validate its own changes.

## Target stack

- Swift 6 with strict concurrency
- iOS 18+
- SwiftUI + MVVM (with alternatives documented)
- Knit for DI
- Swift Testing (no XCTest shims)
- SwiftLint
- Xcode 16.2+
- xcodegen for project generation

## Design principles

1. **Stability where it matters.** Authored skills, templates, and conventions are owned here and git-versioned. Upstream dependencies are referenced with pinned refs, not forked.
2. **Projects stay reproducible.** Templates are *copied* into projects during bootstrap, not symlinked. Updating Swiftwright does not silently change existing projects; refresh is opt-in.
3. **Agent-first but not agent-only.** Every skill and command stands alone as documentation a human can read and apply.
4. **Learn from real work.** The reflection skill feeds lessons from real projects back into Swiftwright, with a noise filter and explicit approval gate.

## Usage (planned flow)

```bash
# Install the plugin once, globally, from your Claude Code:
/plugin install cmoulton/swiftwright

# In a new project directory:
/swift-init
# Answers a handful of scope questions, creates an Xcode project,
# drops in templates, verifies the build, emits MCP config.

# In an existing project:
/swift-retrofit
# Pick one concern at a time; progress tracked in swiftwright-retrofit.md.

# During any session:
/swift-skill-check         # "Which skills apply to what I'm doing?"
/toolkit-review            # "Anything we learned that should go back into Swiftwright?"
/swift-validate-skills     # "Which skills are due for a freshness check?"
```

## Maintenance model

Swiftwright is maintained by [@cmoulton](https://github.com/cmoulton) as a personal toolkit. Updates happen as real iOS projects surface patterns worth generalizing. Every skill carries `lastVerified` and `targetSDK` frontmatter; `/swift-validate-skills` keeps those honest over time.

## Acknowledgements

Swiftwright relies on and references work from:

- [Antoine van der Lee](https://github.com/AvdLee) — Swift Concurrency Agent Skill
- [Paul Hudson & contributors](https://github.com/twostraws/Swift-Agent-Skills) — Swift Agent Skills curation
- [Cash App](https://github.com/cashapp/knit) — Knit dependency injection
- [zadr](https://github.com/zadr) — SwiftLint rule baseline
- [Sosumi](https://sosumi.ai) — AI-accessible Apple documentation
- [Point-Free](https://github.com/pointfreeco) — swift-snapshot-testing (anticipated)
- The Anthropic [Claude Code](https://claude.com/claude-code) and Superpowers plugin systems that make this kind of toolkit possible.

A full `CREDITS.md` with licenses ships with the first public release.

## Licensing

License pending. See [`docs/superpowers/specs/2026-04-17-swiftwright-design.md#licensing-and-public-release`](docs/superpowers/specs/2026-04-17-swiftwright-design.md#licensing-and-public-release) for open decisions, including:

- Swiftwright's own license (leaning MIT).
- Resolution of zadr SwiftLint rules IP status (likely rewrite so Swiftwright owns its config outright).
- Pre-publication checklist (LICENSE, CREDITS, trademark use review).

Until a LICENSE file is committed, treat the contents of this repository as all rights reserved.

## Roadmap

- **v1 (in planning)** — project init + modernization. Everything in the "What's inside" section above.
- **Phase 2 (deferred)** — agentic day-to-day feature development: `/swift-new-feature`, `/swift-bug-fix`, `/swift-release`, visual simulator feedback loops, specialized subagents, pre-commit self-review, and autonomous loops for bounded repetitive work. Designed after v1 has been used on a real project long enough to produce honest requirements. See the [Phase 2 section of the spec](docs/superpowers/specs/2026-04-17-swiftwright-design.md#phase-2-agentic-feature-development-future-work) for current direction.

## Related

- [typing](https://github.com/cmoulton/typing) — iPad typing app; the first real project Swiftwright is validating against.
