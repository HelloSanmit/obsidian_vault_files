
# Lesson 48 — React Security Hardening

Security in a React application is not one feature.

It is a chain of boundaries:

```text
browser
   ↓
React rendering
   ↓
DOM
   ↓
cookies / credentials
   ↓
HTTP requests
   ↓
server functions / APIs
   ↓
authorization
   ↓
database
   ↓
third-party dependencies
```

A vulnerability appears when data crosses one of those boundaries without the right assumptions.

The central rule for this lesson is:

> **Never confuse "this came from my React UI" with "this is trusted."**

Anything coming from the browser can be manipulated.

---

# 1. First: what React already protects you from

React normally escapes text content.

Suppose a user enters:

```text
<img src=x onerror="alert('hacked')">
```

and you render:

```jsx
function Comment({ text }) {
  return <p>{text}</p>;
}
```

React treats that as text.

Conceptually:

```text
untrusted string
      ↓
React JSX text
      ↓
escaped DOM text
```

So the browser doesn't interpret the user's `<img>` as HTML.

That's one of the major security benefits of using a templating system that escapes ordinary values.

OWASP notes the same general principle: modern frameworks reduce many XSS vulnerabilities through escaping, but unsafe framework features and dangerous DOM APIs can reintroduce them. ([OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com "Cross Site Scripting Prevention - OWASP Cheat Sheet Series"))

---

# 2. The moment you bypass escaping, security changes

Now consider:

```jsx
<div
  dangerouslySetInnerHTML={{
    __html: comment
  }}
/>
```

You've effectively said:

> "Browser, please interpret this string as HTML."

The data flow becomes:

```text
attacker-controlled input
        ↓
HTML string
        ↓
innerHTML
        ↓
browser parses HTML
```

That is an injection boundary.

React explicitly warns that `dangerouslySetInnerHTML` is dangerous when its content isn't trusted and sanitized. ([React](https://react.dev/reference/react-dom/components/common?utm_source=chatgpt.com "Common components (e.g. <div>) – React"))

So don't think:

```text
dangerouslySetInnerHTML = bad
```

Think:

```text
raw HTML insertion = security-sensitive boundary
```

That distinction matters.

---

# 3. Why is it called "dangerously"?

Because HTML isn't just visual data.

HTML can contain things that create behavior:

```html
<script>...</script>
<img onerror="...">
<iframe ...>
<a href="...">
<form ...>
```

and browser APIs can create similar injection risks.

So:

```jsx
<div>{userText}</div>
```

and:

```jsx
<div dangerouslySetInnerHTML={{__html: userText}} />
```

have radically different trust models.

The first is:

```text
data → text
```

The second is:

```text
data → markup → DOM behavior
```

---

# 4. XSS is broader than `<script>`

People often learn XSS as:

```html
<script>alert(1)</script>
```

That's too narrow.

XSS means an attacker gets unintended executable behavior into the user's browser.

The dangerous path might involve:

```text
HTML
DOM
JavaScript URLs
event handlers
SVG
third-party widgets
unsafe DOM APIs
```

OWASP describes XSS as capable of consequences including account impersonation and sensitive-data exposure, depending on what the application exposes. ([OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com "Cross Site Scripting Prevention - OWASP Cheat Sheet Series"))

So when auditing React code, don't search only for `<script>`.

Look for **injection sinks**.

---

# 5. Injection sinks

An injection sink is an API where data stops being ordinary data and starts being interpreted as something more powerful.

Examples include:

```js
element.innerHTML = value;
```

```js
document.write(value);
```

```js
eval(value);
```

and other DOM/code-loading APIs.

MDN describes APIs such as `innerHTML` as DOM XSS injection sinks because they interpret their inputs rather than merely storing text. ([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP?utm_source=chatgpt.com "Content Security Policy (CSP) - HTTP | MDN"))

This gives you a powerful audit technique:

> **Find places where data crosses from "value" into "code/markup/resource."**

---

# 6. React does not sanitize your business data

Suppose the server stores:

```json
{
  "bio": "<b>Alice</b>"
}
```

React doesn't magically know whether that HTML is safe.

The application must decide:

```text
Is this plain text?
Is this trusted HTML?
Is this markdown?
Is this user-authored rich text?
```

Those are different data types.

This is why "string" is sometimes not a sufficiently precise model for security-sensitive application data.

---

# 7. Markdown is not automatically safe

Suppose you let users write:

```markdown
# Hello
```

and use a Markdown parser:

```text
Markdown
   ↓
HTML
```

Now you have created an HTML-producing transformation.

The security question becomes:

```text
Does parser output contain dangerous markup?
```

A safe architecture is:

```text
untrusted markdown
       ↓
strict parser
       ↓
sanitizer
       ↓
trusted HTML
       ↓
controlled rendering
```

Not:

```text
untrusted markdown
       ↓
convert to HTML
       ↓
dangerouslySetInnerHTML
```

React itself explicitly points out that even when Trusted Types are used, the policy producing `TrustedHTML` still has to make the content safe. ([React](https://react.dev/reference/react-dom/components/common?utm_source=chatgpt.com "Common components (e.g. <div>) – React"))

---

# 8. Sanitization vs escaping

These are related but different.

## Escaping

You don't want HTML.

Input:

```text
<b>Hello</b>
```

Output:

```text
&lt;b&gt;Hello&lt;/b&gt;
```

The user sees:

```text
<b>Hello</b>
```

as text.

## Sanitization

You intentionally allow some HTML.

Input:

```html
<p>Hello</p>
<img src="..." onerror="...">
```

A sanitizer might keep:

```html
<p>Hello</p>
```

and remove dangerous content.

So:

```text
escaping
= don't interpret markup

sanitization
= interpret markup, but remove unsafe capabilities
```

That distinction becomes essential for rich-text editors.

---

# 9. Trusted Types

This is a more advanced browser defense and especially relevant now.

Current React 19.3 integrates with the browser Trusted Types API, allowing Trusted Type values to pass through to DOM injection sinks without React first coercing them back into strings. ([React](https://react.dev/blog/2026/09/09/react-19-3?utm_source=chatgpt.com "React 19.3 – React"))

The mental model is:

```text
ordinary string
      ↓
dangerous sink
```

versus:

```text
ordinary string
      ↓
approved transformation / sanitizer
      ↓
TrustedHTML
      ↓
dangerous sink
```

Trusted Types makes the second pathway enforceable by the browser.

---

# 10. Why Trusted Types are interesting

Imagine your application has 100 places where DOM HTML could potentially be inserted.

Without enforcement:

```text
100 possible places
```

With Trusted Types enforcement:

```text
only explicitly approved TrustedHTML paths
```

The browser can reject raw strings passed to protected DOM sinks.

MDN describes `require-trusted-types-for` as a CSP mechanism that controls values passed to DOM XSS sinks such as `innerHTML`. ([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/require-trusted-types-for?utm_source=chatgpt.com "Content-Security-Policy: require-trusted-types-for directive - HTTP | MDN"))

And the `trusted-types` directive lets you constrain which policy names are allowed. ([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/trusted-types?utm_source=chatgpt.com "Content-Security-Policy: trusted-types directive - HTTP | MDN"))

---

# 11. Trusted Types don't magically sanitize anything

This is extremely important.

You can create a Trusted Types policy that returns unsafe content.

Conceptually:

```js
trustedTypes.createPolicy('bad-policy', {
  createHTML: value => value
});
```

Now you've created a trusted object without actually making it safe.

So the actual model is:

```text
Trusted Types
     ≠
sanitization
```

Instead:

```text
Trusted Types
     +
good sanitization policy
     =
stronger XSS defense
```

MDN explicitly warns that a Trusted Types policy can be written in a way that fails to sanitize input, which is why policy governance matters. ([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP?utm_source=chatgpt.com "Content Security Policy (CSP) - HTTP | MDN"))

---

# 12. CSP: the browser's second line of defense

Content Security Policy is a browser-enforced policy delivered mainly through HTTP response headers.

Conceptually:

```text
server
  ↓
Content-Security-Policy
  ↓
browser
  ↓
enforces resource/script restrictions
```

A CSP can help limit:

```text
where scripts can come from
whether inline scripts execute
which resources can load
whether the page can be framed
whether unsafe DOM sinks accept strings
```

MDN recommends serving CSP through the `Content-Security-Policy` response header and describes it as a defense layer for XSS, clickjacking, insecure requests, and related threats. ([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP?utm_source=chatgpt.com "Content Security Policy (CSP) - HTTP | MDN"))

---

# 13. CSP is defense in depth

Suppose there's an XSS bug.

Without CSP:

```text
XSS vulnerability
      ↓
attacker script
      ↓
potentially executes
```

With a strong CSP:

```text
XSS vulnerability
      ↓
attacker script
      ↓
browser policy may block it
```

But:

```text
CSP ≠ substitute for fixing XSS
```

It's another layer.

OWASP similarly describes CSP as defense in depth rather than the root fix for unsafe application code. ([OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html?utm_source=chatgpt.com "Content Security Policy - OWASP Cheat Sheet Series"))

---

# 14. A conceptual CSP

A simplified example might look like:

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  object-src 'none';
  base-uri 'self';
  frame-ancestors 'none';
  require-trusted-types-for 'script';
```

Don't blindly paste this into every application.

CSP depends on:

```text
analytics
CDNs
fonts
payments
third-party widgets
SSR
nonce/hash strategy
```

The important lesson is what each rule represents.

---

# 15. `frame-ancestors`

Imagine an attacker embeds your site:

```html
<iframe src="https://yourbank.example"></iframe>
```

and places invisible UI over it.

The victim thinks they're clicking:

```text
"Play video"
```

but actually clicks:

```text
"Transfer money"
```

This is clickjacking.

A CSP directive such as:

```http
frame-ancestors 'none'
```

can tell the browser:

> "Do not allow this page to be embedded."

MDN recommends `frame-ancestors` as a control over framing. ([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP?utm_source=chatgpt.com "Content Security Policy (CSP) - HTTP | MDN"))

---

# 16. `object-src 'none'`

Older browser technologies such as plugins are generally unnecessary for modern React applications.

So a CSP commonly restricts:

```http
object-src 'none'
```

This is another example of shrinking the browser's attack surface.

Security isn't always about adding complexity.

Sometimes it's about saying:

> "This application doesn't need that capability. Turn it off."

---

# 17. HTTPS is fundamental

Your React code can be perfect.

Your API authorization can be perfect.

Your cookies can be carefully configured.

But if sensitive communication travels over insecure transport:

```text
browser
   ↓
HTTP
   ↓
network
```

you have a serious problem.

The intended model is:

```text
browser
   ↓
HTTPS
   ↓
server
```

MDN lists HTTPS for pages and subresources among core web security practices. ([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/Security?utm_source=chatgpt.com "Security | MDN"))

---

# 18. Cookies: why they matter to React

Suppose authentication uses a session cookie:

```http
Set-Cookie:
  session=abc123;
  Secure;
  HttpOnly;
  SameSite=Lax
```

Each attribute answers a different security question.

### `Secure`

Only send over HTTPS.

### `HttpOnly`

JavaScript cannot directly read the cookie.

### `SameSite`

Controls when browsers send the cookie in cross-site contexts.

So:

```text
React
  ↓
fetch('/api/profile')
  ↓
browser automatically attaches session cookie
```

The React application doesn't need:

```js
document.cookie
```

to read the session.

---

# 19. Why HttpOnly matters

Imagine an XSS bug.

If your session token lives in ordinary JavaScript-accessible storage:

```text
XSS
 ↓
read token
 ↓
send token away
```

With an HttpOnly cookie:

```text
XSS
 ↓
JavaScript cannot directly read session cookie
```

That does **not** make XSS harmless.

An injected script may still make requests as the user from the compromised page.

But HttpOnly can prevent one particularly dangerous form of token theft.

This is why security controls should be layered.

---

# 20. SameSite does not mean "CSRF solved forever"

CSRF works because:

```text
attacker site
    ↓
victim browser
    ↓
request to target site
    ↓
browser automatically includes authentication cookie
```

The server may see:

```text
Cookie: session=...
```

and think:

> "This is an authenticated user."

But the request may have been triggered by the attacker.

OWASP recommends using appropriate CSRF protection for state-changing requests in cookie-authenticated applications; exact strategy depends on the authentication model and framework. ([OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com "Cross-Site Request Forgery Prevention - OWASP Cheat Sheet Series"))

---

# 21. CSRF example

Suppose your server supports:

```http
POST /api/email/change
```

and authentication is a cookie.

An attacker may try to cause:

```text
victim browser
      ↓
POST /api/email/change
      ↓
session cookie automatically attached
```

If the application has no request-origin/challenge protection, the server may process the request.

The victim's browser sent it.

The attacker merely caused it.

That's the essence of CSRF.

---

# 22. CSRF tokens

A common defense is a token that the attacker site cannot correctly provide.

Conceptually:

```text
server
 ↓
generate CSRF token
 ↓
browser

browser
 ↓
state-changing request
 ↓
CSRF token
 ↓
server validates token
```

OWASP describes token-based and cookie-to-header patterns for applications using cookie authentication. ([OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com "Cross-Site Request Forgery Prevention - OWASP Cheat Sheet Series"))

The exact mechanism depends on the backend/framework.

The critical point:

> **The server must validate the defense.**

A token stored in React state is not a security control by itself.

---

# 23. CORS is not CSRF protection

This confusion causes a huge number of security mistakes.

People often think:

```text
CORS blocks attacker.com
therefore CSRF impossible
```

Not necessarily.

CORS primarily controls whether browser JavaScript can read cross-origin responses.

CSRF is about causing the request.

These are different:

```text
CORS:
"Can attacker JavaScript read the response?"

CSRF:
"Can attacker cause an authenticated state-changing request?"
```

A browser can sometimes send the request even when the attacker cannot read the response.

So:

```text
CORS ≠ CSRF
```

---

# 24. Authentication is not authorization

We already introduced this earlier, but security hardening needs it deeply.

Suppose:

```text
Alice logs in
```

Authentication establishes:

```text
user = Alice
```

Authorization asks:

```text
Can Alice modify project 42?
```

These are different questions.

---

# 25. React route guards are UX, not security

You might write:

```jsx
if (!user) {
  return <Navigate to="/login" />;
}
```

Good.

But imagine an attacker directly sends:

```http
DELETE /api/projects/42
```

The React route isn't involved.

Therefore:

```text
React guard
    ↓
protects UI navigation
```

while:

```text
server authorization
    ↓
protects actual resource
```

The server must make the security decision.

---

# 26. Client-side permissions are advisory

You might do:

```jsx
{permissions.includes('project.delete') && (
  <button>Delete</button>
)}
```

That's useful.

But the attacker can modify the browser.

They could:

```text
open DevTools
 ↓
change React state
 ↓
call API manually
```

Therefore:

```text
UI permission check
    ≠
authorization enforcement
```

The backend must independently verify:

```text
authenticated identity
+
requested resource
+
required permission
```

---

# 27. IDOR / resource ownership

A particularly important authorization bug is:

```text
GET /api/projects/123
```

The server assumes:

```text
"If you're logged in, you can read it."
```

But maybe user Alice should only see project 123 if:

```text
project.ownerId === Alice.id
```

If the server doesn't check that, Alice may change:

```text
123 → 124 → 125
```

and access someone else's data.

The URL itself is not proof of authorization.

This is one of the reasons server-side resource ownership checks matter.

---

# 28. Server Functions are security boundaries

Recall:

```text
client
 ↓
Server Function
 ↓
database
```

From a security perspective, a Server Function is an API endpoint behind a convenient programming model.

React's current Server Function model allows client code to invoke functions that execute on the server. ([React](https://react.dev/reference/rsc/server-functions?utm_source=chatgpt.com "Server Functions – React"))

Therefore the function must assume:

```text
arguments are attacker-controlled
```

Even if your normal UI always passes clean values.

---

# 29. Never trust hidden UI

Suppose the client sends:

```js
updateUser({
  role: 'admin'
});
```

The server must not think:

> "The UI wouldn't normally expose this."

An attacker can call the underlying endpoint directly.

The server should perform:

```text
authenticate
   ↓
validate input
   ↓
authorize operation
   ↓
perform mutation
```

not:

```text
receive request
   ↓
trust client
   ↓
database update
```

---

# 30. Validation is not authorization

This distinction is subtle.

Suppose:

```js
role
```

is validated as:

```text
'admin' | 'editor' | 'viewer'
```

Good.

But that doesn't answer:

> "Is this user allowed to assign `admin`?"

So:

```text
validation
= Is this value structurally/legal?

authorization
= Is this actor allowed to perform this operation?
```

You usually need both.

---

# 31. Never trust hidden form fields

Suppose your form sends:

```html
<input type="hidden" name="userId" value="42">
```

The browser can modify it to:

```text
userId=43
```

Therefore:

```text
hidden
≠
trusted
```

This applies equally to:

```text
disabled inputs
hidden inputs
React state
route parameters
query parameters
cookies readable by JS
localStorage
```

Anything originating on the client can be manipulated.

---

# 32. Validate on the server

Suppose the client sends:

```json
{
  "price": -999999
}
```

or:

```json
{
  "projectId": "../other-user-project"
}
```

or:

```json
{
  "role": "admin"
}
```

The server must validate.

Client validation is mainly for:

```text
UX
```

Server validation is required for:

```text
trust boundary enforcement
```

---

# 33. Don't put secrets in Vite client environment variables

From our deployment lesson:

```text
VITE_API_URL
```

is client-visible.

Therefore:

```env
VITE_SECRET_KEY=...
```

is not secret.

The build process embeds exposed environment variables into client code.

The browser ultimately receives the resulting JavaScript.

So the rule is:

```text
Anything shipped to the browser is public.
```

No matter what environment variable name you gave it.

---

# 34. Secret management mental model

A secret should live where the browser cannot receive it.

For example:

```text
Browser
  ↓
Server Function / API
  ↓
secret
  ↓
database / payment provider
```

Not:

```text
Browser
  ↓
secret
  ↓
payment provider
```

The browser is a hostile execution environment from the application's trust perspective.

---

# 35. LocalStorage and session tokens

A common pattern is:

```js
localStorage.setItem(
  'accessToken',
  token
);
```

This makes the token accessible to JavaScript.

That can be attractive because it is easy.

But it also means:

```text
XSS
 ↓
JavaScript
 ↓
localStorage
 ↓
token
```

Whether to use token-in-storage, memory, or cookies depends on the architecture and threat model.

For browser session authentication, HttpOnly cookies are often useful because JavaScript cannot read the session cookie directly.

The important lesson is not:

```text
"localStorage is always illegal"
```

but:

> **Know what becomes readable by injected JavaScript and choose the credential architecture deliberately.**

---

# 36. Open redirects

Consider:

```jsx
navigate(next);
```

where:

```text
next = URL parameter controlled by user
```

If the application blindly allows:

```text
https://evil.example
```

you may create an open redirect.

That can become useful for:

```text
phishing
social engineering
trust abuse
```

Safer patterns often restrict redirect targets to:

```text
known internal paths
```

for example:

```text
/dashboard
/projects/42
```

rather than accepting arbitrary external URLs.

---

# 37. URL parameters are untrusted input

We've already studied:

```text
route params
query params
```

from a React Router perspective.

Security adds another dimension.

This:

```js
const { projectId } = useParams();
```

doesn't mean:

```text
projectId is valid
```

It only means:

```text
projectId came from the URL
```

Therefore the server should validate and authorize it.

---

# 38. Safe URL handling

Be careful when user-controlled values become URLs.

For example:

```jsx
<a href={user.website}>
  Website
</a>
```

You need to think about the protocol.

A URL value isn't necessarily harmless merely because it is inside an `href`.

Application policies often allow only expected schemes such as:

```text
https:
http:
```

and reject unexpected executable/navigation schemes.

This is another instance of:

```text
value
 ↓
powerful browser interpretation
```

---

# 39. Browser APIs can become security boundaries

Some APIs deserve special caution:

```js
window.open(...)
location.href = ...
element.innerHTML = ...
eval(...)
new Function(...)
```

Security review should ask:

```text
Can untrusted data reach this?
```

The important phrase is:

> **data flow**

rather than:

> "Does this line look dangerous by itself?"

---

# 40. SSR introduces another trust boundary

Suppose server rendering produces HTML:

```text
database
 ↓
React Server Component
 ↓
HTML
 ↓
browser
```

If untrusted data is rendered incorrectly into:

```text
HTML
attributes
script-like contexts
serialized state
```

you can create injection vulnerabilities.

SSR therefore doesn't eliminate browser security issues.

It changes where part of the rendering happens.

---

# 41. Hydration data deserves attention

Imagine a server renders:

```html
<script>
  window.__INITIAL_DATA__ = {...}
</script>
```

A common architecture needs to transfer server state to the browser.

The dangerous question is:

> "How is that state serialized?"

You must use a serialization mechanism that safely represents arbitrary data rather than blindly concatenating strings into JavaScript source.

This is a general server-rendering lesson:

```text
serialized data
≠
JavaScript source code
```

Treat them differently.

---

# 42. Third-party scripts

Suppose you add:

```text
analytics.js
chat-widget.js
payment.js
```

Every third-party script is effectively code running with powerful access to the page.

So security isn't just:

```text
"is my React code secure?"
```

It is also:

```text
"Which code am I allowing onto the page?"
```

This leads directly to:

```text
CSP
subresource integrity where appropriate
dependency governance
third-party review
least privilege
```

---

# 43. Supply-chain security

Your application may directly depend on:

```json
{
  "dependencies": {
    "react": "...",
    "some-library": "...",
    "markdown-parser": "..."
  }
}
```

But those libraries have their own dependencies.

So the true dependency graph looks like:

```text
your app
  ↓
package A
  ↓
package B
  ↓
package C
  ↓
package D
```

Your attack surface therefore includes code you didn't personally write.

---

# 44. A dependency can become compromised without your code changing

Imagine:

```text
Monday
package X = version 4.1.0
```

and later:

```text
Tuesday
package X = malicious 4.1.1
```

A loose version range may install the new release.

Your source code didn't change.

Yet your shipped JavaScript did.

That's why:

```text
lockfile
+
dependency review
+
controlled updates
```

matter.

---

# 45. Lockfiles

A lockfile records the resolved dependency graph.

Conceptually:

```text
package.json
    ↓
declared range

lockfile
    ↓
exact resolved dependency graph
```

That gives builds much stronger reproducibility.

Without it:

```text
same source code
+
different day
=
potentially different dependencies
```

With it:

```text
same source code
+
same lockfile
=
much more reproducible dependency resolution
```

---

# 46. Dependency updates should be intentional

Don't blindly run:

```bash
npm update
```

and assume everything is harmless.

A production dependency process should consider:

```text
security advisories
breaking changes
transitive changes
maintainer health
package scope
release behavior
```

You don't need paranoia around every package.

You need visibility into what code you're shipping.

---

# 47. Typosquatting

Suppose you intended:

```text
some-useful-library
```

but accidentally install:

```text
some-usefull-library
```

The malicious package may have a similar name.

This is called typosquatting.

It demonstrates a broader principle:

> **Package installation is code execution.**

Treat dependency installation with the same seriousness as adding source code.

---

# 48. Lifecycle scripts

Package managers may execute package lifecycle scripts during installation/builds.

That means:

```text
install dependency
```

can potentially execute code.

This is one reason to carefully consider:

```text
what package you're installing
what scripts it contains
whether your CI has sensitive credentials
what permissions the build environment has
```

A secure CI environment should not unnecessarily expose:

```text
production secrets
cloud credentials
database credentials
signing keys
```

to arbitrary build steps.

---

# 49. Principle of least privilege

This is one of the best security principles to remember.

Give a thing only the capabilities it needs.

For example:

```text
frontend build
```

probably doesn't need:

```text
production database write access
```

A Server Function that only reads public product data doesn't need:

```text
admin-level database permissions
```

A monitoring script doesn't need:

```text
user password access
```

Security often improves when you reduce capability rather than adding more checks.

---

# 50. Environment separation

Think:

```text
development
staging
production
```

as separate trust environments.

You don't want:

```text
local development
 ↓
production database
```

to be the normal workflow.

Instead:

```text
development → development resources
staging     → staging resources
production  → production resources
```

This reduces blast radius.

---

# 51. Database permissions

Your API server may connect to the database.

Don't automatically give it:

```text
everything
```

Use constrained credentials where practical.

For example:

```text
read-only reporting service
        ↓
read-only database permission
```

versus:

```text
reporting service
        ↓
DROP TABLE permission
```

Least privilege applies to infrastructure too.

---

# 52. Error messages can leak information

Suppose production returns:

```json
{
  "error":
  "PostgreSQL relation users_private failed because..."
}
```

That's useful to an attacker and not useful to the end user.

A production application generally wants:

```json
{
  "error": "Something went wrong"
}
```

while internal logging contains:

```text
requestId=abc123
db failure...
stack...
service...
```

This creates two channels:

```text
user-facing error
+
internal diagnostic detail
```

---

# 53. Don't log secrets

A common observability mistake is logging:

```js
console.log(request);
```

when the request contains:

```text
Authorization
cookies
password
credit card data
session identifiers
```

Now your logs become a sensitive data store.

Log structured context instead:

```json
{
  "requestId": "abc123",
  "route": "/api/projects",
  "userId": "42"
}
```

but avoid sensitive values.

Security and observability must be designed together.

---

# 54. Rate limiting

Suppose your Server Function allows:

```text
POST /api/login
```

An attacker can automate requests.

You may therefore need controls such as:

```text
rate limiting
account lockout strategy
progressive delays
bot detection
abuse monitoring
```

The exact strategy depends heavily on the endpoint.

Login:

```text
high abuse potential
```

while:

```text
GET /api/public/products
```

might have a different rate-limit policy.

---

# 55. Resource limits are security too

Consider:

```text
upload file
```

An attacker might upload:

```text
10 GB
```

or send:

```text
50 million JSON array elements
```

or construct:

```text
extremely deep nested input
```

So validation isn't merely:

```text
"Is it the correct type?"
```

It can also be:

```text
"Is it within acceptable resource limits?"
```

This is part of defensive engineering.

---

# 56. File uploads

File uploads deserve special caution.

Never assume:

```text
filename extension = file type
```

or:

```text
MIME type sent by browser = trustworthy
```

A robust backend may need to:

```text
limit size
validate content
rename stored objects
store outside executable web paths
control download headers
scan where appropriate
```

The React UI is only the front door.

The server owns the trust boundary.

---

# 57. Security headers as a production layer

A mature web application often considers a family of browser controls:

```text
HTTPS
CSP
frame-ancestors
Strict-Transport-Security
Referrer-Policy
Permissions-Policy
cookie attributes
cross-origin policies
```

Not every application needs identical settings.

The key idea is:

```text
browser
  ↓
server-provided security policy
  ↓
browser enforces limits
```

This complements application-level checks.

---

# 58. Security is defense in depth

Imagine a malicious input gets through one layer.

Good architecture tries to stop it elsewhere.

For example:

```text
Input validation
      ↓
safe rendering
      ↓
CSP
      ↓
Trusted Types
      ↓
HttpOnly cookie
      ↓
server authorization
      ↓
database permissions
```

No individual control is perfect.

Together, they reduce blast radius.

---

# 59. A realistic attack chain

Let's walk through a hypothetical application.

Suppose:

```text
comment box
```

accepts user input.

Attacker stores:

```text
malicious content
```

Then:

```text
database
 ↓
server
 ↓
React
 ↓
dangerouslySetInnerHTML
 ↓
DOM
 ↓
attacker JavaScript
```

Now that script tries to:

```text
read session token
```

If the session is an HttpOnly cookie:

```text
direct token theft → blocked
```

The script then tries to make a destructive request:

```text
POST /api/change-email
```

Server authorization and CSRF protections still matter.

Then CSP may block some script execution paths.

This demonstrates why security is a system rather than a single checkbox.

---

# 60. Security testing should follow the same architecture

Earlier we built a testing hierarchy.

Security testing can mirror it:

```text
pure validation logic
      ↓
unit tests

authentication / authorization logic
      ↓
server integration tests

API behavior
      ↓
integration/security tests

browser attack surfaces
      ↓
E2E / security automation

production headers / deployment
      ↓
environment-level checks
```

---

# 61. What should React tests verify?

For example:

```text
unauthorized user
 ↓
delete button not shown
```

That's valuable.

But don't stop there.

The server test should verify:

```text
unauthorized request
 ↓
DELETE /projects/42
 ↓
403
 ↓
database unchanged
```

Now you have:

```text
UI security
+
actual security
```

---

# 62. Security regression test example

Suppose an attacker calls:

```http
DELETE /api/projects/42
```

with Alice's valid session.

But Alice isn't a member of project 42.

Expected:

```http
403 Forbidden
```

and:

```text
database unchanged
```

That's a powerful regression test.

Because now the rule is executable:

> An authenticated user who lacks permission cannot mutate another user's resource.

---

# 63. The security mindset for Server Components and Server Functions

Remember our architecture:

```text
Server Component
      ↓
Server Function
      ↓
Database
```

The server boundary is actually a great opportunity.

You can keep:

```text
DB credentials
API keys
internal services
authorization logic
```

on the server.

Don't turn that advantage into a mistake by sending secrets or privileged data across the client boundary.

Always ask:

```text
Does this value actually need to cross into the browser?
```

---

# 64. Data minimization

Suppose a database user record contains:

```json
{
  "id": 42,
  "name": "Alice",
  "email": "...",
  "passwordHash": "...",
  "internalNotes": "...",
  "billingFlags": "...",
  "adminMetadata": "..."
}
```

Your React component needs:

```json
{
  "id": 42,
  "name": "Alice"
}
```

Send only what is needed.

Don't return:

```text
entire database row
```

just because it is convenient.

This reduces accidental exposure.

---

# 65. A powerful architectural pattern: DTOs

Instead of:

```text
database object
 ↓
directly serialize
 ↓
browser
```

use a deliberate API shape:

```text
database model
       ↓
server transformation
       ↓
DTO / response model
       ↓
browser
```

For example:

```js
function toPublicUser(user) {
  return {
    id: user.id,
    name: user.name
  };
}
```

Now sensitive database fields don't cross the trust boundary accidentally.

---

# 66. Never trust client-derived totals

Imagine an e-commerce form submits:

```json
{
  "items": [
    { "id": 42, "quantity": 2 }
  ],
  "total": 1.00
}
```

The client says:

```text
total = $1
```

Should the server believe it?

No.

The server should calculate:

```text
product prices
+
discounts
+
tax
+
shipping
=
authoritative total
```

The browser is a presentation environment, not the source of truth for security-sensitive calculations.

---

# 67. Same rule for roles

The client says:

```json
{
  "role": "admin"
}
```

Does that mean the user is admin?

No.

Roles and permissions should derive from trusted server-side state.

This leads to a broader principle:

> **Never let the client decide facts that the server must trust.**

---

# 68. Security and state management

We've learned many state systems:

```text
useState
useReducer
Context
URL state
TanStack Query
external stores
optimistic state
React Actions
```

None of them are security boundaries.

For example:

```js
const [isAdmin, setIsAdmin] =
  useState(false);
```

doesn't protect anything.

React state is:

```text
client-controlled application state
```

not:

```text
trusted authorization state
```

The distinction is crucial.

---

# 69. A useful trust map

For every piece of data, classify it.

For example:

```text
                         TRUST
                           ↓
database authorization ─ trusted
server session ───────── trusted
validated server state ─ trusted-ish
API response ─────────── must validate assumptions
URL params ───────────── untrusted
form data ────────────── untrusted
localStorage ─────────── untrusted
React state ──────────── untrusted
DOM values ───────────── untrusted
```

"Trusted-ish" is deliberately cautious.

Even data from your own API can be wrong due to bugs or compromised systems.

---

# 70. Security review through data flow

For any feature, trace:

```text
SOURCE
 ↓
TRANSFORMATION
 ↓
TRUST BOUNDARY
 ↓
SINK
```

Example:

```text
user comment
 ↓
Markdown parser
 ↓
sanitizer
 ↓
TrustedHTML
 ↓
dangerouslySetInnerHTML
```

Another:

```text
form
 ↓
HTTP request
 ↓
server validation
 ↓
authorization
 ↓
database mutation
```

Another:

```text
URL ?redirect=
 ↓
validation
 ↓
internal-path check
 ↓
navigate
```

This is a much more scalable security method than memorizing a list of "bad React functions."

---

# 71. Security checklist for a production React app

Think through these categories:

```text
Rendering
 ├─ Are user values rendered as text by default?
 ├─ Where is raw HTML allowed?
 └─ Is rich content sanitized?

Browser
 ├─ HTTPS?
 ├─ CSP?
 ├─ frame protection?
 └─ Trusted Types where appropriate?

Authentication
 ├─ How are sessions stored?
 ├─ HttpOnly?
 ├─ Secure?
 └─ SameSite policy?

Requests
 ├─ CSRF protection where needed?
 ├─ CORS correctly configured?
 └─ Rate limits?

Authorization
 ├─ Server-side permission checks?
 ├─ Resource ownership checks?
 └─ Tenant isolation?

Data
 ├─ Input validation?
 ├─ Size limits?
 ├─ Sensitive-data minimization?
 └─ Safe serialization?

Dependencies
 ├─ Lockfile?
 ├─ Security updates?
 ├─ Dependency review?
 └─ Build isolation?

Operations
 ├─ No secrets in client bundles?
 ├─ Safe logs?
 ├─ Error reporting?
 └─ Environment separation?
```

---

# 72. The most important distinction in this entire lesson

There are two radically different questions:

### "Can the user interface do this?"

and:

### "Can the user actually do this?"

For example:

```text
Delete button hidden
```

answers:

```text
Can the UI expose the operation?
```

Whereas:

```text
DELETE /projects/42 → 403
```

answers:

```text
Can the actor actually perform the operation?
```

Security is primarily concerned with the second.

---

# 73. React's role in security

React helps with:

```text
safe text rendering
component boundaries
escaping by default
```

React does **not** automatically solve:

```text
authentication
authorization
CSRF
database security
secret management
dependency security
server validation
rate limiting
resource ownership
```

So don't think:

```text
React application security
```

Think:

```text
browser security
+
React security
+
HTTP security
+
server security
+
data security
+
infrastructure security
```

---

# 74. One final architecture

A security-conscious production React application might look like:

```text
                         INTERNET
                             │
                         HTTPS/TLS
                             │
                        ┌────▼────┐
                        │ Browser │
                        └────┬────┘
                             │
                 CSP / Trusted Types
                             │
                      React application
                             │
                ┌────────────┴────────────┐
                │                         │
             UI state               Server requests
                │                         │
         routing / forms            CSRF / session
                                          │
                                   Authentication
                                          │
                                   Authorization
                                          │
                                     Validation
                                          │
                                    Business logic
                                          │
                                      Database
```

And around the entire system:

```text
dependency security
logging
monitoring
rate limiting
secret management
CI/CD security
```

That's the production security mental model.

---

# 75. Final mental model

When building a React feature, ask:

```text
1. Who controls this data?
2. Can the browser modify it?
3. Is it rendered as text or interpreted as markup/code?
4. Does it cross a server boundary?
5. Does the server validate it?
6. Does the server authorize the actor?
7. Could another user access the resource?
8. Could a third-party script or dependency see it?
9. What happens if one security layer fails?
```

The deepest lesson is:

> **Security is about controlling what an attacker can cause the system to believe, execute, access, or change.**

React is one layer of that system.

The next topic is **observability and reliability**: logging, metrics, tracing, error monitoring, production error boundaries/root callbacks, correlation IDs, performance monitoring, alerting, retries, failure budgets, and how to actually understand a React application once real users are running it.