## What was added

**Coupons system** — `Coupon` and `CouponRedemption` models (new, added to schema and seed).
- `validateCoupon` checks a code's existence/expiry before checkout
- `createOrder` now accepts an optional `couponCode`, applies the discount inside the same transaction, records the redemption, decrements coupon stock
- `getCoupons`/`getCouponInventory` expose public code lists and admin-facing stock
- Frontend: `Coupons.jsx` (codes/inventory toggle view), `Coupon.jsx` (apply-at-checkout input)

**Serverless functions** (Task 4, Task 1)
- `checkAvailability` — GET, stock-vs-requested-quantity check
- `sendConfirmationEmail` — POST, manually triggerable order confirmation (now effectively superseded by the webhook below, but still usable standalone)

**Webhooks** — event-driven, internal, triggered via `fetch` after a DB write commits. All three are secured with a shared-secret header (`x-webhook-secret` checked against `WEBHOOK_SECRET`), return 401 on mismatch.
- `order-created` → sends order confirmation email
- `product-created` → matches users who've bought in that category before, upserts `Notification` rows, triggers promo digest send
- `stock-low` → fires when a cart addition/update crosses the low-stock threshold (`LOW_STOCK_THRESHOLD` env var, default 5), emails `ADMIN_ALERT_EMAIL`

**Scheduled jobs** (`node-cron`)
- Reservation cleanup — every 5 min, releases `CartItem` rows older than 20 min back to `Product.stock`; ties directly into the cart reservation model from Task 1
- Promo email digest — daily at 9am, batches pending `Notification` rows per user into one email

**Schema/seed**
- Added `Coupon`, `CouponRedemption`, `Notification` models
- `seed.js` now seeds 5 sample coupons
- Cleanup order updated: `couponRedemption`/`notification` cleared before `coupon`/`user` to respect FK constraints

## Task 4: Architecture Decision — Monolith vs Microservices vs Serverless

**Core commerce stays monolithic.** Auth, products, cart, checkout, orders stay in one Express backend.
- Checkout touches Product, CartItem, Order, Coupon in one transaction
- Splitting these apart means distributed transactions for what's fundamentally one atomic operation
- Keeping them together keeps consistency simple

**Reviews and Notifications extracted as microservices.**
- No dependency on the checkout path
- Different load profile — read-heavy/event-triggered, independent of purchase volume
- Notifications specifically: owns its own persistent state (the Notification table) driven by events from the main app, rather than being a stateless pass-through action
- Only seams where splitting cost < staying-together cost

**Why stock-low and order-created stayed in the monolith, not extracted:**
- Both are stateless, single-action webhooks — receive an event, send one email, done
- No persistent state of their own to justify a separate database
- Extracting them adds a network hop and a second deployable to redeploy/monitor, with no load-pattern or release-cadence benefit over keeping them in the monolith
- They already fully satisfy the serverless/event-driven requirement (Task 4) regardless of which codebase they live in — extraction is an architecture decision (Task 3), not a Task 4 requirement

**Event-driven work goes serverless.** Confirmation emails, stock checks, coupon validation, low-stock alerts.
- Triggered by an event, does one small job, exits
- No idle server cost, auto-scales under bursty load (e.g. many orders during a sale)

**Recurring background work uses cron, not serverless.** Reservation cleanup, promo email digest.
- Needs to run on a timer regardless of user activity
- Fits scheduled jobs better than event triggers

**Why not extract more (e.g. Orders, Products):**
- Read/written together constantly (stock checks during checkout, order history joins)
- No meaningfully different load pattern or release cadence between them
- Splitting now = more network calls, more distributed-transaction risk, no real benefit

## Task 4: Cold Starts

**What it is:** delay on the first request to a serverless function after it's been idle — the platform has to spin up a fresh container before running your code.

**What causes it:**
- Function torn down after inactivity to free resources
- Next call has to init a new container, load the runtime, reconnect Prisma to Postgres — before the handler even runs

**What we observed:**
- First call after idle time: noticeably slower (hundreds of ms to a couple seconds)
- Calls within the same warm window: fast, container reused
- Frequently-hit functions (`checkAvailability`) stay warm more often; rare ones (webhooks) cold-start more often

**Why it shaped our design:**
- Scheduled jobs run via `node-cron` on the always-on server, not as serverless functions — no user is waiting on them, so a cold-start delay there buys nothing
- Serverless fits the email/webhook functions specifically because a real user triggered them and the occasional delay is an acceptable tradeoff against not paying for an idle server otherwise

## Task 2: Cloud Service Classification

| Service | Category | Why |
|---|---|---|
| Vercel (frontend hosting) | PaaS | We deploy code, Vercel manages servers/scaling/runtime |
| Vercel (backend/API) | PaaS | Same — we write handlers, platform handles infra |
| Vercel Functions (serverless) | PaaS | We provide function code, platform manages execution environment |
| Vercel Cron | PaaS | Scheduling infra managed by platform, we just define the job |
| Supabase (PostgreSQL) | PaaS | Managed DB — we don't manage the underlying server/OS |
| MongoDB Atlas (activity logs, reviews) | PaaS | Same reasoning — managed database service |
| GitHub | SaaS | Fully hosted product, we just use it as-is |
| GitHub Actions (CI/CD) | PaaS | We define pipeline config; execution infra is managed |
| SMTP email provider | SaaS | Fully hosted email-sending product, no infra managed by us |
| Docker Compose / Dockerfiles / .dockerignore (added in the last project) | IaaS | We define and manage the container environment directly, not abstracted away by a platform |

## Deployment Infrastructure

**Regions**
- Primary deployment region: closest Vercel edge region to majority user base
- Backup region: Vercel's multi-region edge network handles this automatically — no manual failover config needed on our side

**Scaling strategy**
- Frontend/backend: automatic horizontal scaling via Vercel (serverless-style scaling per request)
- Database: Supabase connection pooling handles concurrent load; vertical scaling available if needed as usage grows
- Serverless functions: scale per-invocation automatically, no manual capacity planning

**High availability**
- Vercel's edge network provides redundancy — no single point of failure at the hosting layer
- Supabase Postgres runs with built-in redundancy/backups on their managed infrastructure
- Application itself has no server to keep alive — stateless request handling means any instance can serve any request

**Single point of failure**
- The Postgres database is the main SPOF — if Supabase has an outage, both frontend and backend degrade since almost everything reads/writes through it
- Mitigated by Supabase's own HA guarantees, not something we manage directly at our layer

## CDN & Shared Responsibility

**CDN**
- Vercel's edge network serves static frontend assets (JS bundles, images, CSS) from locations close to the user
- Reduces latency — users don't all hit one origin server
- Reduces origin load — static content never reaches our backend at all

**Shared responsibility model**
- **Provider (Vercel/Supabase) handles:** physical infrastructure, networking, hardware availability, OS patching, platform-level security
- **We handle:** application code, database security (schema, access rules, query safety), user access control (auth/roles), secrets management, keeping dependencies updated
- Split in practice: if a request never reaches our code (e.g. DDoS at the edge), that's the provider's layer; if it's a bug in our auth logic, that's ours

## Kubernetes Concepts → ShopSphere Mapping

| K8s Concept | What it means | ShopSphere equivalent |
|---|---|---|
| Pod | Smallest deployable unit, one or more containers | One running instance of the backend (or Review Service) |
| Node | A machine (VM) that runs Pods | A worker machine in the cluster hosting our containers |
| Deployment | Manages desired Pod state, replicas, rollouts | Declares "run N copies of the backend, keep them healthy" |
| Service | Stable network endpoint routing to Pods | Internal address the frontend/Review Service use to reach the backend regardless of which Pod handles it |
| Ingress | External traffic entry point, routing rules | Routes public requests to the right Service (e.g. `/api` → backend Service) |
| Self-healing | Restarts/replaces failed Pods automatically | If a backend Pod crashes, Kubernetes replaces it without manual intervention |
| Auto-scaling | Adjusts replica count based on load | More backend Pods spun up automatically during high traffic |
# testing CI ###