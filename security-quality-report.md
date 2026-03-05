# Code Quality & Security Report

**File:** `Ageing Well in Later Life_Full Profile_HTML.html`  
**Generator:** Quarto 1.8.25  
**File size:** 22,804 lines  
**Document type:** Static self-contained HTML report  
**Report date:** 2026-03-05

---

## Embedded Third-Party Libraries

| Library             | Version | How Embedded                                |
| ------------------- | ------- | ------------------------------------------- |
| clipboard.js        | 2.0.11  | Inline `<script>`                           |
| TypedArray polyfill | —       | Inline `<script>`                           |
| jQuery              | 3.5.1   | Inline `<script>`                           |
| Popper.js           | 2.11.7  | Inline `<script>`                           |
| axe-core            | 4.10.3  | Dynamic CDN `import()` inside base64 module |

---

## Findings Summary

| Severity  | Finding                                     | Actionable                            |
| --------- | ------------------------------------------- | ------------------------------------- |
| 🔴 HIGH   | No Content-Security-Policy                  | Yes — server config or Quarto YAML    |
| 🔴 HIGH   | axe-core CDN import without SRI             | Yes — Quarto config / disable in prod |
| 🔴 HIGH   | `innerHTML` in 15 library locations         | Low risk — all trusted library code   |
| 🟠 MEDIUM | No clickjacking protection                  | Yes — server config                   |
| 🟠 MEDIUM | 6 images missing `alt` attribute (WCAG 2.1) | Yes — Quarto source `.qmd`            |
| 🟡 LOW    | `document.execCommand()` deprecated         | Quarto upstream                       |
| 🟡 LOW    | Page title is a filename slug               | Yes — Quarto YAML                     |
| 🟡 LOW    | Missing `<meta name="description">`         | Yes — Quarto YAML                     |
| ℹ️ INFO   | 2× `fetch()` calls to `listings.json`       | No action needed                      |

---

## Detailed Findings

### 🔴 HIGH — No Content-Security-Policy

No `<meta http-equiv="Content-Security-Policy">` tag is present. Without CSP, the browser has no enforced policy against XSS: inline scripts run freely and any external origin can be loaded.

**Recommendation:** Set a CSP via your web server's response headers (preferred) or in Quarto's `_quarto.yml`:

```yaml
format:
  html:
    include-in-header:
      - text: '<meta http-equiv="Content-Security-Policy" content="default-src ''self''; script-src ''self'' ''unsafe-inline'';">'
```

> Note: `'unsafe-inline'` is required because of the large volume of inline scripts Quarto generates. A stronger mitigation would be to use nonces or hashes if Quarto supports it in a future version.

---

### 🔴 HIGH — axe-core Loaded from External CDN Without Subresource Integrity (SRI)

Inside a base64-encoded ES module, axe-core is dynamically imported at runtime with no integrity check:

```javascript
await import("https://cdn.skypack.dev/pin/axe-core@v4.10.3-.../axe-core.js");
```

If `cdn.skypack.dev` is compromised or serves tampered content, arbitrary JavaScript will execute in the page context with full DOM access.

**Recommendation:** This is Quarto-generated behaviour. Options:

1. Upgrade Quarto to a version that supports self-hosted axe-core or SRI on the CDN import.
2. Disable the axe accessibility checker on public-facing production builds (it is a development/QA tool):
   ```yaml
   format:
     html:
       a11y: false # or equivalent Quarto option
   ```
3. If the document is served from a controlled server, block outbound requests to `cdn.skypack.dev` via CSP:
   ```
   Content-Security-Policy: script-src 'self' 'unsafe-inline';
   ```
   This prevents the CDN import from loading at all in production.

---

### 🔴 HIGH — `innerHTML` Assigned in 15 Locations

`innerHTML` is used on 15 lines across the embedded third-party libraries. Flagged lines include:

- **clipboard.js** — reads existing DOM node content to perform clipboard copy
- **jQuery 3.5.1** — `Ut.innerHTML = "<form></form><form></form>"` is a static browser feature-detection string inside `createHTMLDocument` (not user content)
- **Quarto framework** — reads tab labels from existing DOM nodes for tab-sync logic

**Actual risk: Low.** This is a static document with no server-rendered user content. None of the `innerHTML` assignments write attacker-controlled values. No immediate action required. If the Quarto source is later extended to embed dynamic user content (e.g. via R `!expr` or Shiny), reassess at that time.

> **Note:** The `<form>` string found at line 4033 is a confirmed **false positive**. It is the literal string `"<form></form><form></form>"` used inside jQuery's `y.createHTMLDocument` browser capability test — it is not an actual HTML form element.

---

### 🟠 MEDIUM — No Clickjacking Protection

The document has no `X-Frame-Options` header and no `frame-ancestors` CSP directive. It could be embedded in an `<iframe>` on an attacker-controlled page to perform UI-redress (clickjacking) attacks.

**Recommendation:** Set at the web server level:

```
X-Frame-Options: SAMEORIGIN
```

Or preferably via CSP (more flexible and future-proof):

```
Content-Security-Policy: frame-ancestors 'self'
```

---

### 🟠 MEDIUM — 6 Images Missing `alt` Attribute (WCAG 2.1 AA)

Six base64-encoded PNG chart images lack `alt` text. Screen readers will announce these as unlabelled images. This violates WCAG 2.1 Success Criterion 1.1.1 (Non-text Content), which is a Level AA requirement.

Affected elements are `<img role="img" src="data:image/png;base64,...">` without `alt=` attributes.

**Recommendation:** In the Quarto source (`.qmd`), add descriptive alt text to each chart using the `fig-alt` chunk option:

```r
#| fig-alt: "Bar chart showing percentage of population aged 65+ reporting good health by county, 2022."
```

For purely decorative images, use an empty alt attribute to explicitly mark them as decorative:

```r
#| fig-alt: ""
```

---

### 🟡 LOW — `document.execCommand()` Deprecated

clipboard.js v2.0.11 uses `document.execCommand('copy')` as a fallback for older browsers. This API is deprecated in all major browsers and may be removed in future releases.

**Recommendation:** This is inside Quarto's bundled clipboard.js. The primary code path already uses the modern `navigator.clipboard.writeText()` API — `execCommand` is only a fallback for legacy browsers. Monitor Quarto releases for an updated clipboard.js bundle. No urgent action required for modern browser targets.

---

### 🟡 LOW — Page `<title>` Is a Filename Slug

```html
<title>ageing-well-in-later-life_full-profile_html</title>
```

This is the raw filename converted to a URL slug. Screen readers announce the `<title>` when a page loads, providing a poor experience for assistive technology users. It also produces weak metadata if the document is ever shared or indexed.

**Recommendation:** Set a human-readable title in the Quarto YAML front matter:

```yaml
title: "Ageing Well in Later Life – Full Profile"
```

---

### 🟡 LOW — Missing `<meta name="description">`

No description meta tag is present in the document `<head>`. While this document may not be publicly indexed, a description improves shareability and browser bookmark previews.

**Recommendation:** Add a description in the Quarto YAML front matter:

```yaml
description: "HSE national health profile report covering key indicators for the population aged 65+, including physical health, mental wellbeing, and social participation."
```

---

### ℹ️ INFO — `fetch()` Calls to `listings.json`

Two `fetch()` calls are present (fetching `listings.json`). This is Quarto's built-in listing/category navigation feature used to activate breadcrumb category links. The URLs are derived from page-relative metadata, not user input.

**No action required.**

---

## Remediation Priority

| Priority | Action                                                                | Effort                                 |
| -------- | --------------------------------------------------------------------- | -------------------------------------- |
| 1        | Add `X-Frame-Options` and `Content-Security-Policy` headers at server | Low — server config change             |
| 2        | Disable axe-core CDN import in production builds                      | Low — Quarto config change             |
| 3        | Add `alt` text to chart images in `.qmd` source                       | Medium — requires reviewing each chart |
| 4        | Set human-readable `title` and `description` in Quarto YAML           | Low — 2-line YAML change               |
| 5        | Monitor Quarto/clipboard.js for `execCommand` deprecation             | Deferred — watch Quarto releases       |

---

## False Positives Excluded

| Finding                             | Reason Excluded                                                                                                                                                                                  |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `<form>` CSRF concern (line 4033)   | Confirmed false positive — the string `"<form></form><form></form>"` is a static literal inside jQuery 3.5.1's `createHTMLDocument` browser feature-detection code, not a real HTML form element |
| External `<script src>` without SRI | No external script tags present — all scripts are inline or base64-encoded data URIs                                                                                                             |

---

_Report generated by static analysis of the HTML file. Dynamic/runtime behaviour (e.g. actual axe-core violations at runtime, Content-Security-Policy header values set by the hosting server) was not assessed._
