---
name: visual-qa
description: Use when making visual changes to a web application—adding, removing, repositioning, or restyling elements, changing layout or responsive behavior, or modifying interactions—to validate their quality and integrity before finalizing. Also for web UI work where visual correctness, interaction integrity, or stakeholder approval matters.
---

# Visual QA

## Role

When you touch anything visible in a web frontend, you become a meticulous frontend engineer and QA engineer. The browser is the only judge of visual correctness. Mock first, then wire behavior, and prove every change with screenshots before the code is finalized or a PR is opened.

**You do not get to skip this skill.** If you made a visual change, you run visual QA. No exceptions.

## Overview

Validate web UI changes in a real browser before any code is finalized. Mock before writing behavior, capture screenshots at every step, exercise every interaction, cover every state and breakpoint, compare against a baseline, and attach proof to the pull request.

**Why this matters:** Static designs and dev tools lie. Only real browser screenshots reveal layout bugs, overflow, cutoff, responsive breakage, z-index collisions, and interaction failures that users actually see. Code that "looks right" in your head frequently renders wrong in the browser.

## Browser Tooling

Drive a real Chrome browser through the **chrome-devtools MCP**. Use it for every navigation, interaction, snapshot, and screenshot in this skill.

Setup (OpenCode `~/.config/opencode/opencode.jsonc`):

```jsonc
"mcp": {
  "chrome-devtools": {
    "type": "local",
    "command": ["npx", "-y", "chrome-devtools-mcp@latest", "--headless", "--no-usage-statistics"]
  }
}
```

Run headless. The MCP keeps a persistent Chrome profile at `~/.cache/chrome-devtools-mcp/chrome-profile`, so a login survives across runs. Do not add `--isolated`; it wipes the profile each run.

**First-time login (prime the profile):** a headless browser cannot show a login page. Before the first headless run, start the MCP once without `--headless`, let the user log in, then restore `--headless`. The profile keeps the session cookies, so every later headless run stays logged in. The agent never enters credentials. If the login is a plain email + password form and the user supplies credentials, the agent may fill the form headless with `fill` + `click`; SSO, magic links, MFA, and provider bot checks cannot be automated.

**Required chrome-devtools MCP operations:** `list_pages`, `new_page` / `navigate_page`, `select_page`, `close_page`, `take_snapshot`, `click`, `fill`, `fill_form`, `type_text`, `press_key`, `evaluate_script`, `take_screenshot`, `resize_page`, `hover`, `wait_for`, `list_network_requests`, `list_console_messages`, `upload_file`.

**Fallback:** if the chrome-devtools MCP is not configured, use the harness `browser` toolset instead. It maps one-to-one (see Quick Reference). It cannot resize the viewport, so for breakpoints fall back to the `playwright` skill (`playwright-cli resize <width> <height>`) until the MCP is installed.

**Tool contract (both toolsets):**
- Refs / element ids come from the latest snapshot for the active page. Never invent or reuse them across pages. Re-snapshot after any DOM change.
- Capture screenshots to a descriptive `filePath`, then read the file with the `read` tool to analyze it. Do not glance and assume.
- All harness `browser` tools take an explicit `tabID`. The chrome-devtools MCP selects a page with `select_page` first.

## When to Use

Use this skill MANDATORILY whenever you:

- Add, remove, reposition, or resize visible elements
- Change styles: spacing, color, typography, shadows, borders, opacity, z-index
- Change layout or responsive behavior at any breakpoint
- Add or alter interactions: buttons, modals, forms, navigation, dropdowns, tabs, toggles, tooltips
- Modify anything that affects what a user sees or clicks
- Work on any task where a stakeholder must approve the visual outcome

Do not use when the change is purely non-visual (e.g., API schema, state logic with no render impact, backend-only code).

## The Iron Rule

**No visual change is done until it has been validated in the real browser with screenshots covering every affected state.**

Violating the letter of this rule is violating the spirit. "It's a small change," "I already checked it," and "I'll screenshot later" are all violations.

## Dev Server Is Mandatory

**No running dev server is never an excuse to skip visual QA.** If the app is not running, start it yourself before any screenshot step:

1. Run the dev server out of your own worktree, never the main checkout — the main checkout serves stale code without your change.
2. In the Metal frontend, a worktree has no `node_modules`. Install them in the worktree with `npm install`, or clone the main checkout's tree: `cp -Rc frontend/node_modules <worktree>/frontend/node_modules`. Never symlink — Turbopack panics on symlinked `node_modules`.
3. Pick a port. The default `3000` is already allowed by the local backend. If you need another port (for example to avoid a parallel agent), derive one and verify it is free with `lsof -i :<port>`.
4. From the worktree's `frontend/` directory, start the server as a background task: `npm run dev -- --port <port>`.
5. Point the frontend at the backend you want (see Local Dev Setup). If the UI depends on backend changes in your worktree, run that backend from the worktree on its own port too.
6. Keep the server running for the whole QA session. Kill it when done.

## Local Dev Setup (Metal platform)

- **Frontend:** `frontend/` (Next.js). Dev via `make dev`, or `npm run dev -- --port <port>`.
- **API base:** `NEXT_PUBLIC_SERVER_URL` in `frontend/.env.local`. The local gateway is `http://localhost:8000`; the production value is a remote host.
- **Local backend:** `black-sabbath-api` gateway on `:8000`, `metallica-api` on `:8081`. Start the stack with `backend/docker-compose.yaml`.
- **CORS:** `backend/services/black-sabbath-api/helpers/cors.js` builds `ALLOWED_ORIGINS` from a fixed list plus `process.env.CLIENT_URL` and `CLIENT_URL2`. `backend/.env` sets `CLIENT_URL=http://localhost:3000`. A frontend origin that is not in that list is rejected by the local backend. To test on another port, add the origin (or set `CLIENT_URL2`) and restart `black-sabbath-api`.
- **Secrets:** `.env.local` and `backend/.env` are secret files. Never commit them. Never overwrite them without asking the user.
- **Auth:** the app uses PropelAuth. Ask the user to log in once inside the MCP browser (headed mode). Never fake a session or ask the user to disable auth.

## Browser Session Setup

1. Call `list_pages` (or `browser.tabs.list`) first to see what the MCP-controlled browser has open.
2. The MCP browser is its own Chrome instance with a persistent profile. It does NOT share the user's everyday browser session — do not assume cookies, extensions, or logins carry over.
3. Open the app with `new_page` (or `browser.tabs.open({ url })`), then `select_page` to make it active.
4. If the app requires auth and the profile is not logged in, use the first-time login procedure above (one headed run). Do not attempt to fill credentials yourself unless the user supplies them. Never ask the user to disable auth or seed fake sessions to work around this.
5. Work in dedicated pages per test flow; use `select_page` to switch and `close_page` when done.

## Core Workflow

Follow these steps in order. Do not skip ahead.

### 0. Read the design system

Before mocking, check whether the project root contains a `DESIGN.md` (or equivalent design-system doc). If it exists, read it and follow its tokens, typography, spacing, motion, and component rules. If a user request conflicts with it, raise the conflict before proceeding.

### 1. Capture the baseline (existing views only)

Before changing any existing view, navigate to every affected page and `take_screenshot` (with a descriptive `filePath`) of each state you are about to touch. Save baselines under a task-specific directory such as `docs/screenshots/<task-name>/baseline/`. Read each with the `read` tool so you actually know what "before" looks like.

Skip only when the task adds a brand-new view with no changes to existing UI.

### 2. Mock before writing behavior

Build the UI with hardcoded placeholder data and static state first. Do not wire real APIs, auth, or dynamic behavior until the layout is approved.

**Why:** Real data and behavior hide layout problems. A static mock surfaces spacing, alignment, and typography issues before they are buried behind logic.

Navigate to the page, screenshot every proposed state, read each screenshot with the `read` tool, analyze it (see QA Checklist), and present what you see to the user. Get explicit approval before proceeding.

### 3. Wire behavior

Only after the mock is approved, replace placeholder data with real state, APIs, and interactions.

### 4. Validate every meaningful change

After each change, restart or refresh as needed, navigate, and screenshot. Run the full QA Checklist below and fix visual issues before moving to the next change. Also check `list_console_messages` for new errors or warnings your change introduced — visual QA includes a clean console.

### 5. Compare against the baseline

Place post-change screenshots next to the step-1 baselines and diff them visually. Look for:

- Visual regressions and any change beyond the task's spec
- Disappearing components or buttons
- Misalignments, broken spacing, or layout shifts in untouched areas
- Visual artifacts, broken images, or render glitches
- Text display errors: unintended line breaks, truncation, repeating or garbled text

Any difference that is not part of the approved spec means revert the offending change and re-validate from step 4.

### 6. Capture final states

Before opening a PR, screenshot every state you created or updated:

- Default/initial view
- Each interaction state (open, closed, hover, active, disabled)
- Empty, loading, error, and success states
- Each step of multi-step flows
- Mobile, tablet, and desktop breakpoints (`resize_page`; `emulate({ viewport })` for device profiles)

Save screenshots with descriptive filenames under a task-specific directory such as `docs/screenshots/<task-name>/`.

### 7. Attach evidence to the PR

Include every final screenshot in the PR description. Do not rely on text alone — a reviewer must see what changed without running the code. If the project has a PR skill, invoke it and reference the screenshot paths.

Host the images through GitHub's user-attachment store. Never commit screenshot files, never push an assets branch, and never use `gh gist` (it rejects binaries).

**Upload each file, then embed the returned URL.** Upload against the repo that owns the PR:

```bash
TOKEN=$(gh auth token)
RID=$(gh api "repos/<owner>/<repo>" --jq .id)
curl -sS "https://uploads.github.com/user-attachments/assets?name=<file>.png&content_type=image/png&repository_id=$RID" \
  -X POST -H "Authorization: Bearer $TOKEN" -H "Accept: application/json" \
  --data-binary "@<file>.png"
# -> {"url":"https://github.com/user-attachments/assets/<uuid>"}
```

Embed the URL as `![alt](url)`. GitHub rewrites it to a fresh signed `private-user-images.githubusercontent.com` URL at render time, so the image shows for everyone with repo access.

**Verify, do not assume.** The attachment URL returns 404 to an anonymous `curl` because the repo gates the asset; that is expected, not a failure. Confirm the renderer accepts it with the Markdown API:

```bash
gh api /markdown --method POST -H "Accept: text/html" -f mode=gfm -f context=<owner>/<repo> \
  -f text='![x](https://github.com/user-attachments/assets/<uuid>)'
```

A correct result contains `private-user-images.githubusercontent.com`.

**Update the PR body without dropping existing content** — fetch, append, edit:

```bash
gh pr view <n> --repo <owner>/<repo> --json body --jq .body > body.md
cat visual-qa.md >> body.md
gh pr edit <n> --repo <owner>/<repo> --body-file body.md
```

Publish to every PR the screenshots prove (a linked frontend + backend pair). Posting the description update is authorized by this workflow and does not touch the repo tree.

## QA Checklist

Run through every applicable category. Use `take_screenshot` with a descriptive `filePath` for each state, then read the screenshot file with the `read` tool and analyze it — do not glance and assume.

### 1. Visual Integrity

- [ ] Elements render in the correct position and size
- [ ] Colors, fonts, spacing, shadows, borders match the design
- [ ] No overflowing or truncated text
- [ ] No cutoff, clipped, or hidden elements
- [ ] No render artifacts, broken images, or missing icons
- [ ] No unintended whitespace or misalignment
- [ ] Dark/light theme renders correctly if applicable

### 2. Layout & Spacing

- [ ] Alignment is consistent across the changed area
- [ ] No elements overlap or collide (check z-index stacking)
- [ ] Container boundaries are respected (no bleed-through)
- [ ] Scroll behavior is correct where content overflows
- [ ] Fixed/sticky elements behave as intended

### 3. Responsive Behavior

- [ ] Test at mobile, tablet, and desktop widths with `resize_page` (or `emulate({ viewport })`)
- [ ] No horizontal scrollbar at any breakpoint
- [ ] No elements shrink to unusable size or disappear
- [ ] Tap targets are large enough on mobile (at least 44px)
- [ ] Layout reflows rather than overflowing at narrow widths

### 4. Interaction Integrity

Use `take_snapshot` to locate interactive elements by their ids. Then:

- [ ] `click` every button, link, tab, menu item, chip, and card — verify the expected action fires
- [ ] `fill` every input and textarea — verify values are accepted and validated
- [ ] Trigger modals, dropdowns, toasts, drawers — verify they open and close correctly
- [ ] Submit forms with valid AND invalid data — verify validation messages appear
- [ ] Click cancel, close, back, escape, and dismiss — verify cleanup is correct
- [ ] Verify focus management and keyboard navigation (`press_key` for Tab, Enter, Escape)
- [ ] For rich text editors (`contenteditable`), focus the element first, then `type_text`

### 5. State Coverage

Screenshot and verify each state:

- [ ] Default/initial view
- [ ] Empty state (no data)
- [ ] Loading state (skeletons, spinners)
- [ ] Error state (failed requests, validation errors)
- [ ] Success state (completed actions, confirmations)
- [ ] Disabled state (buttons, inputs)
- [ ] Hover, active, focus-visible states (`hover` where supported)

### 6. Accessibility Spot-Check

- [ ] Keyboard navigation reaches all interactive elements
- [ ] Focus indicator is visible on every focusable element
- [ ] Modals and drawers trap focus and restore it on close
- [ ] Color contrast is sufficient for text on backgrounds
- [ ] Interactive elements have discernible accessible names

### 7. Regression Check

Your change can break nearby code. Verify the area around your change:

- [ ] Inspect sibling components and adjacent layout sections
- [ ] Verify shared layout containers still align
- [ ] Check that global styles, theme tokens, or utility classes you touched did not cascade into other components
- [ ] Navigate to 2-3 other pages that share the modified component or layout — confirm they still render correctly

## Quick Reference

chrome-devtools MCP is primary. The harness `browser` fallback is shown for when the MCP is not configured.

| Goal | chrome-devtools MCP | `browser` fallback |
|------|---------------------|--------------------|
| List open pages / pick the app | `list_pages`, `select_page` | `browser.tabs.list`, `browser.tabs.focus` |
| Open a page / navigate | `new_page(url)` or `navigate_page({ type: "url", url })` | `browser.tabs.open({ url })`, `browser.navigate({ tabID, url })` |
| Read structure / find element ids | `take_snapshot` (a11y tree) | `browser.snapshot({ tabID })` |
| Click element | `click(uid)` | `browser.click({ tabID, ref })` |
| Fill input / textarea | `fill(uid, value)` | `browser.fill({ tabID, ref, text })` |
| Multi-field form | `fill_form` | `browser.fill_form({ tabID, fields })` |
| Type into focused editor | `type_text` (rich `contenteditable`) | `browser.evaluate` |
| Keyboard shortcuts / Tab / Enter / Escape | `press_key` | `browser.press({ tabID, key })` |
| Run JS / inspect / scroll | `evaluate_script` | `browser.evaluate({ tabID, script })` |
| Capture state | `take_screenshot` with `filePath`; read it | `browser.screenshot({ tabID })`; read returned `path` |
| Show a screenshot to the user | (review pane) | `browser.preview({ path })` |
| Resize viewport | `resize_page(width, height)` / `emulate({ viewport })` | not available — use the `playwright` skill |
| Hover | `hover(uid)` | `browser.hover({ tabID, ref })` |
| Wait for content | `wait_for` | `browser.wait({ tabID, condition, text })` |
| Network / console evidence | `list_network_requests`, `list_console_messages` | `browser.network.list`, `browser.console` |
| Upload a file | `upload_file(uid, filePath)` | `browser.files.upload({ tabID, ref, paths })` |
| Native dialogs | `handle_dialog` | `browser.dialog({ tabID, action })` |
| Drag and drop | `drag` | `browser.drag({ tabID, from, to })` |
| iframes | `list_frames` / `select_frame` | `browser.frames({ tabID })` |
| Accessibility / perf audit | `lighthouse_audit` | `browser.lighthouse({ tabID })` |
| Reload / back / forward | `navigate_page` (reload / history) | `browser.reload`, `browser.back`, `browser.forward` |
| Clean up | `close_page` | `browser.tabs.close({ tabID })` |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| "I'll code it, then show you" | Mock and screenshot first. Approval before behavior. |
| "It's a small change, no QA needed" | Small visual changes still break layout. Run the checklist. |
| "I already checked it looks fine" | You checked your memory, not the browser. Screenshot and read it. |
| "Design is already approved in Figma" | Browser rendering is the truth. Implement and screenshot anyway. |
| One screenshot at the end | Screenshot after every meaningful change and every state. |
| Only testing the happy path | Test invalid input, cancel, close, escape, empty, error, loading, disabled. |
| Only checking the changed element | Regressions happen nearby. Inspect siblings and shared layouts — and diff against the baseline. |
| No baseline captured | Screenshot existing views before touching them (step 1). |
| "No dev server is running, so I can't screenshot" | Start one from your worktree on a free port (see Dev Server Is Mandatory). Skipping visual QA is not an option. |
| Attaching one image to the PR | Every changed state and breakpoint needs a screenshot. |
| Committing screenshots, pushing an assets branch, or using a gist to embed images | Upload through GitHub's user-attachment endpoint (step 7). Nothing is committed. |
| Treating a 404 on the attachment URL as a broken upload | Private-repo assets are gated; verify with the Markdown API (step 7). |
| Trusting dev tools over screenshots | DevTools do not show what users see. Use `take_screenshot`. |
| Screenshot without a `filePath` | Always pass a descriptive `filePath` and read the file. An unnamed temp shot is easy to lose and never analyzed. |
| "I'll add screenshots to the PR later" | Later never comes. Capture before you finalize. |
| Stale element id after a DOM change | Re-run `take_snapshot`; refs are invalidated by mutations. |
| Running headless before the profile is primed | The app shows a login page with no window to complete it. Prime the profile once with a headed run (see Browser Tooling). |
| Editing `.env.local` / backend `.env` freely | These are secrets. Never commit them. Ask before editing. |
| Symlinking `node_modules` into a worktree | Turbopack panics. Use `npm install` or `cp -Rc`. |
| Forgetting CORS for a custom port | A non-`3000` frontend origin is rejected by the local backend until allowed (see Local Dev Setup). |

## Red Flags — Stop and Run QA

- "It's a small change, no mock needed"
- "No dev server is running, I'll verify after merge"
- "I already checked it looks fine"
- "One screenshot is enough"
- "I tested the main button; the rest are the same"
- "I'll add screenshots to the PR later"
- "I'll commit the screenshots (or a gist, or an assets branch) so they render"
- "The design is already approved in Figma"
- "I only changed CSS, it can't break interactions"
- "I'll skip the MCP and use the harness tools"
- "I'll point the frontend at the remote API to avoid CORS"

Any of these means: stop, load the page in the browser, and run the QA checklist with screenshots.