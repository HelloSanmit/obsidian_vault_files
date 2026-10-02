
# Lesson 46 — Production Builds, Environment Variables, Assets, and Deployment

Until now, we've mostly worked in development mode:

```text
npm run dev
   ↓
dev server
   ↓
fast rebuilds
   ↓
browser
```

Production is a different environment.

A production React application typically looks more like:

```text
source code
   ↓
build
   ↓
optimized artifacts
   ↓
hosting/CDN/server
   ↓
browser
```

The important thing is understanding what actually changes between development and production.

---

# 1. Development is not your deployed application

In development, you might have:

```text
React source
Vite dev server
hot module replacement
source maps
unminified code
debugging
```

The browser is effectively connected to your source tree.

In production:

```text
React source
   ↓
bundler
   ↓
optimized files
```

and users receive those generated files.

So:

```text
development
→ execute source-oriented environment

production
→ execute built artifacts
```

That's an important mental boundary.

---

# 2. What `npm run build` actually does

Suppose:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  }
}
```

Running:

```bash
npm run build
```

roughly performs:

```text
source files
   ↓
module graph analysis
   ↓
transform
   ↓
tree shaking
   ↓
code splitting
   ↓
minification
   ↓
asset processing
   ↓
hashed output
```

Vite's current production build command generates a deployable bundle, with `dist/` as the default output directory for a standard client application. ([Vite](https://vite.dev/guide/build))

---

# 3. The browser does not receive your source tree

Suppose you have:

```text
src/
├── App.jsx
├── Dashboard.jsx
├── utils.js
└── main.jsx
```

After a production build, you might get:

```text
dist/
├── index.html
└── assets/
    ├── index-BKJ72x.js
    ├── Dashboard-Q3N8d.js
    └── style-X91kd.css
```

The exact filenames don't matter.

The important idea is:

```text
source modules
→ build artifacts
```

The browser consumes the artifacts.

---

# 4. Why filenames are hashed

You may see:

```text
index-BKJ72x.js
```

rather than:

```text
index.js
```

The hash is generally derived from the asset content.

If the file changes:

```text
index-BKJ72x.js
```

might become:

```text
index-M82pQa.js
```

This is extremely useful for caching.

---

# 5. Browser caching

Imagine the browser downloads:

```text
index-BKJ72x.js
```

The browser/CDN can cache it for a long time.

Then you deploy a new version:

```text
index-M82pQa.js
```

The URL changed.

The browser knows:

```text
old URL
≠
new URL
```

so it can fetch the new asset while keeping the old one cached.

This is one reason content-hashed filenames are so useful.

---

# 6. Why this matters with our code-splitting lesson

Remember:

```text
main.js
reports.js
admin.js
editor.js
```

Now imagine:

```text
reports chunk
```

has a content hash.

Deployment A:

```text
reports-Q8K31.js
```

Deployment B:

```text
reports-M2P90.js
```

A user with the old application may still request:

```text
reports-Q8K31.js
```

So a good deployment should not immediately destroy every asset referenced by older clients.

This is one reason static assets are often retained on CDNs for some period.

---

# 7. The deployment problem we discussed earlier

Imagine:

```text
Server sends HTML from version B
```

but:

```text
browser still has JavaScript from version A
```

Potentially:

```text
HTML B
+
JS A
```

Now you can get:

```text
hydration mismatch
missing exports
missing chunks
unexpected runtime errors
```

This is why versioned assets and deployment coordination matter.

---

# 8. What exactly gets deployed?

For a pure client-side Vite application:

```text
dist/
```

may be all you need.

For an SSR/full-stack application, you may instead have:

```text
client build
+
server build
```

For example:

```text
build/
├── client/
└── server/
```

The exact output depends on your framework.

React Router's current Framework Mode, for example, produces separate client/server build artifacts for server-rendered deployments and supports both static and full-stack hosting. ([React Router](https://reactrouter.com/start/framework/deploying))

---

# 9. Static hosting vs server hosting

This is a major deployment distinction.

## Static hosting

The host serves files:

```text
index.html
assets/*
```

Examples include:

```text
CDN
object storage + CDN
static hosting service
```

There is no application server executing React for each request.

This works well for:

```text
CSR
SPA
static generation
prerendered sites
```

---

## Server hosting

A runtime executes server code:

```text
request
 ↓
Node/server runtime
 ↓
React server rendering
 ↓
HTML
```

This is needed for architectures involving:

```text
SSR
Server Components
Server Functions
dynamic server loaders
personalized server rendering
```

The exact needs depend on your framework and setup.

---

# 10. React itself doesn't require a server

This is worth correcting mentally.

You can absolutely deploy a React application as static files.

For example:

```text
Vite build
 ↓
dist/
 ↓
static host
```

React's current guidance explicitly says React applications can be deployed as client-rendered or statically generated sites to CDN/static hosting, while frameworks can add server-side rendering where appropriate. ([react.dev](https://react.dev/learn/creating-a-react-app))

So:

```text
React
≠
Node server required
```

But:

```text
SSR/RSC/server functions
→ server runtime or compatible platform needed
```

---

# 11. SPA deployment has one famous problem

Suppose your app has:

```text
/dashboard
/projects
/settings
```

You're using client-side routing.

The browser does:

```text
/projects
```

Your web server receives:

```http
GET /projects
```

If the server only knows about:

```text
index.html
```

it might respond:

```text
404
```

even though your React Router knows that `/projects` is valid.

---

# 12. SPA fallback

For a traditional SPA deployment, the server often needs to say:

```text
unknown route
   ↓
serve index.html
```

Then:

```text
browser
 ↓
React Router
 ↓
/projects
```

takes over client-side.

The exact rewrite configuration depends on the hosting platform.

The concept is:

```text
server doesn't resolve application route
→ React router resolves application route
```

---

# 13. SSR changes this

For SSR:

```text
/projects
```

may actually be a real server-side application route.

So:

```text
GET /projects
 ↓
server route
 ↓
loader/RSC/SSR
 ↓
HTML
```

Now you don't simply rewrite every URL to one static `index.html`.

This is one reason static SPA hosting and server-rendered hosting have different deployment configurations.

---

# 14. Environment variables

This is one of the most important production topics.

Suppose you have:

```text
DATABASE_URL
STRIPE_SECRET_KEY
PUBLIC_API_URL
```

These do not all have the same security classification.

---

# 15. Server-only environment variables

Suppose:

```text
DATABASE_URL=postgres://...
```

A server runtime can read it.

Conceptually:

```js
const dbUrl =
  process.env.DATABASE_URL;
```

That's appropriate when this code runs on the server.

But you do **not** want that value in client JavaScript.

---

# 16. Client-visible environment variables are public

In Vite, environment variables beginning with:

```text
VITE_
```

are exposed to client-side code and statically replaced during the build. Vite explicitly warns that such variables should not contain secrets. ([Vite](https://vite.dev/guide/env-and-mode))

For example:

```text
VITE_API_URL=https://api.example.com
```

can be used in browser code.

But:

```text
VITE_DATABASE_PASSWORD=secret
```

is a security mistake.

The value becomes part of the client build.

---

# 17. Why build-time environment variables can surprise you

Suppose:

```text
.env.production

VITE_API_URL=https://api.example.com
```

and you run:

```bash
vite build
```

the value is embedded into the built client code.

So later changing a server environment variable doesn't necessarily modify an already-built static client bundle.

The important flow is:

```text
.env
 ↓
build
 ↓
embedded client value
 ↓
dist/
 ↓
browser
```

This is **build-time configuration**.

---

# 18. Build-time vs runtime configuration

This distinction is crucial.

### Build-time

```text
configure
 ↓
build
 ↓
bundle contains value
```

Common for:

```text
public API URL
public feature flags
build metadata
```

### Runtime

```text
server starts
 ↓
reads environment
 ↓
uses current value
```

Useful for:

```text
database credentials
private service URLs
secret keys
```

A static client application has much less runtime configurability because the client bundle has already been built.

---

# 19. Staging vs production

Suppose you have:

```text
development
staging
production
```

You might want:

```text
API_URL
```

to differ.

Vite supports mode-specific files:

```text
.env
.env.local
.env.staging
.env.production
```

and:

```bash
vite build --mode staging
```

can select staging configuration. ([Vite](https://vite.dev/guide/env-and-mode))

This gives:

```text
development
→ .env.development

staging
→ .env.staging

production
→ .env.production
```

The exact deployment pipeline can instead inject variables directly; you don't have to commit environment values into `.env` files.

---

# 20. Don't commit secrets

Typically:

```text
.env.local
```

should not be committed when it contains secrets.

You might have:

```gitignore
.env.local
.env.*.local
```

Vite's current documentation specifically recommends ignoring local environment files because they may contain sensitive variables. ([Vite](https://vite.dev/guide/env-and-mode))

---

# 21. `NODE_ENV` vs build mode

This is subtle.

You may have:

```text
NODE_ENV
```

and:

```text
Vite mode
```

They are related but not identical.

For example:

```bash
vite build --mode staging
```

can mean:

```text
NODE_ENV = production
MODE = staging
```

depending on environment.

Vite's current documentation explicitly distinguishes `NODE_ENV` from its mode system. ([Vite](https://vite.dev/guide/env-and-mode))

This matters when debugging environment-specific behavior.

---

# 22. Base paths

Suppose your app is deployed at:

```text
/my-app/
```

instead of:

```text
/
```

Then your generated assets need URLs like:

```text
/my-app/assets/app.js
```

rather than:

```text
/assets/app.js
```

Vite's `base` configuration controls the public base path and rewrites generated asset references accordingly. ([Vite](https://vite.dev/guide/build))

For example:

```js
export default defineConfig({
  base: "/my-app/"
});
```

Now the generated asset URLs understand:

```text
/my-app/
```

---

# 23. Why this bug feels weird

Your application may work locally:

```text
http://localhost:5173/
```

but fail in production:

```text
https://example.com/my-app/
```

because:

```text
JS
CSS
images
lazy chunks
```

are requested from:

```text
/assets/*
```

instead of:

```text
/my-app/assets/*
```

This becomes especially painful with code splitting because the initial bundle loads but later lazy chunks fail.

So when a production lazy route suddenly gives:

```text
ChunkLoadError
```

check deployment paths.

---

# 24. Asset URLs and dynamic imports

Remember:

```js
import("./Admin");
```

creates a separate chunk.

The generated application has to know where:

```text
Admin chunk
```

lives.

If your public base URL is wrong:

```text
initial app loads
```

but:

```text
Admin chunk
```

is requested from the wrong path.

This is a classic production issue.

---

# 25. CDN and static assets

A good production architecture often looks like:

```text
Browser
   │
   ├── HTML → application/server
   │
   └── JS/CSS/images → CDN
```

The CDN is excellent for immutable assets:

```text
app-abc123.js
styles-def456.css
image-ghi789.webp
```

because they can be cached close to users.

---

# 26. What should be cached aggressively?

Usually immutable, versioned assets:

```text
hashed JS
hashed CSS
static images
fonts
```

can often have very long cache lifetimes.

The URL changes when the content changes.

So:

```text
same URL
→ same content
```

becomes a safe assumption.

---

# 27. What about HTML?

HTML is different.

For an SSR application:

```text
/profile
```

may contain:

```text
user-specific content
```

so caching it aggressively could leak data.

Therefore:

```text
immutable asset caching
```

and:

```text
HTML/page response caching
```

should be treated separately.

---

# 28. Static asset vs application response

Think:

### Static asset

```text
/assets/app-A1B2.js
```

Usually content-addressed/versioned.

### Application response

```text
/dashboard
```

May depend on:

```text
user
cookies
database
locale
permissions
```

These require much more careful cache policy.

---

# 29. Server environment variables and deployment secrets

Suppose your production server requires:

```text
DATABASE_URL
SESSION_SECRET
PAYMENT_SECRET
```

A typical production setup is:

```text
deployment platform
    ↓
secret manager/environment
    ↓
server process
```

rather than:

```text
Git repository
    ↓
.env.production containing secrets
```

The actual platform differs, but the principle is universal:

> **Secrets belong in your deployment/runtime secret management system, not your source repository and not your client bundle.**

---

# 30. Build-time secret mistake

Suppose a build system has:

```text
SECRET_API_KEY=abc
```

and your client code references:

```js
import.meta.env.SECRET_API_KEY
```

Depending on configuration, that value can become part of the generated JS.

Then:

```text
secret
 ↓
bundle
 ↓
browser
```

The secret is no longer secret.

This is one of the easiest deployment mistakes to make.

---

# 31. Runtime configuration for client applications

Sometimes you need public configuration to change without rebuilding.

For example:

```text
API endpoint
feature flag
tenant configuration
```

A pure static application can't simply read:

```js
process.env.API_URL
```

at browser runtime unless you've built a mechanism for it.

Common strategies include:

```text
build-time injection
runtime config endpoint
generated config.js
server-rendered config
```

The exact technique depends on deployment architecture.

The key distinction is:

```text
static bundle
≠
runtime environment
```

---

# 32. Health checks

If you deploy a server-side React application, the infrastructure often needs a simple health endpoint such as:

```text
/health
```

which can answer:

```json
{
  "status": "ok"
}
```

This lets your platform know:

```text
server process alive?
```

That is different from:

```text
is database healthy?
```

or:

```text
is the whole application functional?
```

Health checks should be designed with those distinctions in mind.

---

# 33. Readiness vs liveness

Another production concept.

### Liveness

```text
"Is this process alive?"
```

### Readiness

```text
"Can this instance safely receive traffic?"
```

Suppose your server just started:

```text
process alive
database connection not ready
```

Liveness:

```text
✅
```

Readiness:

```text
❌
```

The infrastructure should potentially wait before routing user traffic to it.

This isn't React-specific, but it becomes part of production deployment.

---

# 34. Graceful shutdown

Suppose you deploy a new version.

The platform wants to stop:

```text
server A
```

while requests are still in progress.

A graceful server should ideally:

```text
stop accepting new traffic
 ↓
finish in-flight requests
 ↓
close DB connections
 ↓
close WebSockets
 ↓
exit
```

This is particularly important for real-time systems from Lesson 42.

---

# 35. Production logging

Development:

```js
console.log("user", user);
```

Production needs more structure.

You may want:

```json
{
  "level": "error",
  "requestId": "abc123",
  "route": "/projects/42",
  "message": "database timeout"
}
```

This makes logs searchable.

And remember from SSR:

```text
request ID
+
server error
+
client error
```

can help correlate a failure across environments.

---

# 36. Source maps

Production bundles are minified.

You might see:

```text
main.a8dj2.js:1
```

with unreadable stack traces.

Source maps let your error monitoring tools map:

```text
generated bundle
```

back to:

```text
source file
line
column
```

This is extremely valuable for production debugging.

But source maps themselves can reveal source code if publicly accessible.

So teams often configure how source maps are generated and where they are uploaded.

---

# 37. Error monitoring

Remember:

```text
onCaughtError
onUncaughtError
onRecoverableError
```

Now combine that with an external monitoring system.

Conceptually:

```text
React
 ↓
root error callback
 ↓
error monitoring service
 ↓
stack
component tree
request ID
release/version
environment
```

Now you can answer:

```text
Which release?
Which route?
Which user session?
Which component?
How frequently?
```

---

# 38. Release/version metadata

Suppose an error says:

```text
TypeError
```

Not enough.

Include:

```text
release = 2026.09.21-abc123
environment = production
```

Then you know exactly which build produced it.

This is particularly useful when multiple versions are temporarily alive during deployment.

---

# 39. CI/CD

A production React application shouldn't depend on:

```text
developer laptop
```

to build and deploy it manually.

A typical pipeline:

```text
git push
   ↓
CI
   ↓
install dependencies
   ↓
lint/typecheck
   ↓
tests
   ↓
build
   ↓
artifact
   ↓
deploy
   ↓
health check
```

This makes deployment repeatable.

---

# 40. Build should fail before deployment

Suppose TypeScript fails:

```text
Type error
```

The pipeline should stop.

Suppose tests fail:

```text
test failure
```

Stop.

Suppose production build fails:

```text
bundle error
```

Stop.

The principle is:

> **Move failure detection as early as possible.**

Don't discover a broken build after users receive it.

---

# 41. Dependency lockfiles

You likely have:

```text
package-lock.json
```

or:

```text
pnpm-lock.yaml
yarn.lock
```

These lock resolved dependency versions.

That matters for reproducible builds.

Otherwise:

```text
Monday:
dependency version A

Friday:
dependency version B
```

could produce different builds from the same source package manifest.

A deployment should know exactly what dependency graph it is building.

---

# 42. Dependency updates

Remember our current React version:

```text
React 19.3
```

When dependencies are updated, production should ideally go through:

```text
lockfile update
 ↓
CI
 ↓
tests
 ↓
build
 ↓
deployment
```

rather than:

```text
npm install on production server
```

and hoping nothing changed.

---

# 43. `npm install` vs reproducible installation

Production CI often uses a lockfile-respecting installation command such as:

```bash
npm ci
```

rather than allowing dependency resolution to change.

The exact package manager varies.

The principle:

```text
production build
→ deterministic dependency graph
```

is what matters.

---

# 44. Static deployment flow

For a basic Vite SPA:

```text
developer code
   ↓
vite build
   ↓
dist/
   ↓
upload to static host
   ↓
CDN
   ↓
browser
```

Request:

```text
GET /
```

returns:

```text
index.html
```

Then browser requests:

```text
assets/*.js
assets/*.css
```

Then React starts:

```text
createRoot()
```

---

# 45. SSR deployment flow

For a server-rendered React app:

```text
browser
   ↓
GET /products/42
   ↓
application server
   ↓
server-side data
   ↓
React render
   ↓
HTML
   ↓
browser
   ↓
hydrateRoot()
```

Then:

```text
client JS
```

takes over interactive parts.

---

# 46. Full-stack deployment flow

With RSC/Server Functions:

```text
browser
   ↓
route request
   ↓
server
   ├── Server Components
   ├── loaders/data
   └── HTML/RSC payload
   ↓
browser
   ↓
Client Components hydrate
```

Later:

```text
client interaction
   ↓
Server Function
   ↓
server
   ↓
database
```

So your deployment architecture follows the runtime architecture you learned earlier.

---

# 47. Deployment and WebSockets

For a real-time app, deployment must support:

```text
persistent connections
```

A purely static CDN host cannot execute:

```text
WebSocket server
```

by itself.

You may need:

```text
application server
+
load balancer
+
WebSocket-capable infrastructure
```

Potentially with a separate real-time service.

This is a good example of how feature requirements constrain infrastructure.

---

# 48. Horizontal scaling changes the real-time problem

Suppose you have:

```text
Server A
Server B
Server C
```

and clients connect to different instances.

Alice:

```text
→ Server A
```

Bob:

```text
→ Server B
```

Server A receives:

```text
Alice's new message
```

but Bob is connected to B.

How does B learn about it?

You may need:

```text
shared pub/sub
message broker
real-time service
distributed event bus
```

The lesson:

> **A WebSocket connection is local to a server/process unless you deliberately build cross-instance message propagation.**

This is why real-time deployment can become much more complicated than local development.

---

# 49. Static assets and multiple servers

Suppose Server A and B are running different application instances.

Both should be able to serve compatible static assets.

A common approach:

```text
CDN
 ↓
versioned assets
```

while servers handle:

```text
dynamic requests
```

This keeps static asset delivery independent of which server receives the request.

---

# 50. Blue-green / rolling deployment

Production deployments often keep old and new versions alive simultaneously.

Conceptually:

```text
Version A
████████

Version B
████
```

Traffic gradually shifts.

This reduces downtime.

But because users may have:

```text
old HTML
old JS
new HTML
new JS
```

at different moments, your application should tolerate some overlap.

This is another reason robust asset versioning matters.

---

# 51. Database migrations can be the real deployment hazard

Suppose code version B expects:

```text
users.display_name
```

but the production database still has:

```text
users.name
```

You deploy code first.

Now:

```text
application B
 ↓
database old schema
 ↓
failure
```

Therefore database migrations often need compatibility planning.

A common strategy is:

```text
expand
 ↓
deploy code compatible with both
 ↓
migrate data/schema
 ↓
contract/remove old fields
```

This is called an **expand-and-contract** style of migration.

Not React-specific, but very relevant to full-stack application deployment.

---

# 52. Frontend and backend deployment are coupled

Suppose React expects:

```json
{
  "name": "...",
  "avatarUrl": "..."
}
```

but backend deploys a breaking change:

```json
{
  "displayName": "..."
}
```

Your client can fail.

Therefore API changes should generally be backward-compatible when old clients may remain active.

This matters enormously in production because:

```text
not every browser upgrades simultaneously
```

---

# 53. API versioning

You may encounter:

```text
/api/v1/projects
/api/v2/projects
```

or versioning through headers/schema evolution.

You don't necessarily need explicit URL versions for every system.

The broader principle is:

> **Deployments should account for old clients communicating with newer servers and vice versa during the transition period.**

That means:

```text
additive API changes
optional fields
backward compatibility
```

are often safer than abrupt breaking changes.

---

# 54. Security headers

Production web applications commonly configure HTTP security headers such as:

```text
Content-Security-Policy
Strict-Transport-Security
X-Content-Type-Options
Referrer-Policy
```

The exact policy depends on your application.

For example, a Content Security Policy can restrict which scripts/resources the browser is allowed to load, reducing the impact of certain injection attacks.

These are server/browser security controls, not React APIs.

---

# 55. HTTPS

Production should generally use HTTPS.

Why?

Because without transport encryption:

```text
browser
   ↓
network
   ↓
server
```

traffic may be exposed or modified by attackers.

HTTPS is also required for things like:

```text
Secure cookies
```

and secure browser features.

Again, infrastructure, not React.

---

# 56. Logging user data can itself be a security problem

Bad:

```js
console.log({
  password,
  sessionToken,
  creditCard
});
```

Even server logs may be sensitive.

A production logger should avoid putting secrets into:

```text
logs
traces
error reports
analytics events
```

Observability is valuable.

Data minimization applies there too.

---

# 57. Production configuration has three audiences

Think about:

```text
1. BUILD SYSTEM
2. SERVER
3. BROWSER
```

A variable might belong to:

```text
build only
```

or:

```text
server only
```

or:

```text
client-visible
```

Confusion happens when those are mixed.

For example:

```text
DB_PASSWORD
```

must never cross into the browser.

While:

```text
PUBLIC_API_URL
```

may be deliberately public.

---

# 58. A good mental classification

Before adding a config value, ask:

```text
Does the browser need it?
```

If no:

```text
keep server-only
```

If yes:

```text
assume users can inspect it
```

That rule is incredibly safe.

---

# 59. Production architecture in one diagram

```text
                       SOURCE
                         │
                         ▼
                      CI/CD
                         │
               ┌─────────┴─────────┐
               │                   │
            client build       server build
               │                   │
               ▼                   ▼
              CDN               SERVER
               │                   │
          JS/CSS/images       SSR/RSC/API
               │                   │
               └─────────┬─────────┘
                         │
                         ▼
                       BROWSER
                         │
                  hydrate / execute
                         │
             ┌───────────┼────────────┐
             │           │            │
           React      Server Fn   WebSocket
             │           │            │
             └───────────┼────────────┘
                         ▼
                     SERVER/DATA
                         │
                         ▼
                      DATABASE
```

And around everything:

```text
monitoring
logging
security
caching
deployment versioning
```

---

# 60. What actually happens when a user opens your app?

Let's make the whole thing concrete.

Suppose they visit:

```text
https://example.com/projects/42
```

### Step 1 — DNS

Browser resolves the domain to an infrastructure endpoint.

### Step 2 — HTTPS

Secure connection established.

### Step 3 — request

```http
GET /projects/42
```

### Step 4 — server/CDN

Depending on architecture:

```text
static HTML
or
server render
```

### Step 5 — HTML

Browser parses:

```text
<h1>Project 42</h1>
```

### Step 6 — assets

Browser downloads:

```text
app-abc123.js
project-def456.js
```

### Step 7 — React startup

```text
hydrateRoot()
```

### Step 8 — client interaction

```text
button click
```

### Step 9 — mutation

```text
Server Function / API
```

### Step 10 — backend

```text
auth
validation
authorization
database
```

### Step 11 — fresh state

```text
query/loader/revalidation
```

### Step 12 — UI

```text
React render
```

You now understand almost every layer involved.

---

# 61. A production build isn't "a React app uploaded"

It's more accurate to think:

```text
Application source
        ↓
dependency graph
        ↓
compiled artifacts
        ↓
deployment environment
        ↓
network delivery
        ↓
browser runtime
```

This is why production React work often touches:

```text
JavaScript
HTTP
CDN
Docker
DNS
TLS
environment variables
database
logging
CI/CD
```

React is one layer of the system.

---

# 62. The most important deployment mistakes

```text
1.
Shipping secrets in client environment variables.

2.
Wrong SPA fallback configuration.

3.
Wrong asset base path.

4.
Deleting old hashed chunks during deployment.

5.
Caching personalized HTML as public content.

6.
Building against changing dependency versions.

7.
Deploying incompatible frontend/backend versions.

8.
Running SSR with shared mutable request state.

9.
No production error monitoring.

10.
No rollback strategy.

11.
Using static hosting for features that require a server.

12.
Ignoring database migration compatibility.
```

---

# 63. Rollbacks

Suppose release B breaks production.

A healthy deployment system should let you go:

```text
Version B
 ↓
problem
 ↓
rollback
 ↓
Version A
```

This is much easier when:

```text
build artifacts
are immutable/versioned.
```

You want:

```text
release A = known artifact
release B = known artifact
```

rather than:

```text
rebuild old source and hope
```

---

# 64. Observability should answer three questions

Production monitoring should help you answer:

```text
What broke?
```

Example:

```text
TypeError in ProjectEditor
```

```text
Who was affected?
```

Example:

```text
2.4% of project-edit requests
```

```text
Which release caused it?
```

Example:

```text
release abc123
```

That turns:

```text
"something seems broken"
```

into an actionable engineering problem.

---

# 65. Final production architecture

Here's the big picture we've built across the last several lessons:

```text
                         USER
                           │
                           ▼
                        BROWSER
                           │
              ┌────────────┼─────────────┐
              │            │             │
             HTML         JS         WebSocket
              │            │             │
              ▼            ▼             ▼
          SSR/RSC       React        external store
              │            │             │
              │       state/events       │
              │            │             │
              └────────────┼─────────────┘
                           │
                      Server/API
                           │
             ┌─────────────┼──────────────┐
             │             │              │
            Auth         Actions       Loaders
             │             │              │
             └─────────────┼──────────────┘
                           ▼
                        DOMAIN
                           │
                           ▼
                       DATABASE
```

Around it:

```text
CDN
Caching
CI/CD
Logging
Monitoring
Security
Deployment versioning
```

---

# 66. What to permanently remember

```text
Development
→ source-oriented environment

Production
→ built artifacts

vite build
→ creates deployable client artifacts

Hashed assets
→ reliable long-lived caching

Static hosting
→ files only

Server hosting
→ dynamic server execution

SSR/RSC/Server Functions
→ need server-capable infrastructure

VITE_* variables
→ client-visible in Vite

Client-visible env
→ assume public

Server secrets
→ runtime/server-only

Build-time config
→ embedded during build

Runtime config
→ read by running server

SPA fallback
→ unknown paths serve app entry

Base path
→ tells bundler where app is hosted

CDN
→ efficient delivery of static assets

Source maps
→ useful production debugging

CI/CD
→ repeatable build/test/deploy pipeline

Immutable artifacts
→ safer releases and rollbacks

Backward-compatible APIs
→ tolerate old clients during deployment

Health
→ infrastructure knows whether instance can serve traffic
```

And the most important production principle:

> **A React application is not just a collection of components. In production it is a system of build artifacts, browser code, server code, data, infrastructure, security boundaries, and versioned deployments.**

That completes the major **build/deployment** layer. We are now down to roughly **5–7 major lessons**, with smaller gaps folded into them. The remaining path is increasingly practical: **production testing/E2E, security hardening, observability, advanced React Router/RSC integration, then a complete capstone architecture that ties the entire course together.**