# 🎨 ArtMarket Connector - Project Context

## 📌 Overview
A standalone web application allowing artists & galleries to upload, manage, and publish artworks to a WooCommerce marketplace **without direct WP/WC access**. 
- Local PostgreSQL acts as the **source of truth**
- 15-minute batch sync pushes approved artworks to WooCommerce
- WC webhooks sync stock/order data back to local DB
- Simplified UI for creators; admin-controlled pricing, moderation, & rental tiers

## 🛠️ Tech Stack
| Layer | Technology |
|-------|------------|
| Frontend | Next.js 14+ (App Router), TypeScript, Tailwind CSS, React Hook Form, Zod |
| Backend | Next.js Server Actions, Route Handlers, Node.js Worker (Render) |
| Database | PostgreSQL via Supabase (Row Level Security enabled) |
| Storage | Supabase Storage (originals + processed WebP) |
| Queue/Cron | Upstash Redis + BullMQ (15-min sync + retry logic) |
| Auth | Supabase Auth or Clerk (TBD based on setup preference) |
| Image Processing | Sharp.js (resize, compress, WebP conversion) |
| Hosting | Vercel (frontend) + Render (cron/worker) |

## 🔑 Core Business Rules
1. **Creators NEVER access WooCommerce directly**
2. **Pricing Control (Artist-Set):**
   - `base_price` = list price
   - `price_floor` = minimum acceptable price (promotions cannot go below this)
   - `price_ceiling` = optional maximum price
   - Constraint: `price_floor ≤ base_price ≤ price_ceiling`
3. **Rental Model:**
   - Single toggle: `Available for Rent: Yes/No`
   - Duration options: 3, 6, 9, 12 months (global tiers applied at checkout)
   - One-time upfront billing; add-ons (insurance, installation, B2B) handled admin-side
4. **Image Requirements:**
   - 3–8 images required, ≤5MB each
   - Formats: JPG, PNG, WEBP
   - Auto-convert to WebP, generate thumbnails
   - Alt-text mandatory per image
5. **Compliance:**
   - GDPR consent timestamp required before submission
   - Media usage license must be signed/acknowledged
6. **Validation & Errors:**
   - Inline field explanations on failure
   - Backend re-validates on submit
   - Sync failures logged + retry up to 3 times

## 🗄️ Database Schema Summary
*(Full SQL in `db/migrations/001_initial_schema.sql`)*
- `users`: id, email, role (`artist|gallery|admin`), display_name, external_auth_id
- `artworks`: id, creator_id, title, description, base_price, price_floor, price_ceiling, rental_enabled, status, wc_product_id, gdpr_consent_at, media_license_signed
- `artwork_images`: id, artwork_id, original_url, processed_url, thumbnail_url, alt_text, sort_order
- `rental_pricing_tiers`: id, duration_months, monthly_percentage, is_active
- `review_logs`: id, artwork_id, reviewer_id, action, notes, field_changes (JSONB)
- `sync_jobs`: id, artwork_id, direction, status, attempts, next_retry_at, payload (JSONB), error_message

## 🔄 Workflow & Statuses
`draft` → `pending_review` → `approved` → `published`
- `changes_requested`: Admin requests edits → creator resubmits
- `rejected` / `archived`: Manual admin actions
- Auto-save drafts every 30s
- Admin can edit fields, add internal notes, bulk approve/reject

## 🔌 Sync & WooCommerce Integration
- **Direction 1 (DB → WC):** 15-min cron fetches `approved` items → maps to WC REST API → POST `/wp-json/wc/v3/products`
- **Payload includes:** standard fields, custom `meta_data` for technique/year/dimensions, rental config JSON, price overrides/sale_price
- **Direction 2 (WC → DB):** Webhooks update `stock_quantity`, `order_status` in local DB
- **Error Handling:** Logs to `sync_jobs`, exponential backoff (1m → 5m → 15m), max 3 attempts, dashboard + email alerts on exhaustion
- **Conflict Rule:** DB owns product data; WC owns stock/order data. Timestamp-based reconciliation.

## 🤖 AI Development Guidelines
1. **Step-by-step only:** 1 route / 1 table / 1 feature per prompt
2. **Always commit after verification:** `git commit -m "feat: <scope>"`
3. **Use Server Actions for mutations**, Route Handlers for external APIs
4. **Enforce Zod validation** on both frontend & backend
5. **Never hardcode secrets**; use `process.env` + `.env.local.example`
6. **Request tests** for complex logic (sync mapper, price floor math, image validation)
7. **Reference this file** in every prompt to maintain business rule alignment

## 🔐 Environment Variables
```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_STORAGE_BUCKET=artwork-media

WC_API_URL=https://yourmarketplace.com
WC_CONSUMER_KEY=
WC_CONSUMER_SECRET=

REDIS_URL=redis://...
SYNC_INTERVAL_MINUTES=15
MAX_RETRY_ATTEMPTS=3

NEXT_PUBLIC_APP_URL=http://localhost:3000