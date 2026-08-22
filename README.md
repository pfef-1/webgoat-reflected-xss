# WebGoat Reflected XSS — Shopping Cart

Reflected XSS assessment against OWASP WebGoat's shopping cart module, performed as part
of a graduate intrusion testing course (CCCS-455-784, McGill University School of
Continuing Studies). Traffic was intercepted and manipulated with Burp Suite to find and
exploit an unencoded reflection point, then a JavaScript payload was built to
manipulate the DOM directly — resetting quantities and zeroing out prices client-side,
without ever touching the server.

This project demonstrates a full attack lifecycle: systematic vulnerable-field
discovery, payload iteration, exploitation, and a defender's-eye view of what would
actually stop it in production.

> Performed in an isolated lab environment (Kali Linux attacker VM + OWASP Broken Web
> Apps victim VM) against a deliberately vulnerable training application. Not performed
> against any production system.

## Objective

WebGoat's "Reflected XSS Attacks" lesson presents a shopping cart form with several
input fields. The goal was to determine whether any of those fields reflect user input
back into the page without encoding it — and if so, use that to prove real impact by
manipulating the cart (rather than just popping an `alert()` box).

## Methodology

**1. Map the attack surface.** Traffic was captured in Burp Suite while interacting
with the cart normally (`UpdateCart`, `Purchase`) to see the full request body without
guessing:

```
QTY1=1&QTY2=1&QTY3=1&QTY4=1&field2=4128+3214+0002+1999&field1=111&SUBMIT=Purchase
```

Seven parameters total: four quantity fields (`QTY1`–`QTY4`), a credit card number
(`field2`), and a three-digit access code (`field1`).

**2. Test each field for reflection.** Rather than guessing which field was
exploitable, every field was tested individually with a harmless marker payload:

```
'><script>alert('XSS')</script>
```

The leading `'>` is the actual test — it attempts to break out of the current HTML
attribute/tag context. If the server encodes special characters before reflecting
them, the browser displays the payload as inert text. If it doesn't, the browser
parses `<script>` as real markup and executes it.

- `QTY1`–`QTY4`: payload reflected as plain text, no execution.
- `field2` (credit card): input got mangled by the field's own formatting/masking
  before reflection — never rendered as executable markup.
- `field1` (access code): payload executed. WebGoat's lesson-complete banner fired
  immediately, confirming the `<script>` tag was parsed as real HTML, not displayed as
  text.

## Key Finding

The **access code field (`field1`)** reflects user input directly into the HTTP
response with no output encoding. Any HTML or JavaScript submitted in that field is
parsed and executed by the browser exactly as if it were part of the page.

Confirmed unambiguously with a second, more diagnostic payload:

```html
"><svg/onload=alert(document.cookie)>
```

This payload is deliberately more useful than a plain `alert('XSS')` for two reasons:
`"> `attempts to break out of the current HTML context (proving the injection point),
and `<svg onload=...>` is a tag/event combination that has no legitimate reason to
appear in user input — so if it fires, it's unambiguous proof of unencoded reflection,
not a false positive. It fired successfully, popping the live session cookie
(`JSESSIONID=...`) in an alert box — proof that arbitrary JavaScript in the page's
execution context has full access to `document.cookie`.

## Exploitation

With confirmed script execution, the next step was to go beyond a proof-of-concept
`alert()` and demonstrate real application impact: silently modify the shopping cart
to set every item's quantity to 999 and every price to $0.00, entirely client-side.

The payload works in two passes over the DOM, using
[`document.getElementsByTagName`](https://developer.mozilla.org/en-US/docs/Web/API/Document/getElementsByTagName)
to collect elements, then a `for` loop to inspect and modify each one:

```html
"><script>window.onload=function(){
  // Pass 1: every <input> whose name starts with "QTY" gets its value forced to 999
  var inputs = document.getElementsByTagName('input');
  for (var a = 0; a < inputs.length; a++) {
    if (inputs[a].name && inputs[a].name.indexOf('QTY') === 0) {
      inputs[a].value = 999;
    }
  }

  // Pass 2: every <td> whose displayed text is a price ("$69.99") or a plain
  // decimal ("69.99") gets overwritten to display $0.00
  var cells = document.getElementsByTagName('td');
  for (var b = 0; b < cells.length; b++) {
    var text = cells[b].innerHTML;
    if (text.indexOf('$') === 0) {
      cells[b].innerHTML = '$0.00';        // Total column
    } else if (/^\d+\.\d\d$/.test(text)) {
      cells[b].innerHTML = '0.00';         // Price column
    }
  }
}</script>
```

Why two passes instead of one: `<input>` elements expose a `.name` attribute that
directly identifies which field is which (`QTY1`, `QTY2`, ...), so filtering on that is
straightforward. `<td>` elements carry no such identifying attribute — the only way to
tell a "Price" cell from a "Total" cell from any other cell in the table is by
inspecting what's actually displayed inside it, hence the `$` prefix check and the
`^\d+\.\d\d$` regular expression as two separate matching rules.

`window.onload = function(){...}` defers execution until the full page — including the
shopping cart table — has finished loading, so the script isn't racing against
elements that don't exist yet at the moment the payload itself is parsed.

**Result:** every quantity box read `999`, every price and total cell read `$0.00`,
and the total charge dropped to `$0.00` — all without a single request to the server
that WebGoat's own logic didn't already expect.

## Technical Explanation: Why This Is Reflected XSS

Reflected XSS happens when an application takes untrusted input, and immediately
embeds it back into the HTTP response **without encoding it**, so the browser can no
longer tell the difference between "page content" and "code the page is supposed to
run."

Concretely, in this case:

1. The `field1` value gets echoed back into the page's HTML inside the response to the
   `Purchase`/`UpdateCart` request.
2. The server does this with **zero output encoding** — characters like `<`, `>`, and
   `"` are inserted into the HTML verbatim instead of being converted to their safe
   equivalents (`&lt;`, `&gt;`, `&quot;`).
3. The browser has no way to know that `<script>...</script>` in the response was
   supposed to be *data* (the value the user typed) rather than *markup* (part of the
   page). It parses it exactly like any other tag on the page and executes it.
4. Because this executes in the same origin as the real WebGoat page, the injected
   script has full access to that page's DOM, cookies, and any other JavaScript-
   accessible state — which is exactly what let it read `document.cookie` and rewrite
   the cart table.

It's called *reflected* (as opposed to *stored*) because the payload only exists for
the duration of that one request/response — nothing is saved server-side. The
vulnerability lives entirely in one place: the response handler that echoes `field1`
back without encoding it.

## Impact / Risk in a Real Environment

This lab used play-money quantities and prices, but the same mechanism scales
directly to serious impact on a production e-commerce or banking application:

- **Session hijacking.** The SVG payload proved `document.cookie` is readable from
  injected script. On a real site without `HttpOnly` cookies, that's a session token
  an attacker can exfiltrate and use to impersonate the victim.
- **Credential/payment theft.** The same injection technique used here to rewrite
  `<td>` cells could just as easily inject a fake login form or fake payment field
  overlaid on the real page — the user has no way to tell it apart from legitimate
  UI, because it's running inside the legitimate page's origin.
- **Price/business-logic manipulation.** Purely client-side manipulation (as
  demonstrated here) is only dangerous if the server trusts client-submitted totals
  without re-validating them — which is a separate, common flaw worth checking for
  on any target that reflects this kind of vulnerability.
- **Requires user interaction.** Reflected XSS isn't self-propagating — it needs the
  victim to submit the malicious input themselves (e.g. via a crafted link, since a
  reflected payload is typically delivered as part of a URL or form an attacker gets
  the victim to trigger). That's the main thing that limits its blast radius compared
  to stored XSS, which doesn't require re-luring each victim individually.

## Remediation

- **Output encoding, not just input validation.** The core fix: encode user-supplied
  data based on the context it's being rendered into (HTML entity encoding for HTML
  body content, JS-string escaping inside `<script>` blocks, attribute encoding
  inside tag attributes). Input validation alone doesn't prevent XSS — encoding at
  the point of output does.
- **Content Security Policy (CSP).** A strict CSP (e.g. disallowing inline
  `<script>` execution) would have blocked every payload used in this assessment,
  even if the output-encoding bug were still present. Defense in depth.
- **`HttpOnly` cookies.** Marking session cookies `HttpOnly` prevents JavaScript —
  including injected JavaScript — from reading `document.cookie` at all, closing off
  the session-hijacking path even if reflection still exists somewhere.
- **Input validation as a secondary layer.** Fields with a known format (a
  three-digit access code, a 16-digit card number) should be validated server-side
  against that format and rejected outright if they don't match — this wouldn't fix
  the underlying encoding flaw, but it does shrink the attack surface for that
  specific field.

## Skills & Tools

- **Burp Suite** — traffic interception, request body analysis, Repeater for
  iterative payload testing
- **HTTP request/response analysis** — systematic parameter enumeration instead of
  guessing which field to attack
- **JavaScript / DOM manipulation** — `document.getElementsByTagName`, DOM traversal,
  `.innerHTML` manipulation, regex-based content matching
- **Browser DevTools** — Elements/Console panels used to confirm payload behavior
  against the live DOM
- **Vulnerability analysis & reporting** — root-cause explanation (not just
  proof-of-concept), attacker/defender risk framing

---

*Performed for CCCS-455-784 (Intrusion Testing & Security Assessment), McGill
University School of Continuing Studies, Summer 2026, as a group assignment. This
write-up covers the technical execution I led.*
