# Manual web design & debugging (Playwright MCP)

Referenced from CLAUDE.md → Web Design & Debugging. This is the **fallback** for when the dedicated skills (`/verify`, `run`, `/pr-ui-test`, `/web-perf`) don't fit or the Playwright MCP `browser_*` tools aren't loaded. Prefer the skills first.

## Web design by hand (CSS, HTML, layouts)

1. MUST use `browser_navigate` to open the page
2. MUST use `browser_snapshot` to get DOM structure (preferred for understanding layout)
3. MUST use `browser_take_screenshot` to capture visual state
4. Make code changes
5. MUST refresh and verify with screenshot

## Debugging web behaviour (UI bugs, runtime errors, failing flows)

- MUST drive the page while debugging — actually run the broken flow, don't speculate from the source. Click / type / submit through the steps the user described, observe the live result, then iterate on the fix.
- MUST check `browser_console_messages` for runtime errors / warnings before assuming the UI is "fine"
- MUST use `browser_network_requests` to inspect API calls / responses when the bug involves data flow
- After a fix, MUST re-drive the same flow end-to-end to confirm the regression is gone — never just "looks right in snapshot"

## Useful Playwright MCP tools

- `browser_resize` — Test responsive design at different breakpoints
- `browser_evaluate` — Inspect computed styles / state via JavaScript
- `browser_console_messages` — Read runtime console output
- `browser_network_requests` — Inspect HTTP traffic
