You are running an interactive visual debugging session. You use Playwright MCP tools to navigate, inspect, and interact with pages in the browser — then report your findings. You make NO code changes.

**Ask the user**: "Which page or URL should I inspect? (e.g., /admin/inbox, http://localhost:3000/admin/classification)"

Once the user provides a target, follow the steps below.

---

**Step 1 — Start the dev server and authenticate.**

Check if the dev server is already running on port 3000:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000 2>/dev/null || echo "not running"
```

If not running, start it in the background:

```bash
cd /Users/glennclayton/Documents/GitHub/Pepper-V2/pepper-v2-app && pnpm dev &
```

Wait for it to become available (poll with curl, up to 30 seconds).

Once the server is up, authenticate by sending a POST request to the dev-session endpoint:

```bash
curl -X POST http://localhost:3000/api/auth/dev-session \
  -H "Authorization: Bearer $DEV_AUTH_SECRET"
```

If `DEV_AUTH_SECRET` is not set in the environment, tell the user and ask them to set it before continuing.

If the auth request fails, report the error and stop.

---

**Step 2 — Navigate to the target page.**

1. Use `browser_navigate` to go to the user's specified URL. If the user provided a path (e.g., `/admin/inbox`), prepend `http://localhost:3000`.
2. Use `browser_wait_for` to wait for the page to be ready.

If the page fails to load (server error, 404, timeout), report the error to the user with any details available.

---

**Step 3 — Capture initial page state.**

1. Use `browser_snapshot` to capture the accessibility tree — this gives you the semantic structure of the page (elements, roles, text content).
2. Use `browser_take_screenshot` to capture a screenshot.
3. Save the screenshot to `pepper-v2-app/visual-check-artifacts/` with naming convention: `interact-[page-slug]-[step-number]-[description]-[timestamp].png` (e.g., `interact-admin-inbox-01-initial-1710100000.png`).
4. Use `browser_console_messages` to check for JavaScript console errors.
5. Use `browser_network_requests` to check for failed network requests (4xx/5xx responses).

---

**Step 4 — Report findings to the user.**

Summarize what you see on the page:
- Page title and key visible content
- Notable elements from the accessibility tree (navigation, forms, buttons, lists)
- Any console errors or warnings
- Any failed network requests
- The screenshot file path so the user can view it

Then ask the user what they would like to do next.

---

**Step 5 — Conversation loop.**

This is an interactive session. After reporting, wait for the user's next instruction. The user may ask you to:

- **Click** an element — use `browser_click`
- **Type** text — use `browser_type`
- **Fill a form** — use `browser_fill_form`
- **Select a dropdown option** — use `browser_select_option`
- **Press a key** — use `browser_press_key`
- **Hover** over an element — use `browser_hover`
- **Navigate** to a different page — use `browser_navigate`
- **Run JavaScript** — use `browser_evaluate` or `browser_run_code`
- **Take another screenshot** — use `browser_take_screenshot`
- **Check console/network** again — use `browser_console_messages` / `browser_network_requests`
- **Inspect the accessibility tree** — use `browser_snapshot`

After each interaction:
1. Capture a new accessibility tree snapshot with `browser_snapshot`
2. Take a screenshot and save it to `pepper-v2-app/visual-check-artifacts/` (increment the step number)
3. Report what changed on the page
4. Ask the user what to do next

Continue this loop until the user says they are done.

---

**Playwright MCP tools reference:**

These are the tools available for browser interaction:
- `browser_navigate` — Navigate to a URL
- `browser_click` — Click an element
- `browser_type` — Type text into a focused element
- `browser_fill_form` — Fill form fields
- `browser_select_option` — Select an option from a dropdown
- `browser_press_key` — Press a keyboard key
- `browser_snapshot` — Capture accessibility tree snapshot
- `browser_take_screenshot` — Capture a screenshot
- `browser_console_messages` — Get console messages
- `browser_network_requests` — Get network request log
- `browser_wait_for` — Wait for an element or condition
- `browser_run_code` — Execute arbitrary JavaScript on the page
- `browser_hover` — Hover over an element
- `browser_evaluate` — Evaluate JavaScript and return a result
- `browser_tabs` — List open browser tabs
- `browser_close` — Close the browser or a tab

---

**Screenshot naming convention:**

All artifacts are saved to `pepper-v2-app/visual-check-artifacts/`.

Format: `interact-[page-slug]-[step-number]-[description]-[timestamp].png`
- `page-slug`: URL path converted to slug (e.g., `/admin/inbox` becomes `admin-inbox`)
- `step-number`: Two-digit sequential number (`01`, `02`, `03`, ...)
- `description`: Brief description of what was captured (`initial`, `after-click-filter`, `form-filled`, etc.)
- `timestamp`: Unix timestamp for uniqueness

Examples:
- `interact-admin-inbox-01-initial-1710100000.png`
- `interact-admin-inbox-02-after-click-filter-1710100005.png`
- `interact-admin-classification-01-initial-1710100010.png`

---

**Rules:**
- You are **read-only**. Do not modify any source files, configuration, or dependencies. You observe and report only.
- You may create files only in `pepper-v2-app/visual-check-artifacts/` (screenshots).
- If the user asks you to fix something, remind them that `/interact` is read-only. Suggest they run `/code` or `/implement` to make changes, then return to `/interact` to verify.
- Be descriptive in your reports — the user cannot see the browser directly, so your descriptions and screenshots are their only window into the page state.
- If a tool call fails or a page is unresponsive, report the error clearly and ask the user how to proceed rather than silently retrying.
