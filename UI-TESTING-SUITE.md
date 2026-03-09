# UI Testing Suite — Complete Documentation

A comprehensive browser-based testing toolkit for Claude Code. 16 skills, ~482 automated test cases, zero configuration required. Point any skill at a URL and get a structured report with evidence.

**Stack**: Claude Code skills + [Playwright CLI](playwright-cli/skills/playwright-cli/SKILL.md) (accessibility-tree-based browser automation, ~26K tokens/task).

---

## Table of Contents

- [Quick Start](#quick-start)
- [How It Works](#how-it-works)
- [Output & Evidence](#output--evidence)
- [Skills Reference](#skills-reference)
  - [Category A: Functional Testing](#category-a-functional-testing-11-skills)
    - [/test-forms](#test-forms--form-validation--input-testing)
    - [/test-a11y](#test-a11y--accessibility-testing-wcag-21-aa)
    - [/test-responsive](#test-responsive--responsive-design--visual-adaptation)
    - [/test-flows](#test-flows--user-flows-navigation--spa-behavior)
    - [/test-cross-browser](#test-cross-browser--cross-browser-consistency)
    - [/test-security](#test-security--security-vulnerability-testing-owasp)
    - [/test-perf](#test-perf--performance--core-web-vitals)
    - [/test-seo](#test-seo--seo--meta-data)
    - [/test-states](#test-states--ui-states--dynamic-behavior)
    - [/test-links](#test-links--link-integrity--navigation)
    - [/test-regression](#test-regression--regression-detection)
  - [Category B: UX Design Quality](#category-b-ux-design-quality-4-skills)
    - [/test-heatmap](#test-heatmap--attention--interaction-prediction)
    - [/test-ux-writing](#test-ux-writing--micro-copy--content-quality)
    - [/test-consistency](#test-consistency--design-system-consistency)
    - [/test-conversion](#test-conversion--conversion--cognitive-load-analysis)
  - [Orchestrator](#orchestrator)
    - [/test-all](#test-all--full-suite-orchestrator)
- [Recipes & Workflows](#recipes--workflows)
- [Troubleshooting](#troubleshooting)

---

## Quick Start

Every skill follows the same pattern:

```
/test-<name> <url>
```

That's it. No config files, no setup, no flags. Just a URL.

### Your first test (try it now)

```
/test-a11y https://demo.playwright.dev/todomvc/
```

This runs 40 WCAG 2.1 AA+ accessibility checks and produces a report at `test-results/test-a11y/<timestamp>/report.md` with screenshots, snapshots, and traces.

### More quick wins

```
/test-security https://your-staging-app.com
/test-seo https://your-company.com
/test-links https://your-docs-site.com
/test-perf https://your-landing-page.com
```

### Run everything at once

```
/test-all https://your-app.com
```

Runs all 15 skills (~482 test cases) in phased parallel execution.

---

## How It Works

### Architecture

```
You invoke a skill
    |
    v
Skill spawns N parallel subagents (typically 3-5)
    |
    v
Each subagent opens its own named Playwright CLI browser session
    |
    v
Each runs its assigned test cases independently
    |
    v
Results merge into a single report with evidence
```

Each subagent gets an isolated browser session with its own cookies, storage, and history. Sessions are named semantically (e.g., `-s=sec-injection`, `-s=a11y-keyboard`) so you can see exactly what's running.

### Parallelism

Skills maximize throughput by running independent test groups concurrently:

| Skill | Parallel Agents | Why |
|-------|----------------|-----|
| /test-responsive | 4 | One agent per viewport (320, 768, 1024, 1440) |
| /test-cross-browser | 3 | One agent per browser engine (Chromium, Firefox, WebKit) |
| /test-security | 5 | One per attack category (injection, headers, auth, data, client) |
| /test-a11y | 4 | One per concern (structure, keyboard, content, dynamic) |

### What gets tested

Skills use the **accessibility tree** (not screenshots) to analyze page structure. This means tests can detect:
- Missing ARIA labels, broken heading hierarchy, keyboard traps
- Form validation behavior, error message quality, submit states
- Security headers, XSS vectors, exposed secrets
- Computed styles, layout metrics, performance timings

For checks that need visual or runtime data, skills use `run-code` to execute JavaScript directly in the browser via the Playwright API.

---

## Output & Evidence

Every skill writes results to the same directory structure:

```
test-results/
  <skill-name>/
    <YYYYMMDD-HHMMSS>/
      report.md          # Structured test results
      screenshots/        # Visual evidence (PNG)
      snapshots/          # Accessibility tree captures (YAML)
      traces/             # Playwright execution traces
      videos/             # Recorded flows (WebM, when applicable)
```

### Report format

Every report includes:
- **Summary table**: Total / Passed / Failed / Skipped counts
- **Results by category**: Each test case with status, severity, and evidence link
- **Failed test details**: Expected vs actual behavior, with fix suggestions including code examples
- **Evidence index**: Links to all captured artifacts

### Severity ratings

| Level | Meaning | Examples |
|-------|---------|----------|
| **Critical** | Blocks functionality, security risk, data loss | XSS vulnerability, broken login, missing CSRF |
| **Major** | Significant UX/accessibility impact | Missing alt text, broken navigation, no error messages |
| **Minor** | Cosmetic or best-practice deviation | Inconsistent spacing, missing hover states |
| **Info** | Optimization opportunities | Performance hints, SEO suggestions |

---

## Skills Reference

---

## Category A: Functional Testing (11 skills)

*"Does it work correctly?"*

---

### `/test-forms` — Form Validation & Input Testing

**The pain point**: Forms are the primary way users give you data and money. A broken form means lost sign-ups, lost sales, and frustrated users. But forms have dozens of failure modes — invalid input handling, missing error messages, double-submit bugs, file upload edge cases — and manually testing each one is tedious and error-prone.

**What it does**: Runs 27 test cases covering every form interaction pattern — from happy-path submissions to XSS payloads, unicode input, file upload edge cases, and multi-step form state persistence.

**How it helps**: Catches issues like error messages that don't clear after correction, submit buttons that allow double-click, forms that break on special characters, and fields that don't validate on the client side.

**Agents** (4 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Happy path | `forms-happy` | Valid submission, reset, button states, success confirmation |
| Validation | `forms-validation` | Required fields, invalid formats, error display, real-time vs on-submit |
| Boundary | `forms-boundary` | Special chars, unicode, XSS payloads, SQL injection strings, 10K+ char input |
| Advanced | `forms-advanced` | File upload, autocomplete, autofill, Enter key submit, double-click prevention |

**Examples**:

```
# Test your signup form
/test-forms https://myapp.com/signup

# Test a checkout flow
/test-forms https://store.example.com/checkout

# Test a contact form on localhost
/test-forms http://localhost:3000/contact
```

**Smart behavior**: Runs a discovery phase first. If no forms or inputs are found on the page, it reports "No forms found" and skips gracefully — no false failures.

---

### `/test-a11y` — Accessibility Testing (WCAG 2.1 AA+)

**The pain point**: ~15% of users have some form of disability. Inaccessible sites exclude them — and expose you to legal risk (ADA lawsuits have grown 300%+ in recent years). But accessibility has 50+ guidelines, and most teams only check a handful manually.

**What it does**: Runs 40 test cases covering the full WCAG 2.1 AA standard (with some AAA checks), organized across document structure, keyboard navigation, content accessibility, and dynamic content behavior.

**How it helps**: Finds missing ARIA labels, broken tab order, keyboard traps, insufficient color contrast, missing skip navigation, improperly associated form labels, and more — all the things that make your site unusable for screen reader and keyboard users.

**Agents** (4 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Structure | `a11y-structure` | ARIA labels/roles, heading hierarchy, landmarks, page title, lang, duplicate IDs |
| Keyboard | `a11y-keyboard` | Tab order, focus visibility, modal trapping, keyboard traps, dropdown navigation |
| Content | `a11y-content` | Image alt text, color contrast (AA + AAA), form labels, descriptive links, 200% zoom |
| Dynamic | `a11y-dynamic` | Live regions, loading announcements, accordion/tab/carousel patterns, reduced motion |

**Examples**:

```
# Quick accessibility audit on a live site
/test-a11y https://your-company.com

# Check your React app during development
/test-a11y http://localhost:3000

# Audit a specific page with complex interactive widgets
/test-a11y https://app.example.com/dashboard
```

**What you'll learn**: Whether keyboard-only users can navigate your entire site, whether screen readers can understand your page structure, and whether your color choices exclude users with low vision.

---

### `/test-responsive` — Responsive Design & Visual Adaptation

**The pain point**: Your site looks great on your MacBook. But does it work on a 320px phone? A 768px tablet? Does the navigation transform correctly? Do modals fit? Does dark mode break contrast? These are the bugs your users find — on their devices, not yours.

**What it does**: Runs 18 test cases at 4 viewport sizes simultaneously (320x568 mobile, 768x1024 tablet, 1024x768 laptop, 1440x900 desktop), plus dark mode, print styles, RTL layout, landscape orientation, and 200% zoom checks.

**How it helps**: Catches horizontal overflow, tiny touch targets, text truncation, broken modals, sticky elements overlapping content, and images that overflow their containers — at every breakpoint.

**Agents** (4 parallel — one per viewport):
| Agent | Session | Viewport |
|-------|---------|----------|
| Mobile | `resp-320` | 320 x 568 |
| Tablet | `resp-768` | 768 x 1024 |
| Laptop | `resp-1024` | 1024 x 768 |
| Desktop | `resp-1440` | 1440 x 900 |

**Examples**:

```
# Check your landing page at all breakpoints
/test-responsive https://your-startup.com

# Verify a dashboard layout
/test-responsive https://app.example.com/dashboard

# Test a documentation site
/test-responsive https://docs.example.com
```

**Report format**: Matrix view — each test case shown across all 4 viewports, so you can instantly see which breakpoints have issues.

---

### `/test-flows` — User Flows, Navigation & SPA Behavior

**The pain point**: Individual pages might work fine, but real users navigate *between* pages. They log in, browse, go back, refresh, deep-link, and expect their state to persist. SPAs are especially fragile — browser back buttons break, scroll position is lost, deep links fail, and stale data haunts every refresh.

**What it does**: Runs 30 test cases covering authentication chains, registration flows, SPA navigation (back/forward/deep-link/refresh), and data flows (pagination, search, CRUD, optimistic updates).

**How it helps**: Finds broken back-button behavior, lost form state on refresh, failing deep links, missing 404 pages, stale data after mutations, pagination duplicates, and search debounce issues.

**Agents** (4 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Auth | `flows-auth` | Login→dashboard→logout chain, session persistence, remember me, 2FA, lockout |
| Register | `flows-register` | Registration, password reset, profile edit, account deletion |
| Navigation | `flows-navigation` | Back/forward, deep links, 404, breadcrumbs, scroll restoration, hash changes |
| Data | `flows-data` | Pagination, infinite scroll, search + debounce, sort/filter, CRUD cycle |

**Examples**:

```
# Test auth flows with credentials
/test-flows https://app.example.com user@test.com TestPass123!

# Test navigation without auth (auth tests will be skipped gracefully)
/test-flows https://your-blog.com

# Test a SPA's routing behavior
/test-flows http://localhost:3000
```

**Credential handling**: Pass credentials as arguments or set `TEST_USER` / `TEST_PASS` environment variables. If neither is provided, auth tests are **skipped** (not failed) with a note in the report.

---

### `/test-cross-browser` — Cross-Browser Consistency

**The pain point**: "Works on Chrome" is not "works everywhere." Firefox renders some CSS differently. Safari/WebKit has its own quirks with dates, fonts, and scroll behavior. Your users don't all use the same browser you develop in.

**What it does**: Runs 10 test cases simultaneously across Chromium, Firefox, and WebKit. Same tests, same page, three browser engines. Compares structure, forms, navigation, JS features, CSS layout, fonts, animations, and input types.

**How it helps**: Catches browser-specific rendering differences, missing polyfills, broken date pickers in Safari, CSS Grid/Flexbox inconsistencies, and font loading failures — all in one run.

**Agents** (3 parallel — one per browser):
| Agent | Session | Browser |
|-------|---------|---------|
| Chromium | `xb-chromium` | `--browser=chrome` |
| Firefox | `xb-firefox` | `--browser=firefox` |
| WebKit | `xb-webkit` | `--browser=webkit` |

**Examples**:

```
# Cross-browser check your production site
/test-cross-browser https://your-app.com

# Verify a complex form works in all browsers
/test-cross-browser https://app.example.com/settings

# Check CSS layout consistency
/test-cross-browser http://localhost:3000
```

**Report format**: Comparison matrix — each test case shows Chromium / Firefox / WebKit results side by side, with inconsistencies highlighted and suggested fixes (polyfills, vendor prefixes, fallbacks).

---

### `/test-security` — Security Vulnerability Testing (OWASP+)

**The pain point**: Security vulnerabilities cost companies millions. XSS, SQL injection, missing headers, exposed API keys, open redirects — these are the attacks that make the news. But security testing is specialized, expensive, and often deferred until it's too late.

**What it does**: Runs 55 security test cases covering OWASP Top 10 and beyond — injection attacks (XSS, SQL, command, template), HTTP security headers, authentication/session security, data exposure, and client-side vulnerabilities.

**How it helps**: Finds reflected/stored/DOM XSS vectors, missing CSP/HSTS/X-Frame-Options headers, insecure session cookies, exposed API keys in source code, sensitive files accessible via URL, open redirects, and outdated JS libraries with known vulnerabilities.

**Agents** (5 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Injection | `sec-injection` | XSS (reflected/stored/DOM/SVG), SQL, NoSQL, command, HTML, template injection |
| Headers | `sec-headers` | CSP, HSTS, X-Frame-Options, Referrer-Policy, Permissions-Policy, CORS, mixed content |
| Auth | `sec-auth` | CSRF tokens, session cookies (HttpOnly/Secure/SameSite), user enumeration, lockout, JWT storage |
| Data | `sec-data` | Secrets in URLs/localStorage/source, source maps, stack traces, `.env`/`.git` access, PII in comments |
| Client | `sec-client` | Open redirects, clickjacking, window.opener, postMessage, eval/innerHTML, SRI, outdated libraries |

**Examples**:

```
# Security audit your staging environment
/test-security https://staging.your-app.com

# Quick header check on production
/test-security https://your-company.com

# Audit a new deployment
/test-security https://preview-abc123.vercel.app
```

**Important**: This skill is for **authorized security testing** only — your own applications, staging environments, or explicit pentesting engagements. Results are grouped by severity (Critical → Info) so you can triage the most dangerous issues first.

---

### `/test-perf` — Performance & Core Web Vitals

**The pain point**: Slow pages lose users. Google penalizes slow sites in search rankings. But performance problems are insidious — they creep in gradually as you add features, third-party scripts, and unoptimized images. By the time you notice, your LCP is 5 seconds and your CLS is off the charts.

**What it does**: Runs 25 test cases covering Core Web Vitals (LCP, CLS, INP), page weight, DOM complexity, render-blocking resources, runtime performance (long tasks, memory, scroll jank), and asset optimization (image formats, compression, caching, font loading).

**How it helps**: Tells you exactly where your performance budget is spent — which resources are render-blocking, which images lack lazy loading, whether you're serving WebP/AVIF, and whether your CLS is caused by images without explicit dimensions.

**Agents** (3 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Loading | `perf-loading` | LCP, CLS, INP/FID, TTI, page weight, DOM nodes, render-blocking resources |
| Runtime | `perf-runtime` | Long tasks, memory leaks, scroll jank, animation FPS, input responsiveness, lazy loading |
| Assets | `perf-assets` | Image formats, dimensions, compression, caching headers, CSS/JS coverage, font-display, HTTP/2 |

**Examples**:

```
# Audit your landing page performance
/test-perf https://your-startup.com

# Check a content-heavy page
/test-perf https://your-blog.com/long-article

# Verify after optimization work
/test-perf http://localhost:3000
```

**Report format**: Performance scorecard with metrics vs targets (LCP ≤2.5s, CLS ≤0.1, INP ≤200ms, page weight <3MB, DOM nodes <1500).

---

### `/test-seo` — SEO & Meta Data

**The pain point**: You built a great site, but Google can't find it (or ranks it poorly). Missing meta descriptions, broken structured data, no canonical URLs, missing sitemap, poor heading hierarchy — these are silent killers for organic traffic.

**What it does**: Runs 27 test cases covering meta tags (title, description, OG, Twitter Cards), content structure (heading hierarchy, alt text, structured data), and technical SEO (robots.txt, sitemap, status codes, HTTPS, page speed).

**How it helps**: Ensures search engines can discover, crawl, and correctly index your pages. Catches missing or poorly formatted meta tags, broken internal links, noindex on important pages, and missing structured data.

**Agents** (3 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Meta | `seo-meta` | Title (50-60 chars), description (150-160), OG tags, Twitter Cards, canonical, favicon, viewport, lang, hreflang |
| Content | `seo-content` | Single h1, heading hierarchy, image alt text, descriptive anchor text, JSON-LD, breadcrumbs, content ratio |
| Technical | `seo-technical` | robots.txt, sitemap.xml, status codes, noindex check, clean URLs, mobile-friendly, load speed, HTTPS |

**Examples**:

```
# SEO audit your homepage
/test-seo https://your-company.com

# Check a blog post's SEO
/test-seo https://your-blog.com/latest-post

# Verify SEO on a new landing page
/test-seo https://your-startup.com/pricing
```

**Report format**: SEO scorecard (Meta Tags: X/9, Content: X/8, Technical: X/10, Overall: X/27).

---

### `/test-states` — UI States & Dynamic Behavior

**The pain point**: Your app works great when everything goes right. But what happens when the API is slow? When it returns an error? When there's no data? When the user is offline? These "unhappy path" states are where most UX bugs hide — and they're the hardest to test manually because they require simulating failure conditions.

**What it does**: Runs 35 test cases covering loading states (spinners, skeletons, FOUC), empty/error states (404, 500, offline, permission denied), overlays (modals, toasts, tooltips, dropdowns), and animations (smoothness, reduced motion, hover/active states).

**How it helps**: Catches missing loading spinners, empty states without CTAs, error pages that leak stack traces, modals that don't trap focus, toasts that overlap, dropdowns that don't close on outside click, and animations that ignore `prefers-reduced-motion`.

**Agents** (4 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Loading | `states-loading` | Spinners, skeletons, FOUC, CLS during load, progressive loading, optimistic UI, button loading |
| Empty/Error | `states-empty` | Empty states + CTAs, 404, 500 (no stack trace), offline, 403, session expired, partial failure |
| Overlays | `states-overlay` | Modal open/close/focus-trap/escape/backdrop, toast stacking, tooltip keyboard access, dropdown behavior |
| Animation | `states-animation` | Smoothness (jank), reduced motion, transitions, hover/active states, scroll-triggered animations |

**Examples**:

```
# Test all UI states on your dashboard
/test-states https://app.example.com/dashboard

# Check modal and toast behavior
/test-states http://localhost:3000

# Verify error handling on a data-heavy page
/test-states https://app.example.com/reports
```

---

### `/test-links` — Link Integrity & Navigation

**The pain point**: Broken links are embarrassing and hurt SEO. External links rot over time. Internal restructuring creates orphan pages. Links without `rel="noopener"` are a security risk. Links that say "click here" are an accessibility failure. But manually checking every link on a site is mind-numbing.

**What it does**: Runs 23 test cases covering internal link resolution (nav, footer, sidebar, CTAs, logo), external link security and validity, and link behavior (descriptive text, hover/focus states, anchor scrolling, mailto/tel protocols).

**How it helps**: Finds every broken link (internal and external), security issues (missing `noopener noreferrer`), accessibility problems (generic "click here" text), and missing functionality (download attributes, mailto/tel protocols).

**Agents** (3 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Internal | `links-internal` | All internal links resolve (no 404s), nav/footer/sidebar/CTA links work, logo → homepage |
| External | `links-external` | External links open in new tab, have `rel="noopener noreferrer"`, aren't broken, use HTTPS |
| Behavior | `links-behavior` | Descriptive text, visual distinction, hover/focus states, anchor scrolling, download/mailto/tel |

**Examples**:

```
# Check all links on your docs site
/test-links https://docs.example.com

# Audit a marketing site for broken links
/test-links https://your-company.com

# Verify link integrity after a site migration
/test-links https://new-domain.com
```

**Report includes**: Link inventory (total, internal, external, broken, mailto, tel, download).

---

### `/test-regression` — Regression Detection

**The pain point**: You deploy a change. Something else breaks. Maybe a nav link disappeared, maybe the page is 30% slower, maybe a heading changed. Without baseline comparison, regressions slip into production silently.

**What it does**: Two-mode skill. First run captures a **baseline** (accessibility tree, screenshot, performance metrics, element dimensions). Subsequent runs compare the current state against the baseline and flag any regressions.

**How it helps**: Catches structural changes (removed elements), performance degradation (>20% slower), layout shifts (element dimensions ±10%), new console errors, new network failures, and content changes — all automatically compared against your known-good state.

**Test Cases** (8):
| TC | Check | Threshold |
|----|-------|-----------|
| RG1 | Structural diff (accessibility tree) | Any added/removed/changed elements |
| RG2 | Page existence | URL returns 200 |
| RG3 | Critical elements present | Nav links, forms, buttons, headings count |
| RG4 | New console errors | Errors not in baseline |
| RG5 | New network errors | Failed requests not in baseline |
| RG6 | Performance regression | Load time ±20% |
| RG7 | Layout regression | Element dimensions ±10% |
| RG8 | Content regression | Text diff on headings, nav, footer |

**Examples**:

```
# Step 1: Capture baseline on your known-good state
/test-regression https://your-app.com --baseline

# Step 2: After deploying changes, compare against baseline
/test-regression https://your-app.com

# Baseline a staging environment before a release
/test-regression https://staging.your-app.com --baseline
```

**Two-step workflow**: Always `--baseline` first, then run without the flag to compare. Baselines are stored in `test-results/baselines/` and persist across runs.

---

## Category B: UX Design Quality (4 skills)

*"Is it designed well for users?"*

These skills go beyond "does it work" to analyze whether the design effectively serves user goals. They use computational analysis of layout, text, styles, and interaction patterns.

---

### `/test-heatmap` — Attention & Interaction Prediction

**The pain point**: You designed a landing page, but is the CTA actually prominent? Are users' eyes drawn to the right elements? Is your hero image fighting your CTA for attention? You'd normally need actual user testing or expensive eye-tracking studies to answer these questions.

**What it does**: Runs 24 test cases that computationally predict where users will look and click, based on established UX research (F-pattern, Z-pattern, Fitts's Law, Gutenberg diagram). Analyzes element positions, sizes, contrast, and whitespace to score attention distribution.

**How it helps**: Tells you whether your CTA is the most prominent element (or if it's buried), whether you have dead zones with no interactive elements, whether competing CTAs dilute each other, and whether critical content sits in banner-blindness zones.

**Agents** (3 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Layout | `heat-layout` | F/Z-pattern compliance, above-fold %, visual hierarchy scoring, whitespace balance, grid alignment, Gutenberg diagram |
| Interaction | `heat-interaction` | CTA prominence + positioning, Fitts's Law, click target density/dead zones, button hierarchy, element spacing |
| Attention | `heat-attention` | Contrast-based attention, size-based attention, animation distraction, CTA isolation, competing elements, banner blindness |

**Examples**:

```
# Analyze your landing page's visual hierarchy
/test-heatmap https://your-startup.com

# Check CTA placement on a pricing page
/test-heatmap https://your-saas.com/pricing

# Audit a product page for conversion
/test-heatmap https://store.example.com/product/123
```

**Report includes**: Visual hierarchy ranking (top 10 elements by predicted attention), CTA effectiveness score, dead zones, and specific layout adjustment recommendations.

---

### `/test-ux-writing` — Micro-Copy & Content Quality

**The pain point**: Your buttons say "Submit." Your error messages say "Error." Your empty states say nothing. Micro-copy is the most neglected aspect of UX — and it has an outsized impact on conversion, user confidence, and task completion. Bad copy creates confusion; good copy guides users effortlessly.

**What it does**: Runs 26 test cases analyzing every piece of user-facing text — CTA buttons, error messages, success messages, loading text, form labels, placeholders, help text, and terminology consistency.

**How it helps**: Catches generic CTAs ("Submit" instead of "Create Account"), unhelpful errors ("Invalid" instead of "Enter a valid email like name@example.com"), placeholder text used as labels, inconsistent terminology ("Sign in" on one page, "Log in" on another), jargon in user-facing text, and ALL CAPS abuse.

**Agents** (3 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| CTAs | `uxw-cta` | Action verbs, specificity, word count, misleading labels, destructive action warnings, double negatives |
| Feedback | `uxw-feedback` | Error specificity + fix suggestions, plain language, success confirmation, loading text, empty state copy |
| Content | `uxw-content` | Descriptive form labels, placeholder vs label, required field indicators, consistent terminology, jargon, date/number formatting |

**Examples**:

```
# Audit micro-copy on your signup flow
/test-ux-writing https://your-app.com/signup

# Check error messages and feedback
/test-ux-writing https://app.example.com/settings

# Analyze a full marketing site
/test-ux-writing https://your-company.com
```

**Report format**: For each failed test, shows **current text**, **suggested improvement**, and **reasoning** — so you can hand the report directly to a copywriter.

---

### `/test-consistency` — Design System Consistency

**The pain point**: Your buttons are 3 different shades of blue. Your font sizes are a random mix of 13px, 14px, 15px, and 16px. Your cards have different border-radii on different pages. Without a strict design system, visual entropy accumulates until your app looks like it was built by 5 different teams (it probably was).

**What it does**: Runs 30 test cases comparing computed styles across multiple pages — color palette, typography scale, spacing, border-radius, shadows, and component patterns (buttons, inputs, cards, navigation, headings). Discovers pages automatically by crawling internal links.

**How it helps**: Finds every inconsistency in your visual design — 23 unique colors when you should have 12, font sizes that don't follow a scale, buttons that look different on different pages, headers with different heights, and spacing that doesn't follow a grid.

**Agents** (3 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Visual | `cons-visual` | Color count (flag >15), brand colors, font families (max 3), font size/weight scale, spacing scale, border-radius, shadows |
| Components | `cons-components` | Button, input, card, nav, footer, heading, link, alert, avatar, table consistency across pages |
| Pages | `cons-pages` | Content width, header height, sidebar width, padding, backgrounds, loading/error/empty state patterns, animation easing |

**Examples**:

```
# Audit design consistency across your app
/test-consistency https://app.example.com

# Check a marketing site across all pages
/test-consistency https://your-company.com

# Verify design system compliance after a redesign
/test-consistency http://localhost:3000
```

**Report format**: Consistency scorecard (Visual Tokens: X/10, Components: X/10, Cross-Page: X/10). Each inconsistency shows where (which pages/elements), the varying values found, and which value to standardize on.

---

### `/test-conversion` — Conversion & Cognitive Load Analysis

**The pain point**: Users visit your site but don't convert. Is it too many form fields? Too many choices? Not enough trust signals? No progress indicator on your 5-step checkout? Conversion optimization usually requires expensive A/B testing and analytics tools. This skill gives you a baseline assessment for free.

**What it does**: Runs 30 test cases analyzing friction in conversion flows (step count, field count, error recovery), cognitive load (choice overload, visual clutter, information hierarchy), and trust signals (social proof, security indicators, privacy links, clear pricing).

**How it helps**: Tells you exactly how many clicks it takes to complete your primary conversion, whether your forms have too many fields, whether your menus trigger choice paralysis (Miller's Law: >7 items), whether competing CTAs dilute your message, and whether you're missing critical trust signals near conversion points.

**Agents** (3 parallel):
| Agent | Session | Focus |
|-------|---------|-------|
| Friction | `conv-friction` | Steps to conversion, field count, mandatory vs optional, autofill support, progress indicators, guest checkout, error recovery |
| Cognitive | `conv-cognitive` | Choice overload (>7 items), information density, visual clutter, competing CTAs, scannability, progressive disclosure |
| Trust | `conv-trust` | Trust badges, security indicators, social proof, contact info, privacy policy, guarantees, professional quality, clear pricing |

**Examples**:

```
# Analyze conversion friction on your pricing page
/test-conversion https://your-saas.com/pricing

# Audit your signup funnel
/test-conversion https://your-app.com/signup

# Check trust signals on an e-commerce page
/test-conversion https://store.example.com
```

**Report format**: Conversion scorecard (Friction: X/10, Cognitive Load: X/10, Trust Signals: X/10). Includes conversion funnel analysis with estimated drop-off points.

---

## Orchestrator

### `/test-all` — Full Suite Orchestrator

**The pain point**: You want the full picture — not just accessibility, or just security, or just performance, but *everything*. Running 15 skills manually is tedious. And running them all simultaneously would overwhelm your system with ~53 browser sessions.

**What it does**: Orchestrates all 15 skills across 5 sequential phases, each internally parallel. Produces a unified summary report covering all ~482 test cases.

**Execution phases**:

```
Phase 1 (5 parallel):  forms + a11y + responsive + security + links     → 217 tests
Phase 2 (3 parallel):  flows + cross-browser + states                    → 95 tests
Phase 3 (2 parallel):  perf + seo                                        → 52 tests
Phase 4 (4 parallel):  heatmap + ux-writing + consistency + conversion   → 110 tests
Phase 5 (sequential):  regression                                        → 8 tests
                                                                         ─────────
                                                                         ~482 tests
```

**Examples**:

```
# Full audit of your production site
/test-all https://your-company.com

# Comprehensive pre-launch check
/test-all https://staging.your-app.com

# Complete assessment of a new project
/test-all http://localhost:3000
```

**Output**: Unified `test-results/full-suite/<timestamp>/summary.md` with:
- Executive summary table (Category A vs B scores)
- Critical failures highlighted with severity
- Per-skill scorecards with links to individual reports
- Top prioritized recommendations
- Evidence index for all 15 sub-skill artifacts

**Error handling**: If any sub-skill fails, the orchestrator logs the failure and continues with remaining skills. Failed skills show as "ERROR" in the summary.

---

## Recipes & Workflows

### Pre-launch checklist

Run these before any production deployment:

```
# 1. Security first
/test-security https://staging.your-app.com

# 2. Accessibility compliance
/test-a11y https://staging.your-app.com

# 3. Performance budget
/test-perf https://staging.your-app.com

# 4. SEO readiness
/test-seo https://staging.your-app.com
```

### Post-deploy regression check

```
# Baseline before deploy (do this once, on known-good state)
/test-regression https://your-app.com --baseline

# After deploy, compare
/test-regression https://your-app.com
```

### Landing page optimization

```
# Visual hierarchy and CTA effectiveness
/test-heatmap https://your-landing-page.com

# Conversion friction analysis
/test-conversion https://your-landing-page.com

# Content quality
/test-ux-writing https://your-landing-page.com
```

### Design system audit

```
# Check consistency across all pages
/test-consistency https://your-app.com

# Then verify responsive behavior
/test-responsive https://your-app.com
```

### Weekly maintenance scan

```
# Catch link rot and regressions
/test-links https://your-site.com
/test-regression https://your-site.com
```

### Competitor analysis

```
# Understand what competitors do well (UX quality only — no security testing on others' sites!)
/test-heatmap https://competitor.com
/test-ux-writing https://competitor.com
/test-conversion https://competitor.com
```

---

## Troubleshooting

### "No forms found" / "No modals found" / similar skip messages

This is expected behavior. Skills detect whether the page has the relevant elements before testing. If your page doesn't have forms, `/test-forms` reports "No forms found" and skips gracefully. This is not an error.

### Auth tests skipped

`/test-flows` requires credentials. Pass them as arguments or set environment variables:

```
# Via arguments
/test-flows https://app.com user@test.com MyPassword123

# Via environment variables
export TEST_USER="user@test.com"
export TEST_PASS="MyPassword123"
/test-flows https://app.com
```

### Stale browser sessions

If a previous run crashed mid-test, sessions may linger:

```bash
playwright-cli list        # See active sessions
playwright-cli close-all   # Clean them up
playwright-cli kill-all    # Force-kill if close-all doesn't work
```

### Large sites and timeouts

For sites with many pages, some crawl-based tests (links, consistency) may take longer. The skills test up to ~10 discovered pages by default to keep runtime reasonable.

### Output location

All results go to `test-results/` in your current working directory:

```
test-results/
  test-a11y/20260308-143052/report.md
  test-security/20260308-143052/report.md
  full-suite/20260308-143052/summary.md
  baselines/baseline-data.json          # Regression baselines
```

### "This skill is for authorized testing only"

`/test-security` includes injection payloads and vulnerability probes. Only run it against sites you own or have explicit authorization to test. Testing third-party sites without permission may violate computer fraud laws.
