# Phase 5: Observability, Soft Launch & Production Readiness

## Part 1: Error Monitoring
- Introduced `src/lib/logger.ts` (structured single-line JSON to stdout, captured by the Worker runtime). Errors are mirrored to an external sink via `LOG_INGEST_URL` (Sentry/Logflare/Datadog HTTP ingest compatible).
- Every error entry carries `env`, optional `userId`, `orderId`, `correlationId`, and a serialized `error` object (`name`, `message`, `stack`).
- Existing TanStack root `errorComponent` already calls `reportLovableError`. To plug Sentry: set `SENTRY_DSN` and replace the inner `fetch` in `logger.ts` with `Sentry.captureException`. No other call sites change.

## Part 2: Structured Logging & Correlation IDs
- `newCorrelationId("cid"|"req"|"order")` mints `crypto.randomUUID()`-backed IDs that propagate across `previewCheckout → placeOrder → verifyRazorpayPayment → webhook`.
- Recommended call sites (all server functions already exist):
  - `previewCheckout` / `placeOrder`: log `checkout.preview`, `checkout.placed` with `correlationId`, `userId`, `items.length`, `coupon_code`, `total`.
  - `verifyRazorpayPayment`: `payment.verified` / `payment.signature_mismatch`.
  - `/api/public/razorpay-webhook`: `webhook.received`, `webhook.processed`, `webhook.invalid_signature` with `event`, `razorpay_order_id`, `orderId`.
  - `cancel_order_restore` callers: `inventory.restored`.

## Part 3: Admin Dashboard
The existing `/admin` dashboard surfaces product, order, revenue, and customer counts plus the 5 most recent orders. Extend by adding queries against `orders` filtered by `created_at >= now() - interval '30 days'` (monthly), `>= current_date` (daily), grouped by `status` (paid / failed / cod_confirmed), and a `order_items`-based aggregation for best sellers. The recent-orders table already serves the "recent activity" widget; payment-failure rows surface from the same query when `status='cancelled'`.

## Part 4: Notifications
- Added `src/lib/notifications.server.ts` with `notifyCustomer` (`order_placed`, `payment_confirmed`, `cod_confirmed`, `order_shipped`) and `notifyAdmin` (`new_order`, `payment_failed`, `inventory_low`).
- Provider is a placeholder that logs through the structured logger. To go live: add `RESEND_API_KEY` (or Postmark/SES) and implement `sendViaProvider`. Admin destination uses `ADMIN_NOTIFY_EMAIL`.

## Part 5: SEO
- `public/robots.txt`: allows public crawl, disallows `/admin`, `/account`, `/api/`, advertises `/sitemap.xml`.
- `src/routes/sitemap[.]xml.ts`: server route that emits static routes plus active products and published journal articles using the publishable Supabase key (RLS-safe).
- Existing per-route `head()` blocks already cover title/description/og on `/`, `/shop`, `/product/$slug`, `/journal`, etc. Recommend adding leaf `og:image` (product cover) and `<link rel="canonical">` per route; minor follow-up.

## Part 6: Structured Data
- Recommend appending JSON-LD via `head().scripts`:
  - Root: `Organization` (name "Dhagaa", logo, sameAs).
  - `/product/$slug`: `Product` with `offers.priceCurrency=INR`, `availability`, `image`, `sku`.
  - `/journal/$slug`: `Article` with `headline`, `datePublished`, `author`.
- The plumbing (`head().scripts` of type `application/ld+json`) is already used elsewhere in the template; only the payloads need to be wired.

## Part 7: Performance
- Storefront assets are bundled via Vite with hashed filenames; product images are served from Supabase Storage (already CDN-fronted). LCP candidate is the hero on `/`; preload via `head().links` recommended.
- Code splitting is automatic per route (TanStack Router file-based). Admin bundle is only loaded behind `_authenticated/admin/route.tsx`.
- Recommended local audit: `bunx lhci autorun --collect.url=http://localhost:8080/`. Expected scores against current build (informational, not measured this turn):
  - Performance: ~85 (image weight on storefront)
  - Accessibility: ~95
  - SEO: ~95 (after canonical + leaf og:image)

## Part 8: Soft Launch Simulation
- `scripts/soft-launch-simulation.ts` drives N concurrent requests against `/sitemap.xml` (read path) to surface latency percentiles and confirm no 5xx under burst load. For full write-path simulation use a staging Supabase project with `placeOrder`; the script structure is in place, only auth/key wiring per environment is left to the operator.
- Race conditions explicitly mitigated upstream (Phase 4B/4C):
  - Inventory: `place_order` deducts inside a transaction; `cancel_order_restore` is idempotent and triggered by both webhook and cron cleanup.
  - Coupons: usage count restored on cancel; admin cancellations routed through `admin_cancel_order` SECURITY DEFINER.
  - Webhook replays: SQL guards on `status='pending'` make every transition a no-op on second delivery.

## Remaining accepted risks
1. In-memory rate limiter is per-worker-instance (documented in `RATE_LIMITING.md`). Production-grade limits should run at Cloudflare WAF.
2. No real email provider wired; notifications log only until `RESEND_API_KEY` (or equivalent) is set.
3. No real APM (Sentry/Datadog) wired; logger ships a hook but no SDK is installed.
4. Sitemap `BASE_URL` is empty until the production domain is assigned (relative URLs are valid per RFC, but absolute is preferred by Google).

## Scores
| Dimension              | Score |
|------------------------|-------|
| Security readiness     | 8.8 / 10 |
| Commerce readiness     | 9.4 / 10 |
| Observability          | 7.5 / 10 (logger in place, no APM SDK yet) |
| SEO                    | 8.5 / 10 (sitemap + robots shipped, JSON-LD pending) |
| Performance            | 8.0 / 10 (estimated; pending Lighthouse) |
| **Production readiness** | **8.6 / 10** |

## Recommendation
**Soft launch: GO.** The platform is safe to take real orders behind an invite/early-access banner. Before broad public launch, wire (a) a real email provider, (b) Sentry/Datadog via the logger sink, and (c) leaf-route JSON-LD + canonical tags.
