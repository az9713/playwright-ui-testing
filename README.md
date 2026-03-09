# Playwright UI Testing Suite

Comprehensive browser-based UI testing toolkit for [Claude Code](https://claude.ai/claude-code). 16 skills, ~482 automated test cases, zero configuration. Point any skill at a URL and get a structured report with evidence.

Powered by [Playwright CLI](https://github.com/anthropics/playwright-cli) — a token-efficient browser automation CLI that uses accessibility trees instead of screenshots (~26K tokens/task).

## Install

Copy the `.claude/skills/playwright-ui-testing/` directory into your project's `.claude/skills/` folder. That's it — the skills are immediately available as slash commands.

```bash
# Clone and copy into your project
git clone https://github.com/az9713/playwright-ui-testing.git
cp -r playwright-ui-testing/.claude/skills/playwright-ui-testing your-project/.claude/skills/
```

### Prerequisites

- [Claude Code](https://claude.ai/claude-code) CLI
- [Playwright CLI](https://github.com/anthropics/playwright-cli) installed globally or via npx

## Quick Start

```
/test-a11y https://your-site.com
```

That's it. No config files, no setup, no flags. Just a URL.

## Available Skills

### Category A: Functional Testing

| Skill | Command | Tests | What It Checks |
|-------|---------|------:|----------------|
| Forms | `/test-forms <url>` | 27 | Validation, input handling, submit states, file uploads |
| Accessibility | `/test-a11y <url>` | 40 | WCAG 2.1 AA+ compliance, keyboard nav, screen readers |
| Responsive | `/test-responsive <url>` | 72 | 4 viewports, dark mode, print, RTL, zoom |
| Flows | `/test-flows <url>` | 30 | Auth, registration, SPA navigation, CRUD |
| Cross-Browser | `/test-cross-browser <url>` | 30 | Chromium, Firefox, WebKit consistency |
| Security | `/test-security <url>` | 55 | OWASP+ (XSS, injection, headers, auth, data exposure) |
| Performance | `/test-perf <url>` | 25 | Core Web Vitals, runtime perf, asset optimization |
| SEO | `/test-seo <url>` | 27 | Meta tags, structured data, crawlability |
| States | `/test-states <url>` | 35 | Loading, empty, error, modals, toasts, animations |
| Links | `/test-links <url>` | 23 | Broken links, external link security, link semantics |
| Regression | `/test-regression <url>` | 8 | Baseline comparison, structural/perf/content diff |

### Category B: UX Design Quality

| Skill | Command | Tests | What It Checks |
|-------|---------|------:|----------------|
| Heatmap | `/test-heatmap <url>` | 24 | Visual hierarchy, CTA placement, attention scoring |
| UX Writing | `/test-ux-writing <url>` | 26 | Micro-copy quality, error messages, terminology |
| Consistency | `/test-consistency <url>` | 30 | Design tokens, component patterns, cross-page uniformity |
| Conversion | `/test-conversion <url>` | 30 | Friction analysis, cognitive load, trust signals |

### Orchestrator

| Skill | Command | Description |
|-------|---------|-------------|
| Full Suite | `/test-all <url>` | Runs all 15 skills in 5 phased waves (~482 tests) |

## How It Works

```
You invoke a skill ──> Skill spawns 3-5 parallel subagents
                           │
                           ├── Agent 1: opens browser session, runs tests
                           ├── Agent 2: opens browser session, runs tests
                           └── Agent N: opens browser session, runs tests
                                          │
                                          v
                              Merged report with evidence
```

Each subagent gets an isolated Playwright CLI browser session with its own cookies, storage, and history. Skills maximize parallelism — `/test-responsive` runs 4 viewports simultaneously, `/test-security` runs 5 attack categories in parallel.

## Output

Every skill writes to the same structure:

```
test-results/<skill-name>/<timestamp>/
  report.md          # Structured results with pass/fail, severity, fix suggestions
  screenshots/       # Visual evidence (PNG)
  snapshots/         # Accessibility tree captures (YAML)
  traces/            # Playwright execution traces
  videos/            # Recorded flows (WebM)
```

### Severity Ratings

| Level | Meaning |
|-------|---------|
| Critical | Blocks functionality, security risk, data loss |
| Major | Significant UX/accessibility impact |
| Minor | Cosmetic or best-practice deviation |
| Info | Optimization opportunities |

## Examples

```bash
# Pre-launch security audit
/test-security https://staging.your-app.com

# Accessibility compliance check
/test-a11y https://your-company.com

# Landing page optimization
/test-heatmap https://your-startup.com
/test-conversion https://your-startup.com
/test-ux-writing https://your-startup.com

# Post-deploy regression check
/test-regression https://your-app.com --baseline    # capture known-good state
/test-regression https://your-app.com               # compare after deploy

# Full site audit (~482 tests)
/test-all https://your-app.com
```

## Natural Language — Just Ask

You don't have to memorize slash commands. Claude Code understands intent and will invoke the right skill automatically. Here are some prompts you can try:

### Security & Compliance

> "Check my staging site for security vulnerabilities"
>
> "Is my site vulnerable to XSS? Test https://staging.myapp.com"
>
> "Run a security audit on https://preview-abc123.vercel.app before we go live"
>
> "Make sure our login page doesn't have any OWASP issues"

### Accessibility

> "Is https://myapp.com accessible for screen reader users?"
>
> "Check if keyboard-only users can navigate my site"
>
> "Run WCAG compliance tests on our dashboard"
>
> "We're getting ADA complaints — audit https://our-company.com for accessibility"

### Performance

> "Why is our landing page so slow? Test https://our-startup.com"
>
> "Check the Core Web Vitals on my blog"
>
> "Are we passing Google's page speed requirements?"

### Responsive Design

> "Does our site work on mobile? Test https://our-app.com"
>
> "Check if the dashboard breaks on small screens"
>
> "Test dark mode and print styles on our marketing site"

### Forms & User Flows

> "Test all the forms on our signup page"
>
> "Can users actually complete the checkout flow? Test it end-to-end"
>
> "Try submitting our contact form with weird characters and see what breaks"
>
> "Test the login → dashboard → settings → logout flow on staging"

### SEO

> "Is our homepage optimized for search engines?"
>
> "Check if we have the right meta tags and structured data"
>
> "Audit the SEO on our new blog post"

### Links

> "Find all the broken links on our docs site"
>
> "Check if our external links are still valid"
>
> "We just migrated domains — make sure nothing is broken"

### UX Quality

> "Is our CTA prominent enough on the landing page?"
>
> "Audit the button labels and error messages on our app — are they clear?"
>
> "Check if our design is consistent across pages"
>
> "How many clicks does it take to sign up? Analyze the conversion funnel"

### Regression

> "Save a baseline of our production site before we deploy"
>
> "We just deployed — did anything break compared to the baseline?"

### Full Audit

> "Run every test you have against https://our-app.com"
>
> "Give me a complete quality assessment of our staging site"
>
> "We're launching next week — do a full audit"

## Documentation

See [UI-TESTING-SUITE.md](UI-TESTING-SUITE.md) for complete documentation including:
- Detailed skill descriptions with pain points and how each helps
- Agent breakdown and parallelism strategy per skill
- Recipes and workflows (pre-launch, post-deploy, landing page optimization, etc.)
- Troubleshooting guide

## Project Structure

```
.claude/skills/playwright-ui-testing/
  SKILL.md                          # Master index
  references/
    common-setup.md                 # Shared setup patterns
    evidence-capture.md             # Tracing, screenshots, snapshots
    report-format.md                # Report template, severity ratings
  test-forms/SKILL.md               # 27 form validation tests
  test-a11y/SKILL.md                # 40 accessibility tests
  test-responsive/SKILL.md          # 72 responsive design tests
  test-flows/SKILL.md               # 30 user flow tests
  test-cross-browser/SKILL.md       # 30 cross-browser tests
  test-security/SKILL.md            # 55 security tests
  test-perf/SKILL.md                # 25 performance tests
  test-seo/SKILL.md                 # 27 SEO tests
  test-states/SKILL.md              # 35 UI state tests
  test-links/SKILL.md               # 23 link integrity tests
  test-regression/SKILL.md          # 8 regression tests
  test-heatmap/SKILL.md             # 24 attention prediction tests
  test-ux-writing/SKILL.md          # 26 micro-copy tests
  test-consistency/SKILL.md         # 30 design consistency tests
  test-conversion/SKILL.md          # 30 conversion optimization tests
  test-all/SKILL.md                 # Full suite orchestrator
```

## Acknowledgements

This project was inspired by [Claude Code + Playwright = INSANE Browser Automations](https://www.youtube.com/watch?v=I9kO6-yPkfM&t=40s) by [Chase AI](https://www.youtube.com/@_chase_ai).

## License

MIT
