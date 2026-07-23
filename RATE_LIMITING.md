# Rate Limiting: Phase 4C

## Implementation

Best-effort, **non-distributed** rate limiting in two layers:

### Server-side (in-memory, per-Worker)
`src/lib/rate-limit.server.ts` exposes `rateLimit(key, { limit, windowMs })`
backed by a module-scope `Map<string, Bucket>`. The key is built from the
client IP (`cf-connecting-ip` → `x-forwarded-for` fallback) plus a label.

Currently wired into:
- `previewCheckout`: 30 req / 60s / IP
- `placeOrder`: 8 req / 60s / IP  (covers checkout creation + coupon)

The same helper can be added to any other server function with one line:
```ts
enforce(rateLimit(await clientKeyFromRequest("contact"), { limit: 5, windowMs: 60_000 }));
```

### Client-side (per-browser, localStorage)
`src/routes/auth.tsx#clientThrottle` rejects:
- sign-in: 5 attempts / minute / browser / email
- sign-up: 3 attempts / 10 minutes / browser

Used because the actual auth calls go straight from the browser to Supabase
Auth (we cannot wrap them in our own server fn without losing the session
exchange).

## Known limitations

1. **Not distributed.** Cloudflare Workers run as many isolates; an attacker
   hitting different cold instances bypasses the in-memory counters.
2. **Lost on redeploy / cold start.** Buckets reset.
3. **Client throttle is bypassable.** Anyone clearing localStorage or using
   another browser/incognito tab gets a fresh budget. It only deters casual
   reload spam.
4. **Supabase Auth has its own limits** (server-side, leaky bucket) that act
   as the real defence for `signInWithPassword` / `signUp`. Configure them
   in the Supabase dashboard.

## Production-grade recommendation

For real abuse protection, replace the in-memory layer with one of:
- **Cloudflare WAF Rate Limiting Rules** (no code change; configure
  thresholds per route in the Cloudflare dashboard). Recommended first
  choice: already on the Cloudflare network in front of our worker.
- **Durable Objects**: strongly consistent per-key counter; ~1ms overhead.
- **Upstash Redis** or **Cloudflare KV** with `INCR`+TTL.

Until one of the above is wired in, treat the current limits as a polite
"please slow down"; not a security control.
