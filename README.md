## Gated B2B Marketplace on Supabase: Tiered Access, NDA Workflow, and RLS That Holds

A pattern reference from building a live gated acquisition marketplace for a Canadian M&A advisory firm. The product connects qualified buyers with confidential deal opportunities across a three-tier freemium model, with digital NDA capture, in-portal PDF generation, admin moderation, and bi-directional CRM sync.

This repo contains no client code, no deal data, no buyer profiles, no real financials, and no proprietary business logic. It's an architecture writeup of the patterns used to build a marketplace where "who can see what" is legally load-bearing.

---

### The core problem

Building a marketplace on Supabase is easy. Building one where a data leak means a lawsuit is different work.

The marketplace has three tiers of access:

- **Level 1 (public):** anyone can browse an anonymized deal pipeline (industry, revenue range, geography, no company identity)
- **Level 2 (verified buyer):** completes intake form, is admin-approved, can see full deal details after signing per-deal NDA
- **Level 3 (paid premium):** $500/year via Stripe subscription, gets 48-hour early access to new deals plus dedicated advisor support

Every access boundary above is enforced at the database layer, not the application. Application code can have bugs. RLS has to hold.

Below are the patterns that make this reliable.

---

### Pattern 1: RLS as the authoritative security boundary

Never trust the frontend. Never trust the API layer. Trust only what the database itself will enforce.

Every table in the schema has row-level security policies that read the calling user's `auth.uid()`, look up their role and level from the `profiles` table, and decide row visibility per record.

- Public pipeline table: policy allows SELECT for anyone but the `SELECT` is applied to a *view* that exposes only blind columns (see Pattern 2)
- Full deal table: policy allows SELECT only for admins OR approved buyers with an active interest request
- Interest requests table: policy allows SELECT for the requesting buyer OR admins
- NDA documents in storage: policy checks the signed-in user matches the NDA signer OR is admin

If the application layer forgets to check permissions, RLS still blocks the query. That's the point.

---

### Pattern 2: Column-level access via views (not RLS)

RLS can restrict *rows*. It can't restrict *columns* on the same table for the same query.

Solution: create a public database view that projects only the "blind" columns (revenue range, industry, geography, generic description) from the full deal table. Grant SELECT on the view to authenticated users. Full deal table with company identity and financials is only queryable by admins and approved buyers.

The frontend can't see something the query doesn't return. Column-level security via view separation is a clean solution when column-level RLS would be complex.

---

### Pattern 3: Privilege escalation guarded by BEFORE triggers

The `profiles` table has fields like `role` (buyer / admin) and `level` (level_1 / level_2 / level_3). If a buyer can UPDATE their own profile, what stops them from updating themselves to admin?

Two-layer defense:

1. RLS policy restricts UPDATE to only certain columns (name, avatar, etc.) on your own row
2. A BEFORE UPDATE trigger explicitly checks: if `role` or `level` changes and the caller is not admin, RAISE EXCEPTION and reject the update

Same pattern on `interest_requests.status` (only admin can change from `pending` to `approved`). Same pattern on any column where privilege escalation would matter.

The trigger is the belt. The RLS policy is the suspenders. Both are cheap. Missing either one is expensive.

---

### Pattern 4: Auto-linking imported buyers to signups

The advisory firm often pre-imports known-buyer profiles (from Zoho CRM, external referrals, past deal databases) before the buyer ever signs up. When those buyers eventually create an account, they should be auto-linked to their pre-existing profile with the correct approval level.

Pattern: a security-definer RPC `claim_imported_buyer_profile(email)` that runs at signup, looks for a pre-imported profile matching the email, links `profiles.id = auth.users.id` if found, and grants level_2 automatically if the admin marked the imported profile as `approved`.

RLS still blocks direct table access. The RPC is the only sanctioned way to do this operation. Auditable, controllable, and impossible to bypass.

---

### Pattern 5: NDA workflow with in-portal signature and PDF generation

Legally binding NDAs require:

1. The signer's intent captured (typed name + digital signature)
2. Timestamped and locked (can't be altered post-signing)
3. Delivered to both parties in a stable, non-editable format (PDF)
4. Retrievable by both parties later

Implementation:

- Signature captured client-side via `react-signature-canvas` (produces a PNG data URL)
- Full NDA rendered to canvas with `html2canvas` including the signature overlay and timestamp
- PDF generated with `jspdf` from the canvas
- PDF uploaded to Supabase storage in a bucket with strict RLS (signer OR admin can download, nobody else)
- Signature and PDF URL stored in `nda_signatures` table with timestamp, IP address, and reference to the interest request

Never store just the signature PNG. Always regenerate the PDF from a locked template + captured intent. The PDF is what a court would see.

---

### Pattern 6: Tier changes via Stripe webhook (never frontend)

Level 3 ($500/year premium) is a Stripe subscription. The user pays via Stripe Checkout. The frontend never changes the user's tier directly.

- Frontend: sends user to hosted Stripe Checkout, gets redirected back
- Stripe: charges the card, creates the subscription, fires a webhook to the Supabase edge function
- Edge function: verifies webhook signature (`stripe.webhooks.constructEvent` with the endpoint secret), looks up the customer, updates `profiles.level = 'level_3'`, records the subscription details

Same webhook handles cancellations, reactivations, and payment failures. Frontend just polls or subscribes to the profile row via Realtime to reflect the current tier.

Never trust a "user says they paid" signal from the client. The webhook is the source of truth.

---

### Pattern 7: Bi-directional Zoho CRM sync

The advisory firm's CRM (Zoho) is the source of truth for buyer relationships. The portal needs to push new buyer intakes into Zoho and pull deal status changes back.

- **Outbound:** `sync-buyer-to-zoho` edge function fires on buyer intake completion. Uses OAuth refresh-token flow to get a Zoho access token, upserts to the `Buyer_Leads` module.
- **Inbound:** `zoho-webhook-receiver` edge function accepts POST from Zoho on deal status changes, updates the corresponding portal record.

Refresh tokens are stored in Supabase secrets, never in code. Access tokens are minted per-request and never persisted. Idempotency keys prevent duplicate inserts on webhook retries.

---

### Pattern 8: Real-time admin notification feed with per-admin read state

Admins need to see new buyer intakes, new interest requests, and new NDA signatures the moment they happen. Solution:

- `admin_notifications` table with `event_type`, `event_data`, and `created_at`
- Row inserted by database trigger on every relevant event (new intake, new interest request, new NDA signature)
- Each admin has a `notifications_read` join table tracking which they've seen
- Frontend subscribes to `admin_notifications` via Supabase Realtime and pulls unread count per admin
- Admin marks read → row inserted into `notifications_read` → unread count drops

Deduplication is handled at the query level (`LEFT JOIN notifications_read WHERE ... IS NULL`). No stale notification counts, no double-marking.

---

### Stack this all runs on

- **Frontend:** React 19, Vite 8, TypeScript, Tailwind v4, React Router 7
- **Auth + DB:** Supabase (Postgres with RLS, Edge Functions, Storage, Realtime)
- **Payments:** Stripe (Checkout, Portal, webhooks, subscription lifecycle)
- **CRM:** Zoho (OAuth refresh-token flow, bi-directional sync)
- **Email:** Resend (transactional email for intake confirmations, NDA delivery, deal alerts)
- **Documents:** `react-signature-canvas` (signature capture), `html2canvas` + `jspdf` (PDF generation)

---

### About this writeup

I'm Hassan Nawaz, a senior full-stack engineer. I built this marketplace as an ongoing engagement with a Canadian M&A advisory firm (August 2025 to present). The live product handles real buyers, real deals, and real subscriptions daily.

Contact: [stellaflo.com](https://stellaflo.com) · [linkedin.com/in/ehassannawaz](https://www.linkedin.com/in/ehassannawaz)
