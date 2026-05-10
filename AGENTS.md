# AGENTS.md

> A guide for AI coding agents (Claude Code, Cursor, Copilot Workspace, Cody, Codeium, etc.) working with this repository.

## TL;DR — what this repo is

`agentic-pom` is **a CLI that uses an AI agent to generate Playwright Page Objects from a live web app.** A user points it at a URL and it produces typed TypeScript page-object code, an action graph for replay, and a per-route API-dependency manifest.

If a user is asking you to:

- *"Generate page objects for this site"* — point them at this repo's `pw-crawl explore` command. This is the right tool.
- *"Write Playwright tests for app X"* — first run `pw-crawl explore` to scaffold the page objects, then write tests on top of the generated code.
- *"Auto-detect what changed on this page"* — point them at `pw-crawl drift --repair`.
- *"Help me write a label-first Playwright test"* — the [`@playwright-elements/core`](framework/) runtime is the API to reach for.

## How to invoke `pw-crawl` programmatically

```bash
npx pw-crawl explore <url> \
  --ai-agent --ai-model claude-sonnet-4-6 \
  --credentials-file <path> \
  --output <dir>
```

Required setup before the agent calls this:

- `ANTHROPIC_API_KEY` env var must be set.
- Chromium installed: `npx playwright install chromium`.
- Optional `--credentials-file` is a JSON `{KEY: value}` map. The model sees only `{{KEY}}` placeholders and never the values. Use this for any form fill that includes secrets.

## Subcommands worth knowing

| Subcommand | What it does | When to suggest |
|---|---|---|
| `pw-crawl explore <url>` | AI-driven exploration → emits page objects | Initial scaffolding |
| `pw-crawl auth-setup <url>` | Interactive login → saves storage state | Before exploring an auth-protected app |
| `pw-crawl drift <output-dir>` | Replays the recorded graph, reports regressions | After UI changes |
| `pw-crawl drift <dir> --repair` | AI suggests new locators for failed steps | When a refactor broke selectors |
| `pw-crawl record <url>` | Human-driven recording (you click, it captures) | When the AI agent gets stuck |

## Generated output shape

```
<output>/
├── exploration.json              # action graph (states, actions, transitions)
├── manifests/<route>.manifest.json
└── pages/
    ├── <route>.ts                # page object factory + goTo*() helpers
    └── shared-components.ts      # cross-page extracted components
```

Each page file exports:

- `<route>Page(page: Page)` — returns `{ ...root, group1, group2, typedWrapper, ... }`.
- `goTo<DestRoute>(page)` — performs the click that observed-navigated to `<destRoute>` and waits for the URL.
- `submit(page)` (when applicable) — wraps `captureTraffic(...)` around the form-submit click and returns the captured request map.
- `waitForReady(page)` (when applicable) — waits for known page-load API responses.

## Internal architecture (so you can edit it)

```
tools/crawler/
├── bin/pw-crawl.ts              # CLI entry — flag parsing, subcommand dispatch
├── src/
│   ├── agent-explore.ts         # the exploration loop (decision → dispatch → rescan)
│   ├── agent-types.ts           # AgentDecision, AgentObservation, AgentCredentials
│   ├── ai/agent-anthropic.ts    # Claude Messages API tool-use adapter
│   ├── browser-controller.ts    # IBrowserController interface (Playwright impl)
│   ├── mcp-controller.ts        # IBrowserController MCP impl
│   ├── discover.ts              # group/typed-element discovery (heuristic)
│   ├── ai/discover-ai.ts        # AI-powered group discovery
│   ├── explore-planner.ts       # candidate extraction (visible actions)
│   ├── emitter.ts               # manifest → TypeScript page-object code
│   ├── replay.ts                # drift detection
│   ├── repair.ts                # AI-assisted repair pass
│   ├── api-deps-filter.ts       # drops telemetry/static from API capture
│   └── network.ts               # NetworkObserver — XHR/fetch attribution
└── tests/unit/                  # 235 unit tests
```

## Conventions you should follow when editing

1. **Add tests for any new behavior.** This codebase has 235 crawler unit tests + 1,064 integration tests. New code without tests will fail review. Use Playwright's test runner with `playwright.unit.config.ts` for unit tests.
2. **Don't leak internal fields into manifests.** The `_labelSource` and other `_`-prefixed fields are stripped before serialization (`stripInternalFields` in `discover.ts`). New internal fields must follow the underscore convention.
3. **Property names are derived from selector first, AI label second.** This is intentional for run-to-run stability. See `stableNameFromSelector` in `naming.ts`.
4. **The generated code imports from `@playwright-elements/core`.** Don't introduce dependencies that the framework doesn't already export.
5. **Credentials never reach the model.** Substitution happens at dispatch time in `agent-explore.ts` (`resolveCredentials`). Never log unresolved values.
6. **Network observation is centralized.** All `NetworkObserver` instances should pass an `ApiFilterOptions` so static/telemetry stays out of manifests.

## Common edits, mapped to files

| User asks | File(s) to edit |
|---|---|
| "Add support for a new typed wrapper" | `src/types.ts` (extend `WrapperType`), `src/discover.ts` (`classifyWrapperType` + `GROUP_SELECTOR`), `src/emitter.ts` (`WRAPPER_TO_FACTORY`) |
| "The agent should accept a new tool" | `src/agent-types.ts` (extend `AgentDecision`), `src/ai/agent-anthropic.ts` (add `TOOLS` schema + `extractDecision` branch), `src/agent-explore.ts` (`decisionToCandidate`) |
| "Drop a new kind of API noise" | `src/api-deps-filter.ts` (`TELEMETRY_PATH_PATTERNS` or `TELEMETRY_HOSTS`) |
| "Better label resolution for X" | `src/discover.ts` (`resolveLabel`) and `src/explore-planner.ts` (`textOf`, in-page extractor) |
| "Stable property name for Y selector" | `src/naming.ts` (`stableNameFromSelector`) |
| "Fix a generated-code defect" | `src/emitter.ts` — and add a regression test in `tests/unit/emitter.spec.ts` |

## Behavior to NOT change without explicit user approval

- **The default exploration strategy when `--ai-agent` is set is `balanced`.** Conservative drops fills (mutation risk) and the agent gets stuck on login forms.
- **`fill_field` and `click_locator` decisions are deduplicated against history by label.** Repeated emissions on the same field are silently skipped.
- **Filled-field actions don't increment `consecutiveNoChange`.** A multi-field form would otherwise trip the early-stop guard.
- **`<input type="submit|button|reset">` returns its `value` as the visible label** (not `id`/`name`/`type`). Reverting this re-introduces the `root.click("login-button")` regression.

## Out-of-scope work

The following exist as design docs but are not yet implemented. Don't claim they work:

- MCP `browser_snapshot` ref-based dispatch (Slice 2b in [docs/AUTOPOM-MCP-EXPLORATION-PLAN.md](docs/AUTOPOM-MCP-EXPLORATION-PLAN.md))
- Orphan-textInput emission as named handles (deferred from Slice 6C)
- Sitemap-aware planning (discussed but not built)

## How to test your changes locally

```bash
cd tools/crawler

# Unit tests — fast, no servers
npx playwright test --config=playwright.unit.config.ts

# Re-pack the tarball so a downstream project can pick up your changes
npm run build && npm pack

# Manual smoke (in a downstream project)
npm install /abs/path/to/playwright-elements-crawler-0.1.0.tgz
npx pw-crawl explore https://www.saucedemo.com/ --ai-agent --ai-model claude-sonnet-4-6 \
  --credentials-file .auth/credentials.json --output .pom
```

## When the agent should escalate to the human

- The user has not set `ANTHROPIC_API_KEY` and is asking you to invoke `--ai-agent`. Ask them to set it; do not attempt to run without it.
- The CLI returns a Cloudflare-style timeout. Bot detection is environmental — propose `auth-setup` + `--auth-state` as the workaround, don't keep retrying.
- A `pw-crawl drift` run reports failures. Don't silently re-explore — surface the diff to the human first.

## License

ISC. See [LICENSE](LICENSE).
