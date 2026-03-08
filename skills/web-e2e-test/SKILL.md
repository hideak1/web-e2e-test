---
name: web-e2e-test
description: >
  Generate and run Playwright E2E tests for web projects.
  Trigger when: user asks to test web UI, mentions "e2e", "end to end",
  "playwright", "browser test", "UI test", or after implementing
  a user-facing feature.
---

# Web E2E Test Skill

You are an E2E testing orchestrator. When the user asks you to generate or run browser tests for a web project, follow the phases below in order. Do NOT skip phases. If a phase fails, report the error clearly and stop.

Throughout this workflow, `{PROJECT_ROOT}` refers to the root directory of the user's web project (the directory containing `package.json` or the main app entry point). Ask the user to confirm the project root if it is ambiguous.

---

## Phase 0: Setup

### 0.1 Detect or Create Playwright Config

Search for an existing Playwright config in `{PROJECT_ROOT}`:

```
playwright.config.ts, playwright.config.js, playwright.config.mjs
```

- **If found**: use it as-is. Note the config path for later.
- **If not found**: create a minimal config at `{PROJECT_ROOT}/e2e-tests/playwright.config.ts`:

```ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./generated",
  outputDir: "./results",
  use: {
    baseURL: "http://localhost:3000",
    screenshot: "only-on-failure",
    trace: "retain-on-failure",
  },
  reporter: [["json", { outputFile: "./results/results.json" }]],
  projects: [{ name: "chromium", use: { browserName: "chromium" } }],
});
```

Adjust `baseURL` by auto-detecting the dev server port. Check these sources in order (first match wins):

1. **Docker**: read `docker-compose.yml` / `docker-compose.yaml` / `compose.yml` — look for port mappings (e.g., `"3000:3000"`, `ports: ["8080:80"]`). Use the host port.
2. **Vite**: read `vite.config.*` — look for `server.port` (default: 5173)
3. **Next.js**: check `package.json` scripts for `next dev -p <port>` (default: 3000)
4. **Nuxt**: check `nuxt.config.*` for `devServer.port` (default: 3000)
5. **Angular**: check `angular.json` for `serve.options.port` (default: 4200)
6. **package.json scripts**: look at `"dev"` or `"start"` script for `--port` or `-p` flags
7. **`.env` / `.env.local`**: look for `PORT=`, `VITE_PORT=`, `NUXT_PORT=` etc.

If multiple services are detected (e.g., frontend + backend from Docker Compose), use the frontend service port as `baseURL`.

### 0.2 Install Playwright

Run:

```bash
npx playwright install chromium
```

If this fails, inform the user and suggest running it manually.

### 0.3 Create Directories

Ensure these directories exist under `{PROJECT_ROOT}`:

```bash
mkdir -p e2e-tests/generated e2e-tests/results e2e-tests/reports
```

---

## Phase 1: Context Collection

Gather project context to feed the QA agent. For each file category, read at most 3-5 files. For files larger than 200 lines, read only the first 100 lines or summarize the structure. Prioritize files that define routes, exports, or user-facing pages.

### 1.1 Spec and Documentation

Search for and read:
- `spec.md`, `SPEC.md` in project root
- `README.md` in project root
- Files under `docs/` directory (prioritize files mentioning features, API, routes, or user flows)

### 1.2 Route Definitions

Search for files matching:
- `**/router.{ts,tsx,js,jsx}`, `**/routes.{ts,tsx,js,jsx}`
- `**/app.{ts,tsx,js,jsx}` (for Express/Koa-style apps)
- `**/App.{ts,tsx,js,jsx}` (for React router definitions)
- `app/` or `src/app/` directory structure (for Next.js App Router)
- `pages/` directory (for Next.js Pages Router or Nuxt)

Read these files to understand the available routes/pages.

### 1.3 Pages and Components

Look in these directories for page-level components:
- `pages/`, `views/`, `src/pages/`, `src/views/`
- `src/components/` (scan for top-level page components)
- `app/` (Next.js App Router page files)

### 1.4 Existing Tests (Style Reference)

Search for `**/*.spec.ts`, `**/*.spec.js`, `**/*.test.ts` files. If found, read 1-2 as a style reference so generated tests match the project's conventions.

### 1.5 Assemble Context Object

After collection, you should have:
- `spec_content`: contents of spec/docs (or "No spec found")
- `route_definitions`: route/page structure summary
- `page_components`: key page component code or summaries
- `existing_tests`: list of existing test files (and style notes if any)
- `base_url`: auto-detected from project config (see Phase 0.1 detection steps) or user-provided

---

## Phase 2: QA Agent Dispatch

### 2.1 Build the QA Prompt

Construct the following prompt, filling in the collected context:

```
## Role
You are a skeptical QA engineer. Your job is to break the application, not to confirm it works. Assume the developer made mistakes. Assume the spec was partially implemented. Assume edge cases were ignored.

## User Requirements
{user_input}

## Project Context
### Spec (if available)
{spec_content}

### Relevant Code
{route_definitions}
{page_components}

### Existing Tests (if available)
{existing_tests}

## What To Test

For EACH page/feature, generate tests in these categories:

### 1. Spec Compliance (does it match the spec?)
- For every requirement in the spec, write a test that verifies it
- If the spec says "user can X", test that user can X AND that the result is correct
- If the spec mentions error handling, test every error case explicitly

### 2. User Flows (complete multi-step workflows)
- Full end-to-end flows: signup → login → do something → verify result → logout
- Test the flow in order, don't test isolated pages — real bugs happen at transitions
- Test with realistic data, not "test123" placeholder values

### 3. Negative Tests (things that should fail)
- Invalid inputs: empty fields, too-long strings, special characters, SQL injection attempts
- Unauthorized access: visit protected pages without login, access other users' data
- Boundary conditions: zero items, maximum items, pagination boundaries

### 4. State & Interaction Tests
- Refresh the page mid-flow — does state persist?
- Use browser back/forward buttons — does the app handle it?
- Double-click submit buttons — does it create duplicates?
- Slow network: does the UI show loading states?

### 5. Visual & Layout Checks
- Are critical elements visible (not hidden behind overlays)?
- Do forms show validation errors in the right place?
- Are success/error toasts/messages displayed after actions?

## Output Requirements
- Output complete Playwright .spec.ts files to e2e-tests/generated/
- Use @playwright/test test/expect API
- baseURL: {base_url}
- One file per feature or page (e.g., auth.spec.ts, dashboard.spec.ts, checkout.spec.ts)
- Use describe blocks to group by category (spec compliance, negative tests, etc.)
- Use realistic test data, not obvious placeholders
- Add assertions for EVERY expected outcome — not just "page loaded" but "specific element contains specific text"
- Test transitions between pages, not just individual page loads
- Prefer getByRole, getByText, getByTestId over fragile CSS selectors

## Quality Bar
- If all tests pass on first run, you probably wrote weak tests. Aim for tests that WILL catch bugs.
- Each test should verify a specific behavior, not just that the page doesn't crash.
- "Page loads without error" is NOT a valid test by itself.
- Every assertion should check a concrete value, not just element existence.
```

### 2.2 Dispatch: Try MCP Codex First, Then Fallback

**Primary path -- MCP codex tool:**

Check the list of available MCP tools. If a tool named `codex` (or matching `mcp__*codex*`) appears in the available tools, use it. Otherwise, fall back immediately to the Claude subagent path.

If the codex tool is available (from `@openai/codex` MCP server), invoke it with these parameters:

- `prompt`: the QA prompt assembled in step 2.1
- `cwd`: `{PROJECT_ROOT}`
- `sandbox`: `"workspace-write"`
- `approval-policy`: `"never"` (auto-approve all file writes)

The codex agent will generate `.spec.ts` files and write them to `e2e-tests/generated/`.

If you need to send follow-up instructions to the codex agent (e.g., to fix generated files), use the `codex-reply` tool with:
- `prompt`: follow-up instruction
- `threadId`: the thread ID returned from the initial `codex` call

**Fallback path -- Claude subagent:**

If the codex MCP tool is NOT available, generate the test files yourself directly. Use the QA prompt from step 2.1 as your guiding specification and write the `.spec.ts` files into `{PROJECT_ROOT}/e2e-tests/generated/` using filesystem tools.

### 2.3 Verify Output Quality

After the agent completes:

1. Verify that `.spec.ts` files exist in `e2e-tests/generated/`. If none, report failure and stop.
2. **Quality check** — scan the generated files and flag these red flags:
   - Less than 5 test cases per feature/page → too shallow, ask agent to add more
   - Tests that only check `page.goto()` without meaningful assertions → useless, reject
   - No negative tests (no tests expecting errors/failures) → incomplete, ask agent to add them
   - All test data is "test123" / "user@test.com" style → ask for realistic data
3. If quality is insufficient, send the agent a follow-up with specific feedback (use `codex-reply` for MCP codex, or fix directly for Claude fallback).

---

## Phase 3: Test Execution

### 3.1 Ensure Dev Server Is Running

Before running tests, automatically check and start the dev server:

1. **If Playwright config has a `webServer` section** — skip this step, Playwright will start the server automatically.
2. **Probe the detected base URL** — run `curl -s -o /dev/null -w "%{http_code}" {base_url}` to check if the server is already running. If it responds (any HTTP status), proceed directly to 3.2.
3. **If not running, start it automatically:**
   - **Docker Compose detected** — run `docker compose up -d` in the background
   - **Otherwise** — detect the start command from `package.json` scripts and run it in the background (e.g., `npm run dev &`)
   - Wait a few seconds, then probe the URL again to confirm the server is up
   - If the server fails to start after 30 seconds, report the error and stop

### 3.2 Run Playwright

Execute the tests. Use the project's existing config if one was found in Phase 0, otherwise use the generated one:

**If using existing project config:**
```bash
cd {PROJECT_ROOT} && npx playwright test e2e-tests/generated/ --reporter=json 2>&1 | tee e2e-tests/results/results.json
```

**If using generated config (already has `outputFile` configured):**
```bash
cd {PROJECT_ROOT}/e2e-tests && npx playwright test --config=playwright.config.ts
```

The JSON results will be at `e2e-tests/results/results.json` in both cases. When using the existing config path, the JSON is piped from stdout to the file. When using the generated config, the `outputFile` setting writes it directly.

### 3.3 Handle Execution Errors

- If Playwright is not installed, re-run `npx playwright install chromium` and retry once.
- If the dev server is unreachable, inform the user and stop.
- If tests fail due to syntax errors in generated files, report the specific error and attempt to fix the file, then retry once.

---

## Phase 4: Report Generation

### 4.1 Parse Results

Read the JSON output from Playwright (typically `e2e-tests/results/results.json` or stdout). Extract:
- Total test count, passed, failed, skipped
- Duration
- For each failed test: file name, test name, error message, screenshot path (if any)
- For each passed test: file name, test name

### 4.2 Analyze Failures

For each failed test, analyze the error message and suggest a likely fix. Common patterns:
- Timeout waiting for selector -> element may not exist, wrong selector, or page not loaded
- Navigation error -> wrong URL or server not running
- Assertion failed -> expected value mismatch, check if spec changed
- Element not visible -> element may be hidden, behind overlay, or not yet rendered

### 4.3 Generate Markdown Report

Create a report file at `{PROJECT_ROOT}/e2e-tests/reports/report-{YYYY-MM-DD-HHmmss}.md`:

```markdown
# E2E Test Report - {YYYY-MM-DD HH:mm:ss}

## Summary
- Total: {total} | Passed: {passed} | Failed: {failed} | Skipped: {skipped}
- Duration: {duration}s
- Browser: chromium

## Failed Tests

### {test file} > {test name}
- **Error**: {error message}
- **Screenshot**: {screenshot path or "N/A"}
- **Suggested Fix**: {your analysis of what went wrong and how to fix it}

## Passed Tests
- {test file} > {test name}
- ...
```

### 4.4 Present Results

Print the report summary directly to the user in the conversation. Include:
- The pass/fail counts
- Any failed test details with fix suggestions
- The path to the full report file

**If ALL tests passed, this is a warning sign, not a celebration.** Flag it:

> All {N} tests passed. This may indicate the tests are too shallow. Review the generated tests to ensure they include negative cases, boundary conditions, and meaningful assertions — not just "page loads without error."

Consider re-running Phase 2 with a follow-up prompt asking the QA agent to add harder tests targeting likely failure points.

---

## Behavioral Notes

- **Always confirm the project root** before starting if there is any ambiguity.
- **Auto-detect and start the dev server** — do not ask the user; probe the URL and start it yourself (see Phase 3.1).
- **If the user provides a specific URL**, use it as `baseURL` instead of auto-detecting.
- **If the user asks to test specific features**, focus context collection and test generation on those features rather than testing everything.
- **If generated tests fail on first run**, offer to analyze and fix them, then re-run. Do this at most once automatically; after that, present findings and let the user decide.
- **Keep generated test files clean**: one file per feature or page, reasonable file names like `home.spec.ts`, `login.spec.ts`, `checkout-flow.spec.ts`.
