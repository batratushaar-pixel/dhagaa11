# Fix Report — dhagaa-luxury-build-v2

## Credentials wired (see `.env`)

**Supabase (Lovable Cloud project `tushaarbatra6` / ref `gciostgufqgpnacmuljp`)**
- `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SERVICE_ROLE_KEY` (server-only)
- `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, `VITE_SUPABASE_PROJECT_ID` (browser)

**Razorpay LIVE**
- `RAZORPAY_KEY_ID = rzp_live_T8GogHqN8ZRw6R` (server + optional VITE mirror)
- `RAZORPAY_KEY_SECRET = 1n47d9hvJjaPhDOL7KwROQm1` (server-only)
- `RAZORPAY_WEBHOOK_SECRET = ""` — set this after you create a webhook in
  Razorpay Dashboard → Settings → Webhooks pointing to
  `https://<your-domain>/api/public/razorpay-webhook`.

## Why the three issues happened

1. **`Missing Supabase environment variable(s): SUPABASE_SERVICE_ROLE_KEY`** —
   `.env` was missing the service key. `src/integrations/supabase/client.server.ts`
   throws on import, which crashed the checkout server function before it ever
   spoke to Razorpay. Fixed.

2. **Razorpay "Pay" button not working** — same root cause. Server function
   `placeOrder` in `src/lib/checkout.functions.ts` throws at line ~165 when
   `RAZORPAY_KEY_ID` / `RAZORPAY_KEY_SECRET` are missing, so `Razorpay.open()`
   never runs on the client. Both keys are now set. **Restart `bun run dev`
   after unzipping** so Vite picks up the new `.env`.

3. **Products disappear on reload** — NOT a localhost issue. Products live in
   Supabase; the same database serves dev and prod. New products default to
   `status: "draft"` (`src/routes/admin/products.tsx`). The public shop/index
   queries filter `.eq("status", "active")` (`src/routes/shop.tsx:57`,
   `src/routes/index.tsx:45`). Set **Status → Active** in the product editor
   and the product will stay visible everywhere.

## IMPORTANT security note

Your Razorpay LIVE secret key and the Supabase service_role key are now inside
`.env`. That is fine for local development, but:
- **Never commit `.env`** to Git (`.gitignore` already excludes it — verify).
- When you deploy, put these same variables in your hosting provider's
  environment settings, **not** in the source code.
- If this `.env` ever leaks (e.g. shared over chat/email), rotate both keys
  immediately from the Razorpay Dashboard and Supabase Dashboard.

I did NOT paste the secrets into any source file — they stay only in `.env`,
which the server reads at runtime. This is the correct pattern.

## Run

```
bun install
bun run dev
```
