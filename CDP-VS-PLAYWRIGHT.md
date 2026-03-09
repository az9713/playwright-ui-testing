# CDP vs Playwright CLI — Architecture Deep Dive

A technical comparison of two browser automation approaches: raw Chrome DevTools Protocol (CDP) via [chrome-cdp-agent](https://github.com/az9713/chrome-cdp-agent) and the Playwright abstraction via [playwright-cli](https://github.com/anthropics/playwright-cli).

---

## The Big Picture

```
                        ┌─────────────────────────────────┐
                        │         You (Claude Code)        │
                        │    "test my site" / "/test-a11y" │
                        └───────────┬─────────────────────┘
                                    │
                 ┌──────────────────┴──────────────────┐
                 │                                     │
                 ▼                                     ▼
    ┌────────────────────────┐          ┌────────────────────────┐
    │    chrome-cdp-agent    │          │     playwright-cli      │
    │                        │          │                         │
    │  42 CLI commands       │          │  Persistent sessions    │
    │  Stateless (connect    │          │  Named (-s=auth)        │
    │   per command)         │          │  Accessibility tree     │
    │  DOM + screenshots     │          │  run-code for anything  │
    │                        │          │  Tracing & evidence     │
    └───────────┬────────────┘          └───────────┬─────────────┘
                │                                   │
                │ WebSocket                         │ Internal protocol
                │ (raw CDP)                         │ (wraps CDP + more)
                │                                   │
                ▼                                   ▼
    ┌────────────────────────┐          ┌────────────────────────────────┐
    │  Chrome DevTools       │          │     Playwright Engine           │
    │  Protocol (CDP)        │          │                                 │
    │                        │          │  ┌────────┬─────────┬────────┐ │
    │  The raw wire protocol │          │  │CDP shim│ FF shim │WK shim │ │
    │  Chrome exposes on     │          │  │(Blink) │(Gecko)  │(WebKit)│ │
    │  port 9222             │          │  └───┬────┴────┬────┴───┬────┘ │
    └───────────┬────────────┘          └──────┼─────────┼────────┼──────┘
                │                              │         │        │
                ▼                              ▼         ▼        ▼
    ┌────────────────────┐        ┌──────────┐ ┌───────┐ ┌────────┐
    │                    │        │          │ │       │ │        │
    │  Chrome / Chromium │        │ Chromium │ │Firefox│ │ WebKit │
    │                    │        │          │ │       │ │(Safari)│
    │  (your actual      │        │(headless │ │       │ │        │
    │   running browser) │        │ or new)  │ │       │ │        │
    │                    │        │          │ │       │ │        │
    └────────────────────┘        └──────────┘ └───────┘ └────────┘

         DIAGNOSTIC                         AUTOMATION
      ───────────────                    ─────────────────
      "What's happening                 "Does this work
       in this browser                   correctly across
       right now?"                       all browsers?"
```

**CDP is to Playwright what syscalls are to libc** — one is the raw interface, the other is a portable abstraction built on top of it.

---

## What Is CDP?

Chrome DevTools Protocol is the **raw debugging wire protocol** that Chrome/Chromium exposes over WebSocket on port 9222 (by default). It's the same protocol that Chrome DevTools itself uses to inspect pages.

```
Chrome launched with --remote-debugging-port=9222
    │
    ├── HTTP endpoint: http://localhost:9222/json/list    (tab enumeration)
    ├── HTTP endpoint: http://localhost:9222/json/version (browser info)
    │
    └── WebSocket per tab: ws://localhost:9222/devtools/page/<id>
              │
              ├── Page domain      (navigation, screenshots, lifecycle)
              ├── Runtime domain   (JS evaluation, console)
              ├── DOM domain       (node inspection, manipulation)
              ├── Network domain   (request interception, monitoring)
              ├── Input domain     (mouse, keyboard, touch events)
              ├── Emulation domain (device metrics, media, geolocation)
              ├── Performance domain (metrics collection)
              ├── CSS domain       (stylesheet inspection, coverage)
              ├── Accessibility domain (a11y tree)
              └── ... 40+ domains total
```

CDP is **Chrome-only**. Firefox has a partial implementation, Safari has none. The protocol is semi-stable — Google ships breaking changes across Chrome versions.

## What Is Playwright?

Playwright is a **browser automation framework** by Microsoft that provides a unified API across Chromium, Firefox, and WebKit. Under the hood:

- For **Chromium**: Playwright speaks CDP, but through its own patched Chromium build with additional protocol extensions
- For **Firefox**: Playwright uses a custom Juggler protocol (patches applied to Firefox source)
- For **WebKit**: Playwright uses a custom protocol (patches applied to WebKit source)

```
                    Playwright API
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    CDP + patches   Juggler proto   WK proto
          │              │              │
          ▼              ▼              ▼
    Chromium*        Firefox*        WebKit*
    (* = patched builds shipped with Playwright)
```

**playwright-cli** is a CLI wrapper around Playwright that makes it callable from the command line with token-efficient output (accessibility trees instead of screenshots).

---

## Architecture Comparison

### Connection Model

```
chrome-cdp-agent                          playwright-cli
─────────────────                         ──────────────

Command 1:                                Session "auth":
  connect → execute → disconnect            open (session starts)
                                             ↕ persistent connection
Command 2:                                   click e5
  connect → execute → disconnect             fill e3 "text"
                                             snapshot
Command 3:                                   close (session ends)
  connect → execute → disconnect

Each command is independent.              Commands share session state.
No state between commands.                Cookies, storage, history persist.
Browser stays running.                    Browser lifecycle is managed.
```

**CDP agent** is stateless by design. Every command opens a fresh WebSocket connection, does its work, and closes. This is simple and crash-resilient — a failed command doesn't corrupt state. But it means you can't carry context between commands without the browser itself remembering (cookies, DOM state).

**Playwright CLI** manages persistent named sessions. You open a session, run multiple commands within it, and close it. State (cookies, localStorage, navigation history) persists across commands within a session. You can also save and restore auth state across sessions.

### How They "See" the Page

```
chrome-cdp-agent                          playwright-cli
─────────────────                         ──────────────

┌──────────────────────┐                  ┌──────────────────────┐
│     Page.capture     │                  │   Accessibility      │
│     Screenshot()     │                  │   Tree Snapshot      │
│                      │                  │                      │
│  Returns: PNG image  │                  │  Returns: YAML       │
│  (large, visual)     │                  │  (small, structural) │
│                      │                  │                      │
│  + Runtime.evaluate  │                  │  - page              │
│    for DOM queries   │                  │    - heading "Title"  │
│    (text, selectors) │                  │      [ref: e1]       │
│                      │                  │    - textbox "Email"  │
│                      │                  │      [ref: e3]       │
│                      │                  │    - button "Submit"  │
│                      │                  │      [ref: e5]       │
└──────────────────────┘                  └──────────────────────┘

Element selection:                        Element selection:
  click 0 "Submit Button"                   click e5
  (by visible text + tab index)             (by accessibility ref ID)
```

**CDP agent** sees the page through **screenshots and DOM queries**. It captures PNGs via `Page.captureScreenshot()` and extracts text via `Runtime.evaluate()` running JavaScript in the page context. Elements are selected by visible text content or CSS selectors. This is intuitive but token-expensive — a screenshot can be thousands of tokens.

**Playwright CLI** sees the page through **accessibility tree snapshots**. These are YAML representations of the page's semantic structure — the same tree screen readers use. Elements get ref IDs (e.g., `e5`) that you use in subsequent commands. This is ~26K tokens per page — far cheaper than screenshots, and it captures semantic meaning (roles, labels, states) that screenshots miss.

### Element Interaction

```
chrome-cdp-agent                          playwright-cli
─────────────────                         ──────────────

Click flow:                               Click flow:
1. Runtime.evaluate() to find             1. Reference e5 from snapshot
   element by text content                2. Playwright locates element
2. Walk DOM to nearest clickable          3. Auto-waits for actionable
   ancestor                               4. Scrolls into view
3. getBoundingClientRect()                5. Clicks center point
4. Input.dispatchMouseEvent()
   at center coordinates                  One step: click e5

Fill flow:                                Fill flow:
1. Find input by name/id/label            1. Reference from snapshot
2. Focus via click                        2. fill e3 "value"
3. Ctrl+A to select all                      (auto-focuses, clears, types)
4. Input.insertText()

Low-level, explicit, manual.              High-level, auto-waiting, semantic.
```

**CDP agent** does everything manually — find element via JS, calculate coordinates, dispatch synthetic input events. This gives you total control but requires more steps and is fragile if the page layout changes.

**Playwright CLI** uses Playwright's smart selectors with auto-waiting. When you say `click e5`, Playwright waits for the element to be visible, enabled, and stable, scrolls it into view, then clicks. This is more resilient to timing issues and dynamic content.

### Session & Parallelism

```
chrome-cdp-agent                          playwright-cli
─────────────────                         ──────────────

Your Chrome (port 9222)                   Session "a11y-keyboard"
├── Tab 0  ◄── command targets by         ├── Isolated browser context
├── Tab 1      tab index                  ├── Own cookies, storage
├── Tab 2                                 └── Own navigation history
└── Tab 3
                                          Session "a11y-content"
Multiple commands can hit                 ├── Separate browser context
the same Chrome instance.                 ├── Completely isolated
No isolation between them.                └── Can run in parallel

Parallel: possible but                    Parallel: first-class support
  no isolation guarantees.                  via named sessions (-s=).
  Commands share cookies,                   Each session is a separate
  storage, everything.                      browser context.
```

**CDP agent** connects to **your existing Chrome**. All commands share the same browser state — cookies, localStorage, everything. You can run commands in parallel (separate processes), but they're not isolated. Changing a cookie in one command affects all others.

**Playwright CLI** creates **its own browser instances** with isolated contexts per named session. `-s=auth` and `-s=public` have completely separate cookies, storage, and history. This is essential for parallel test execution — the ui-testing suite runs up to 5 parallel sessions per skill.

---

## Capability Matrix

### Unique to chrome-cdp-agent

| Command | What It Does | Why CDP Excels Here |
|---------|-------------|---------------------|
| `waterfall` | Network request timing waterfall | Direct access to Network domain timing data |
| `dead-code` | Find unused CSS/JavaScript | Uses CSS.startRuleUsageTracking + JS coverage |
| `dom-history` | Track DOM mutations over time | Real-time MutationObserver via CDP |
| `tokens` | Extract design tokens (colors, fonts, spacing) | Deep computed style inspection |
| `layout` | Visualize CSS layout structure with dimensions | Direct DOM + CSSOM access |
| `animation` | Pause/resume/slow animations | Animation domain control |
| `highlight` | Highlight elements by selector | Overlay domain |
| `record` | Screencast frames as PNG sequence | Page.screencastFrame events |
| `responsive` | Multi-breakpoint screenshots in one command | Emulation.setDeviceMetricsOverride |
| `diff` | Compare two pages structurally | Side-by-side tab analysis |
| `emulate` | Device emulation (iPhone14, iPad, Pixel7) | Emulation domain presets |
| `mock-api` | Intercept and mock API responses | Network.requestIntercepted |
| `design-review` | Extract full page structure for review | DOM + CSSOM combined |

### Unique to playwright-cli

| Command | What It Does | Why Playwright Excels Here |
|---------|-------------|----------------------------|
| `snapshot` | Accessibility tree as YAML | Built-in a11y tree extraction |
| `-s=<name>` | Named isolated sessions | Browser context isolation |
| `state-save/load` | Persist/restore auth state | Context-level storage state |
| `tracing-start/stop` | Full execution trace with DOM snapshots | Playwright tracing infrastructure |
| `video-start/stop` | Record session as WebM | Built-in video recording |
| `--browser=firefox/webkit` | Cross-browser testing | Multi-browser engine support |
| `run-code` | Execute arbitrary Playwright code | Full Playwright API access |
| `route` | Mock network requests persistently | Context-level request interception |
| `tab-new/select/close` | Multi-tab management within session | Tab lifecycle within context |

### Shared Capabilities

Both can do these, but via different mechanisms:

| Capability | CDP Agent | Playwright CLI |
|-----------|-----------|----------------|
| Navigate | `open 0 <url>` | `goto <url>` |
| Click | `click 0 "text"` | `click e5` |
| Type | `type 0 "text"` | `fill e5 "text"` |
| Screenshot | `screenshot 0` | `screenshot` |
| Evaluate JS | `eval 0 "expr"` | `eval "expr"` |
| Read console | `console 0` | `console` |
| Monitor network | `network 0` | `network` |
| Cookies | `cookies 0` | `cookie-list` |
| Back/forward | `back 0` | `go-back` |
| Page content | `content 0` | `snapshot` |

---

## Token Economy

This matters when an LLM is driving the browser — every byte of output consumes context window.

```
chrome-cdp-agent output:                  playwright-cli output:

$ screenshot 0                            $ snapshot
→ screenshot-12345.png (200KB)            → page-snapshot.yaml (5KB)
  ≈ 1500+ tokens as base64                 ≈ 200 tokens as text
  (or reference to file)

$ elements 0                              (included in snapshot above)
→ JSON array of all interactive           → YAML tree with ref IDs
  elements with bounding boxes              and semantic roles
  ≈ 500-2000 tokens                         already included

$ content 0                               (included in snapshot above)
→ Full visible text content               → Text content embedded
  ≈ 500-5000 tokens                         in tree structure

Total for "understand the page":          Total for "understand the page":
  screenshot + elements + content           snapshot
  ≈ 2500-8500 tokens                        ≈ 200 tokens
```

**Playwright CLI is dramatically more token-efficient** for LLM-driven automation. The accessibility tree snapshot gives the LLM everything it needs — structure, text content, interactive elements with ref IDs — in a compact YAML format. CDP agent's approach of screenshots + DOM queries is richer visually but costs 10-40x more tokens.

---

## When to Use Each

### Use chrome-cdp-agent when:

1. **Inspecting your actual running browser** — you have Chrome open with tabs you want to analyze, not a test page you want to automate
2. **Real-time diagnostics** — network waterfalls, DOM mutation tracking, performance profiling, animation debugging
3. **Design analysis** — extracting design tokens, visualizing layout structure, finding dead CSS/JS, doing design reviews
4. **Quick one-off commands** — fire a single command, get a result, done. No session setup needed
5. **CI/shell scripting** — stateless commands compose well in bash scripts
6. **You only need Chrome** — cross-browser isn't a requirement

### Use playwright-cli when:

1. **Automated test suites** — structured, repeatable testing with evidence capture
2. **Cross-browser testing** — need to verify Firefox and Safari/WebKit behavior
3. **Parallel isolated sessions** — multiple independent browsers running simultaneously
4. **LLM-driven automation** — token-efficient accessibility tree output
5. **Auth-dependent testing** — save and restore login state across test runs
6. **Evidence capture** — tracing, screenshots, snapshots, video recording as structured artifacts
7. **Complex multi-step flows** — persistent sessions that maintain state across many commands

### Use both together when:

1. **Pre-launch audit**: Run playwright-ui-testing suite for structured pass/fail testing, then use CDP agent for deep diagnostics on failures (waterfall on slow pages, dead-code on heavy pages, layout analysis on broken layouts)
2. **Design review workflow**: Use CDP `tokens` and `design-review` to extract the current design system, then use `/test-consistency` to verify it's applied consistently
3. **Performance investigation**: Use `/test-perf` to identify performance issues, then use CDP `waterfall`, `perf`, and `memory` for root cause analysis

---

## Protocol-Level Comparison

### CDP Command Example (raw)

```javascript
// What chrome-cdp-agent does internally for a click:

// 1. Connect to Chrome
const client = await CDP({ port: 9222, target: tabId });

// 2. Find element by text
const { result } = await client.Runtime.evaluate({
  expression: `
    const walker = document.createTreeWalker(
      document.body, NodeFilter.SHOW_TEXT
    );
    while (walker.nextNode()) {
      if (walker.currentNode.textContent.includes("Submit")) {
        let el = walker.currentNode.parentElement;
        while (el && !el.matches('a, button, [role=button], input')) {
          el = el.parentElement;
        }
        if (el) {
          const rect = el.getBoundingClientRect();
          return JSON.stringify({
            x: rect.x + rect.width / 2,
            y: rect.y + rect.height / 2
          });
        }
      }
    }
    return null;
  `
});

// 3. Dispatch mouse events at coordinates
const { x, y } = JSON.parse(result.value);
await client.Input.dispatchMouseEvent({
  type: 'mousePressed', x, y, button: 'left', clickCount: 1
});
await client.Input.dispatchMouseEvent({
  type: 'mouseReleased', x, y, button: 'left', clickCount: 1
});

// 4. Disconnect
await client.close();
```

### Playwright CLI Equivalent

```bash
# What playwright-cli does for the same click:
playwright-cli click e5
# That's it. Playwright handles:
# - Finding the element by ref ID from accessibility tree
# - Waiting for it to be visible, enabled, stable
# - Scrolling into view
# - Computing click coordinates
# - Dispatching input events
# - Returning updated page snapshot
```

### The Abstraction Trade-off

```
                Control
                   ▲
                   │
    CDP agent ─────┤  You control every mouse event,
                   │  every JS evaluation, every pixel.
                   │  Maximum power, maximum responsibility.
                   │
                   │
                   │
                   │
 Playwright CLI ───┤  Framework handles waiting, scrolling,
                   │  event dispatch, error recovery.
                   │  Less control, more reliability.
                   │
                   └──────────────────────────► Convenience
```

**CDP = maximum control**. You decide when to wait, what to scroll, how to dispatch events. This is powerful for diagnostics and edge cases, but fragile for automation — a timing issue or layout change breaks your script.

**Playwright = maximum convenience**. The framework handles the hard parts (waiting, retrying, scrolling, cross-browser differences). You give up some control but gain reliability and portability.

---

## Summary Table

| Dimension | chrome-cdp-agent | playwright-cli |
|-----------|-----------------|----------------|
| **Protocol** | Raw CDP over WebSocket | Playwright internal (wraps CDP + custom) |
| **Browsers** | Chrome/Chromium only | Chromium + Firefox + WebKit |
| **Connection** | Stateless per-command | Persistent named sessions |
| **Target browser** | Your running Chrome | Its own managed instances |
| **Page understanding** | Screenshots + DOM queries | Accessibility tree (YAML) |
| **Element selection** | Text content / CSS selectors | Ref IDs from a11y tree (e5) |
| **Token cost** | High (screenshots/DOM dumps) | Low (~26K tokens/task) |
| **Auto-waiting** | None (manual) | Built-in (wait for actionable) |
| **Parallelism** | Possible but not isolated | First-class isolated sessions |
| **Session state** | Browser-level (shared) | Context-level (isolated) |
| **Evidence capture** | Screenshots only | Traces + screenshots + snapshots + video |
| **Auth state** | Manual cookie management | state-save / state-load |
| **Network mocking** | Ephemeral (10s window) | Persistent route interception |
| **Diagnostics** | Excellent (waterfall, dead-code, mutations, perf) | Basic (console, network) |
| **Best metaphor** | Stethoscope (real-time diagnosis) | Lab test suite (comprehensive analysis) |
| **Commands** | 42 specialized | Fewer but `run-code` for anything |
| **Crash recovery** | Automatic (stateless) | Session may need cleanup |
| **Learning curve** | Lower (simple CLI commands) | Lower (simple CLI commands) |
| **Best for** | Inspection, diagnostics, design analysis | Testing, automation, cross-browser |
