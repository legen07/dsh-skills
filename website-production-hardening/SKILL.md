---
name: website-production-hardening
version: 1.0.0
category: web-engineering
description: >
  Audits and remediates a website to production-grade quality across
  performance, SEO, accessibility, security, UX, legal compliance,
  and analytics. Produces a verified, launch-ready site.
license: MIT
metadata:
  author: Joe Legen <https://github.com/legen07>
  domain: frontend
  runtime: any
triggers:
  - "harden this website"
  - "make site production ready"
  - "audit and fix website"
  - "prepare site for launch"

---

# Website Production Hardening

Audits and remediates a website to production-grade quality across performance, SEO,
accessibility, security, UX, legal compliance, and analytics. Produces a verified,
launch-ready site.

---

## Inputs

- `repo_path`: string — local path or repo URL
- `staging_url`: string — deployed preview URL
- `brand`: { name, domain, contact_email, phone, address }
- `legal`: { jurisdiction, company_legal_name }
- `analytics`: { provider: "plausible|ga4|umami|matomo", id }

## Outputs

- `remediation_report.md`
- `changed_files.diff`
- `lighthouse_report.html`
- `axe_report.json`
- `sitemap.xml`, `robots.txt`
- `404.html`, `privacy.html`, `terms.html`
- `cookie-consent.js`, `analytics-init.js`

## Tools

- shell, filesystem, git
- playwright — headless browsing, screenshots, link crawling
- lighthouse-ci
- axe-core / pa11y
- linkinator / lychee — broken link check
- html-validate
- npm/pnpm, vite/next/etc (auto-detected)

## Guardrails

- Never commit secrets; rotate any found in history (report only).
- Never modify DNS or production without explicit user confirmation.
- All destructive edits happen on a branch: `chore/prod-hardening`.
- Require human approval before pushing or deploying.

---

## Agent Operating Procedure

The agent executes 10 phases, each with a verify gate. Failures loop back.

### Phase 0 — Recon & Branch

```bash
git checkout -b chore/prod-hardening
```

1. Detect stack (`package.json`, `next.config`, `vite.config`, `astro.config`).
2. Start dev server. Launch Playwright, capture baseline screenshots @ 360 / 768 / 1280 px.
3. Run Lighthouse + axe + link crawler. Persist raw artifacts to `/audit/baseline/`.

**Gate:** baseline reports exist for every route in the sitemap.

### Phase 1 — Broken Links & Navigation

**Tasks**

1. Crawl every route with Playwright; collect all `<a href>` and `fetch()` targets.
2. Classify: `200` | `3xx` | `4xx` | `5xx` | `external-dead` | `mailto/tel` malformed.
3. Fix internal links by rewriting to canonical routes.
4. Remove nav items pointing to non-existent or deprecated routes ("unused navigation").
5. Ensure one canonical `<nav>` per page; mark mobile nav `aria-hidden="true"` when closed.

**Verify**
```bash
npx linkinator ./dist --recurse --skip "linkedin|twitter" --format json > audit/links.json
```

**Gate:** 0 internal 4xx/5xx.

### Phase 2 — Layout: Horizontal Scroll & Mobile Overflow

**Tasks**

1. Inject diagnostic script: find every element where `scrollWidth > clientWidth`.
2. Common culprits & fixes:
   - Fixed width: `1200px` → `max-width: 100%`
   - `100vw` padding issues → `width: 100%` + `box-sizing: border-box`
   - Long unbroken strings → `overflow-wrap: anywhere`
   - Tables → wrap in `.table-scroll`
   - Negative margins on full-bleed sections → `overflow-x: clip` on body
3. Add global rule:
```css
html, body { overflow-x: clip; max-width: 100%; }
img, video, svg { max-width: 100%; height: auto; }
```

**Verify:** Playwright viewport 320px → assert `document.documentElement.scrollWidth <= clientWidth`.

### Phase 3 — Header, Logo, Mobile Menu & Animation

**Tasks**

1. Logo → wrap in `<a href="/" aria-label="{brand} home">`.
2. `tel:` link: `<a href="tel:+{E164}">`.
3. `mailto:` link: `<a href="mailto:{email}">`.
4. Rebuild mobile menu:
   - Use a `<dialog>` or `<div role="dialog" aria-modal="true">`.
   - Focus trap, Esc closes, restore focus to trigger.
   - Animation: `transform: translateX(100%)` → `0` with `cubic-bezier(.2,.8,.2,1)`, 200–280ms.
   - Respect `@media (prefers-reduced-motion: reduce)` → instant.
   - `will-change: transform` only during transition.
   - Lock body scroll with `overflow: hidden` while open; preserve scroll position.
```css
@media (prefers-reduced-motion: reduce) {
  * { animation-duration: .01ms !important; transition-duration: .01ms !important; }
}
```

**Verify:** Playwright — open menu, tab-traps, Esc closes, focus returns. Visual diff < 2% unintended.

### Phase 4 — SEO: Titles, Meta, Sitemap, Robots, 404

**Per-page checklist**

1. Unique `<title>` ≤ 60 chars, format `Primary Keyword | Brand`.
2. `<meta name="description">` 120–158 chars, unique.
3. `<link rel="canonical">`.
4. OpenGraph + Twitter card.
5. `<html lang="en">`.
6. Single `<h1>`.

**Generate**

1. `sitemap.xml` (from route manifest, lastmod from git log).
2. `robots.txt`:
```text
User-agent: *
Allow: /
Disallow: /admin/
Sitemap: https://{domain}/sitemap.xml
```
3. `404.html` — branded, with search + top links, `Cache-Control: no-store`.

**Verify:** html-validate passes; sitemap URL count == route count.

### Phase 5 — Security & Secrets

**Tasks**

1. Scan for secrets: `gitleaks detect --no-banner --redact`.
2. Move any frontend-exposed keys (Stripe secret, OpenAI key, DB URL) to server env.
3. Ensure only `*_PUBLIC_*` / `NEXT_PUBLIC_*` / `VITE_*` vars ship to the client.
4. Rotate exposed keys → open a security ticket.
5. HTTPS enforcement:
   - HSTS header: `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`
   - Redirect http → https (301) at the edge (Netlify/Vercel/Cloudflare/Nginx).
6. Add security headers: Content-Security-Policy, X-Content-Type-Options, Referrer-Policy, Permissions-Policy.

**Gate:** gitleaks clean; `curl -I http://{domain}` returns 301 to https.

### Phase 6 — Forms, Validation, Error Messages, Spam

**Tasks**

1. Client-side validation with clear inline errors (`aria-describedby`, `role="alert"`).
2. Server-side validation duplicates every rule (never trust client).
3. Accessible error pattern:
```html
<label for="email">Email</label>
<input id="email" type="email" required aria-invalid="true" aria-describedby="email-err">
<p id="email-err" role="alert">Please enter a valid email address.</p>
```
4. Spam protection (choose one, prefer layered):
   - Honeypot field (hidden, tabindex="-1", autocomplete="off").
   - Time-to-submit minimum (< 3s = bot).
   - Cloudflare Turnstile (privacy-friendly) or hCaptcha.
   - Rate limit by IP + email.
5. Broken buttons: audit every `<button>` / `<a>` — every one must have type, handler, or href. Dead CTAs get real routes or removed.

**Gate:** Submit empty form → error shown; submit honeypot-filled → silently rejected.

### Phase 7 — Footer, Copyright, Legal Pages

**Tasks**

1. Fix all footer links (same crawler as Phase 1).
2. Copyright: `© {new Date().getFullYear()} {Company}. All rights reserved.` — computed at build time.
3. Create `privacy.html` (or `/privacy`) with: data collected, cookies, third parties, GDPR/CCPA rights, contact.
4. Create `terms.html` with: acceptance, use license, IP, liability, governing law (`{jurisdiction}`).
5. Link both in footer + cookie banner + signup forms.

### Phase 8 — Analytics & Cookie Consent

**Tasks**

1. Cookie consent banner:
   - GDPR-compliant: reject as easy as accept.
   - Categories: necessary / analytics / marketing.
   - Persist choice (localStorage with version key).
   - Block analytics scripts until consent granted.
2. Analytics: inject provider snippet only after consent; use `anonymize_ip` where supported.
```js
// analytics-init.js
if (getConsent('analytics')) {
  loadAnalytics('{provider}', '{id}');
}
```

**Gate:** With consent denied, network tab shows zero analytics requests.

### Phase 9 — Accessibility (WCAG 2.2 AA)

**Tasks**

1. `npx @axe-core/cli {staging_url} --save audit/axe.json`
2. Fix all critical + serious:
   - Color contrast ≥ 4.5:1 (text), 3:1 (large/UI).
   - Every `<img>` has meaningful alt (or `alt=""` if decorative).
   - Every interactive element reachable by keyboard, visible focus ring.
   - Skip-to-content link.
   - Form labels bound to inputs.
   - Landmark regions (header, nav, main, footer).
   - `aria-live` for dynamic content.
3. Screen-reader smoke test with Playwright + `@axe-core/playwright`.

**Gate:** 0 critical/serious violations; keyboard-only traversal completes every flow.

### Phase 10 — Performance

**Tasks**

1. Run Lighthouse mobile + desktop, 3 runs, take median.
2. Targets: LCP < 2.5s, INP < 200ms, CLS < 0.1, Perf ≥ 90, A11y = 100, Best Practices ≥ 95, SEO = 100.
3. Fixes:
   - Images → AVIF/WebP, srcset, `loading="lazy"` (except LCP), explicit width/height.
   - Fonts → `font-display: swap`, preload critical.
   - JS → code-split, tree-shake, defer non-critical.
   - CSS → purge unused, critical inline.
   - Preconnect to third parties; cache headers (immutable for hashed assets).

**Deliverables**

| File | Purpose |
|------|---------|
| `remediation_report.md` | Every issue, severity, fix, verification evidence |
| `changed_files.diff` | Full patch |
| `audit/baseline/*` | Before artifacts |
| `audit/final/*` | After artifacts (Lighthouse, axe, links) |
| `public/404.html`, `privacy.html`, `terms.html` | New pages |
| `public/sitemap.xml`, `public/robots.txt` | SEO |
| `src/lib/consent.ts`, `src/lib/analytics.ts` | Consent + analytics |
| `SECURITY.md` | Secret rotation instructions |

---

## Verification Gate (Definition of Done)

The agent must produce machine-checkable evidence for each:

```yaml
done_when:
  - no_horizontal_scroll: "playwright: scrollWidth <= clientWidth @ 320px on all routes"
  - no_broken_links: "linkinator: 0 internal 4xx/5xx"
  - mobile_menu: "focus trap + esc + focus restore pass"
  - meta_complete: "every route has unique title + description"
  - footer_links_valid: true
  - copyright_current_year: true
  - error_messages_present: "empty-form submit shows aria-live error"
  - no_unused_nav: true
  - clickable_contact: "tel: and mailto: present in header+footer"
  - mobile_optimized: "Lighthouse mobile >= 90 on all routes"
  - legal_pages_exist: ["/privacy", "/terms"]
  - secrets_clean: "gitleaks: 0 findings"
  - https_enforced: "http -> 301 -> https"
  - cookie_consent: "analytics blocked until consent"
  - sitemap_robots: "both served, sitemap valid XML"
  - alt_text: "axe: 0 image-alt violations"
  - perf_targets: "LCP<2.5 INP<200 CLS<0.1"
  - form_validation: "client+server rules match"
  - spam_protection: "honeypot+turnstile active"
  - analytics: "fires only post-consent"
  - a11y: "axe: 0 critical/serious"
```

If any gate fails → loop back to the owning phase. Do not mark complete.

---

## Human-in-the-Loop Checkpoints

The agent pauses for approval before:

1. Rotating secrets / revoking keys.
2. Changing DNS, HSTS preload, or CSP in production.
3. Publishing legal pages (content must be reviewed by counsel).
4. Deploying to production.

---

## Invocation Example

```text
@agent run skill website-production-hardening
  repo_path: ./my-site
  staging_url: https://staging.example.com
  brand: { name: "Acme", domain: "acme.com", contact_email: "hi@acme.com", phone: "+15551234567" }
  legal: { jurisdiction: "California, USA", company_legal_name: "Acme Inc." }
  analytics: { provider: "plausible", id: "acme.com" }
```

Agent responds with the branch, phased execution log, remediation report, and a single PR titled:
`chore: production hardening — SEO, a11y, security, perf, legal, analytics`.
