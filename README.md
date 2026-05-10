# agentic-pom

> **An AI-agent-driven Playwright Page Object generator.** Point it at a URL, hand it credentials if the app needs them, and get back a typed TypeScript page-object suite — including login flow, navigation helpers, typed wrappers for `<select>` / date pickers, and an API-dependency manifest.

[![Tests](https://img.shields.io/badge/tests-1940%20passing-brightgreen)](#)
[![Node](https://img.shields.io/badge/node-%3E%3D20-blue)](.nvmrc)

## What it does

You run:

```bash
npx pw-crawl explore https://app.example.com \
  --mcp --ai-agent --ai-model claude-sonnet-4-6 \
  --credentials-file .auth/credentials.json \
  --output ./.pom
```

You get a folder of Playwright Page Objects:

```
.pom/
├── pages/
│   ├── home.ts              # login form + goToInventory(page) helper
│   ├── inventory.ts         # productSort: select(...), inventoryList, goToInventoryItem()
│   ├── inventoryItem.ts     # productDetailCard, goToInventory() (back-button)
│   └── shared-components.ts # nav/footer/header extracted across pages
└── manifests/
    ├── home.manifest.json
    └── inventory.manifest.json
```

…and immediately runnable tests:

```ts
import { homePage, inventoryPage, goToInventory } from "./.pom/pages/home";

test("buys a backpack", async ({ page }) => {
  await page.goto("https://app.example.com");
  await goToInventory(page);                                // logs in, waits for /inventory
  const inv = inventoryPage(page);
  await inv.productSort.choose("Price (low to high)");      // typed select wrapper
  await inv.inventoryList.click("Sauce Labs Backpack");
});
```

## How it works

A Claude-driven exploration agent picks browser actions one at a time (`click_candidate`, `fill_candidate`, `navigate`, `stop`) over a Playwright transport. Each step rescans the DOM, attributes API calls to actions, and accumulates a per-route manifest. The emitter turns the manifest into typed page-object code that imports from the [`@playwright-elements/core`](framework/) framework.

Three modes:

- **`--ai-agent`** — Claude tool-use loop drives the browser (recommended).
- **`--mcp`** — routes browser actions through Microsoft's [`@playwright/mcp`](https://github.com/microsoft/playwright-mcp) server. Useful when you want to plug in to other MCP-compatible agents.
- **(no flags)** — heuristic exploration, no AI calls. Fastest, dumbest path. Good for sanity checks.

A separate **drift** subcommand replays the recorded action graph against the live app and surfaces page-object regressions:

```bash
npx pw-crawl drift ./.pom --repair         # AI-assisted locator updates when something changed
```

## When to use this

- ✅ You have a real running web app and want a baseline Playwright POM in minutes, not days.
- ✅ The app has a login flow you can express as a `{KEY: value}` credentials map.
- ✅ You want **typed** page objects (`inv.productSort.choose("...")`), not raw locators.
- ✅ You want API-call assertions wired up automatically (`waitForResponse(/users\/me/)` etc.).

## When NOT to use this

- ❌ The app is behind a Cloudflare bot challenge or similar headless-browser block.
- ❌ The app's primary interactions are non-DOM (canvas, WebGL, deep keyboard-only flows).
- ❌ You need pixel-perfect snapshots (this is a structural tool — pair with visual-regression for that).

## Quick start

```bash
git clone https://github.com/ShaddieRaq/agentic-pom.git
cd agentic-pom

# Install root + all workspace packages
npm install

# Build the crawler (publishes a tarball you can install in your test project)
cd tools/crawler && npm run build && npm pack

# In your downstream project:
cd /path/to/your/test/project
npm install /abs/path/to/agentic-pom/tools/crawler/playwright-elements-crawler-0.1.0.tgz
npx playwright install chromium

# Set your Anthropic key
export ANTHROPIC_API_KEY=sk-ant-…

# Optional: credentials for forms the agent will fill
mkdir -p .auth && echo '{"USER":"standard_user","PASS":"secret_sauce"}' > .auth/credentials.json

# Crawl
npx pw-crawl explore https://www.saucedemo.com/ \
  --mcp --ai-agent --ai-model claude-sonnet-4-6 \
  --credentials-file .auth/credentials.json \
  --max-actions 20 --output .pom
```

## Limitations (read this before launching)

- **Bot detection trips it up.** Sites running Cloudflare's "Verify you are human" challenge will time out at navigation. Manual workaround: log in once with `pw-crawl auth-setup`, then explore with `--auth-state`.
- **`--mcp` + `--auth-state` together are unsupported.** The CLI auto-falls back to direct Playwright in that combination and prints a warning.
- **First exploration of a login form may double-attempt fills/clicks.** The agent retries with a different locator when its first synthesized locator misses. This costs ~3-4 actions of the budget. Not a correctness issue.
- **Group property names favor stability over readability** when the underlying selector is a structural class. `headerSecondaryContainer` reads worse than `productsToolbar`, but it stays identical across runs.
- **Credentials never leave your machine in plaintext to the model.** Values are looked up at dispatch time from a local file; the model only sees `{{KEY}}` placeholder names.

## Project layout

This repo is a monorepo with two published packages and a curated set of fixture apps used to validate the framework.

```
agentic-pom/
├── tools/crawler/          # ← @playwright-elements/crawler — pw-crawl CLI
├── framework/              # ← @playwright-elements/core — runtime label-first wrapper
├── apps/                   # ← 7 fixture apps (vanilla, React, Vue, Angular, Svelte, Next, Lit) used as test targets
├── docs/
│   ├── REQUIREMENTS.md
│   ├── ROADMAP.md
│   └── AUTOPOM-MCP-EXPLORATION-PLAN.md   ← design doc for the exploration agent
├── AGENTS.md               # ← instructions for AI coding agents working in this repo
└── README.md
```

## Two libraries, one story

- **[`@playwright-elements/core`](framework/)** — runtime label-first wrapper around Playwright. `await loginForm.write("Email", value)` instead of CSS selectors. Auto-detects element types via a handler registry.
- **[`@playwright-elements/crawler`](tools/crawler/)** — the `pw-crawl` CLI. Drives a real browser with an exploration agent, captures the DOM + network behavior, generates code that imports from the framework.

The crawler is the headline; the framework is what makes the generated code worth reading.

## Tests

```bash
npx playwright test --config=playwright.unit.config.ts   # unit (235 crawler + 312 framework)
npx playwright test                                      # integration (1,393 across 7 apps)
```

**1,940 tests passing** as of the latest commit.

## License

ISC. See [LICENSE](LICENSE).

---

Built as a passion project. Star the repo if it saves you a day of writing locators.
