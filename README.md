# web-e2e-test

End-to-end testing skill for Claude Code — generates and runs Playwright browser tests to catch the real bugs that pass unit tests but break in the browser.

## The Problem

Claude is great at writing unit tests, but unit tests alone don't catch real-world bugs: broken navigation, forms that don't submit, pages that crash on load, API integration failures. You've seen it — all tests green, but the app is broken. End-to-end browser tests are the missing layer, but writing them manually is tedious and Claude often skips them.

This skill automates the entire end-to-end testing workflow: a QA agent analyzes your code and generates Playwright test cases, runs them against your actual application in a real browser, and produces an actionable report with fix suggestions. The QA agent can be OpenAI Codex (via MCP) or Claude itself — using an independent agent for QA reduces the chance of the same blind spots that caused the bugs in the first place.

## How It Works

When triggered, the skill runs a 4-phase workflow:

1. **Setup** — Detects or creates a Playwright config, auto-detects your dev server port (from Docker Compose, Vite, Next.js, Angular, etc.), installs Chromium if needed.
2. **Context Collection** — Reads your project specs, route definitions, page components, and existing tests to understand the application.
3. **QA Agent Dispatch** — Sends the collected context to a QA agent to generate `.spec.ts` test files. Uses OpenAI Codex via MCP if available, otherwise Claude generates the tests itself.
4. **Test Execution & Report** — Starts the dev server if not running, executes the tests with Playwright, and produces a Markdown report with pass/fail summary, screenshots, and fix suggestions for every failure.

## Installation

```bash
# Add the marketplace
claude plugin marketplace add hideak1/web-e2e-test

# Install the skill
claude plugin install web-e2e-test
```

## Prerequisites

- **Node.js** (v18+)
- **Playwright** — auto-installed by the skill (`npx playwright install chromium`)
- **OpenAI Codex MCP** (optional, recommended) — provides an independent QA agent for better test quality. Install with:
  ```bash
  claude mcp add codex -- npx -y @openai/codex mcp-server
  ```
  Without Codex, Claude generates the tests itself. This still works, but using a separate agent for QA means different perspectives and fewer shared blind spots.

## Usage

Trigger the skill with natural language:

```
"Test the login flow"
"Run end-to-end tests for this project"
"Generate browser tests for the checkout page"
"Write Playwright tests based on the spec"
```

The skill activates on keywords: `e2e`, `end to end`, `playwright`, `browser test`, `UI test`, or after implementing a user-facing feature.

## QA Agent Options

| Agent | When Used | How |
|-------|-----------|-----|
| **OpenAI Codex** (`@openai/codex`) | Auto-detected if `codex` MCP tool is available | Invoked via MCP with `prompt` + `cwd`, writes test files directly |
| **Claude (fallback)** | When Codex is not available | Claude generates and writes test files using filesystem tools |

No manual configuration needed — the skill checks for Codex automatically and falls back to Claude.

**Why use a separate QA agent?** When Claude writes both the code and the tests, it tends to test what it *thinks* the code does rather than what it *should* do. An independent agent brings fresh eyes — it reads the spec and the code separately, and is more likely to catch assumptions that led to bugs.

## Configuration

### Zero-Config Defaults

| Setting | Default | Source |
|---------|---------|--------|
| Base URL | Auto-detected | Docker Compose, `vite.config.*`, `next.config.*`, `angular.json`, `package.json`, `.env` |
| Browser | Chromium | Playwright config |
| Test directory | `e2e-tests/generated/` | Convention |
| Results directory | `e2e-tests/results/` | Convention |
| Report format | Markdown (for agent consumption) | Convention |

### Customization

If your project already has a `playwright.config.ts` (or `.js`/`.mjs`), the skill uses it as-is. Otherwise, it generates a minimal config at `e2e-tests/playwright.config.ts` that you can edit.

To override the base URL, either:
- Tell Claude directly: "test against `http://localhost:5173`"
- Edit the generated `playwright.config.ts`

## Project Structure

After running, the skill creates:

```
your-project/
  e2e-tests/
    playwright.config.ts   # Generated if none exists
    generated/             # .spec.ts test files from QA agent
    results/               # Playwright JSON output, screenshots, traces
    reports/               # Markdown test reports (report-YYYY-MM-DD-HHmmss.md)
```

## CLAUDE.md Integration

Add this to your project's `CLAUDE.md` to automatically trigger end-to-end testing during development:

```markdown
## Testing Policy

- After implementing any user-facing feature or fixing a UI bug, run end-to-end tests using the web-e2e-test skill.
- End-to-end tests live in `e2e-tests/generated/`.
- Fix failing end-to-end tests before marking a feature complete.
- Unit tests passing is not sufficient — browser tests must also pass.
```

## License

MIT
