# Cleaning Business App — Architecture & Tech Stack

_Client project. Two-sided marketplace: Customers book cleaning services, Providers fulfill them._

## 1. Product Overview

Five phases, from the original flowchart:

1. **Promotion & Discovery** — marketing funnel (ads, QR, referrals) into app download
2. **Sign Up / Register** — Customer (fast: OTP → profile → address) vs Provider (heavier: OTP → profile → documents → background check → admin approval)
3. **Booking & Service Flow** — customer books → nearby providers notified → first to accept gets the job → live status (en route → arrived → in progress → completed)
4. **Payment Flow** — fee breakdown → payment → receipt → payout to provider
5. **Ratings, Reviews & Loyalty** — reviews, loyalty points, repeat booking, referrals

## 2. Apps & Hosting

| App | Platform | Hosting |
|---|---|---|
| Customer app | React Native (Expo) | App Store / Play Store |
| Provider app | React Native (Expo) | App Store / Play Store |
| Admin dashboard | Next.js | Vercel |
| Backend API | Node/TypeScript, Express, REST | Railway |

## 3. Tech Stack Detail

### Customer App & Provider App (React Native / Expo)
- **Framework:** Expo (TypeScript, Expo Router — file-based routing, same mental model as Next.js App Router)
- **Styling:** NativeWind (Tailwind for RN — direct skill transfer from existing Next.js/Tailwind work)
- **Server state:** TanStack Query
- **Client state:** Zustand
- **Forms:** React Hook Form + Zod
- **Push notifications:** expo-notifications (wraps FCM/APNs)
- **Maps/GPS:** react-native-maps + expo-location, on **Google Maps Platform**
- **Auth:** Supabase Auth client SDK

### Admin Dashboard (Next.js)
- **Framework:** Next.js App Router, TypeScript
- **Styling:** Tailwind CSS + shadcn/ui
- **Server state:** TanStack Query
- **Forms/tables:** React Hook Form + Zod, TanStack Table

### Backend API
- **Runtime/framework:** Node.js + TypeScript, Express, REST architecture
- **Database:** PostgreSQL via **Supabase** (also provides Auth, Storage, Realtime)
- **ORM:** Prisma
- **File storage:** Supabase Storage (RLS-secured buckets — e.g. providers can only access their own documents; admins can view verification docs). Provider ID/documents, job before/after photos.
- **Realtime (job status, "en route" updates):** Supabase Realtime channels (Postgres change subscriptions) — no separate WebSocket service needed
- **Job matching / "first to accept wins":** Atomic DB update pattern (`UPDATE jobs SET status='accepted', provider_id=$1 WHERE status='pending' RETURNING *`) to prevent race conditions
- **Push notifications (server side):** Firebase Cloud Messaging
- **OTP/SMS provider:** Twilio (connected to Supabase Auth phone OTP)
- **Payments:** PayMongo — GCash + QR Ph (Maya). Payment Intent created server-side, customer completes payment via PayMongo's hosted flow/SDK, PayMongo webhook confirms success → backend updates booking status. Card/wallet credentials never touch our server (keeps us out of PCI DSS scope).

### Why Postgres + Railway over AWS (Cognito/DynamoDB/S3/Lambda)
Chosen for: native relational fit for Users/Bookings/Payments/Reviews, simple atomic transactions for job-claiming, built-in geospatial (PostGIS) for "nearby providers," built-in realtime, and much lower setup/ops overhead for a solo/small build. Fully portable later (Postgres → AWS RDS) if the client ever needs AWS specifically.

## 4. Decisions Made

| Area | Decision |
|---|---|
| Platform | 2 mobile apps (Customer, Provider) + 1 web admin dashboard + 1 backend |
| Mobile framework | React Native (Expo) |
| Backend architecture | Express, REST API |
| Database | PostgreSQL via Supabase |
| File storage | Supabase Storage |
| Hosting (backend) | Railway |
| Hosting (admin) | Vercel |
| Payments | PayMongo (GCash + QR Ph/Maya) |
| Provider verification (MVP) | Manual admin review of submitted documents |
| OTP/SMS | Twilio |
| Maps | Google Maps Platform |

## 5. Ownership & Scope — Resolved

- **App Store / Play Store accounts** — Project owner (not the client) controls these directly. Apple Developer Program ($99/yr) and Google Play Console ($25 one-time) are set up under the owner's accounts.
- **MVP scope** — Confirmed: loyalty points ship in **v2**, alongside referrals. v1 stays focused on core booking, payment, and ratings.
- **Phased timeline / milestones** — Deferred to a dedicated planning pass, to be done later.

## 6. MVP Build Order

1. Narrow MVP scope (confirm with client — see Open Items)
2. Define data model & core entities (Users, Providers, Bookings, Services, Payments, Reviews)
3. Set up Supabase project (DB, Auth, Storage, Realtime) + Railway backend + Twilio + Firebase + PayMongo accounts
4. Design Customer + Provider dashboards/screens
5. Build booking + notification loop first (highest-risk, highest-value)
6. Wire up payments last
7. Ship a rough version, test with real users, iterate

## 7. Data Model

Finalized — see `schema.prisma` for the full Prisma schema (source of truth for fields/types).

**Core tables:** `users` (base identity, `role` = customer/provider/admin), `customer_profiles`, `provider_profiles`, `provider_documents`, `addresses`, `services`, `service_extras`, `bookings`, `booking_extras`, `payments`, `reviews`, `loyalty_transactions` (v2).

**Key relationships:**
- `users` 1—1 `customer_profiles` OR `provider_profiles` (based on `role`)
- `provider_profiles` 1—many `provider_documents`
- `users` (customer) 1—many `addresses`
- `bookings` many—1 `users` (customer), many—1 `users` (provider, nullable), many—1 `services`, many—1 `addresses`
- `bookings` 1—many `booking_extras` — many `service_extras`
- `bookings` 1—1 `payments`
- `bookings` 1—1 `reviews`

**Design notes:**
1. Single `users` table with a `role` column (not separate customer/provider tables) — keeps auth simple, one login flow. Role-specific fields live in `customer_profiles`/`provider_profiles`. Admin is just another role.
2. `provider_id` on `bookings` is nullable until accepted — enables the atomic "first to accept wins" claim: `UPDATE bookings SET provider_id=$1, status='accepted' WHERE id=$2 AND status='pending'`. Only the first matching UPDATE succeeds; a second provider trying to accept gets zero rows affected → "job no longer available."

## 8. Phased Timeline / Milestones

Phase-based (not fixed calendar dates), given the flexible timeline. Estimates assume solo/small build effort.

| Phase | Focus | Apps involved | Est. effort |
|---|---|---|---|
| 0 | Foundation setup — Supabase, Railway, Firebase, Twilio, PayMongo sandbox, Google Maps key, repos, first Prisma migration | All four (project setup across the board) | ~3-5 days |
| 1 | Auth & onboarding — OTP sign up (both roles), customer profile/address, provider profile/documents, manual admin verification | Customer app + Provider app (sign up UI) · Admin dashboard (verification review screen) · Backend (auth/profile APIs) | ~1-2 weeks |
| 2 | Service catalog & booking core — admin CRUD for services/extras, customer booking flow | Admin dashboard (service/extra CRUD) · Customer app (booking flow UI) · Backend (booking API) | ~1-2 weeks |
| 3 | Provider job flow (core loop) — online/offline toggle, FCM push to nearby providers, atomic accept endpoint, Realtime status updates (en route → arrived → in progress → completed) | Provider app (toggle, accept, status updates) · Customer app (live status view) · Backend (matching logic, push, Realtime) | ~2-3 weeks, highest-risk phase |
| 4 | Payments — PayMongo (GCash + QR Ph/Maya), webhooks, receipts, manual payout tracking | Customer app (checkout UI) · Backend (PayMongo integration, webhooks) · Admin dashboard (payment/payout view) | ~1 week |
| 5 | Ratings & admin polish — reviews, admin views for bookings/disputes/approvals | Customer app (review submission) · Admin dashboard (bookings/disputes/approvals views) · Backend (review API) | ~1 week |
| 6 | Testing & soft launch — internal testing, TestFlight/Play internal tracks, client demo | All four | ~1-2 weeks |

**Total estimate: ~8-11 weeks of steady solo build time for v1**, before accounting for normal gaps (client feedback cycles, other work). Phase 3 is where the most iteration is likely, since it's the riskiest and most novel part of the system.

## 9. V2 — Loyalty & Referrals (Design)

**Loyalty Points**
- Customer earns points on every completed booking (e.g., 1 point per ₱50 spent — exact rate is a client business decision), credited via a `loyalty_transactions` row (`type: EARNED`) when a booking hits `completed`
- Points redeemable at checkout as a discount on a future booking (`type: REDEEMED`), capped at some % of total so it can't zero out a booking
- `customer_profiles.loyalty_points` holds the running balance for fast lookups; `loyalty_transactions` is the full audit trail

**Refer & Earn**
- Each customer gets a unique `referral_code` (auto-generated on `customer_profiles` at signup)
- New signups using a code create a `referrals` row (`status: PENDING`)
- Reward triggers only when the referred customer **completes their first booking** (not just signs up — prevents fake/incentiveless signups)
- Both referrer and referred customer receive a loyalty point bonus (`type: REFERRAL_BONUS`) or flat voucher — client's call on the exact reward

**Data model additions** (already reflected in `schema.prisma`): `referrals` table, `referral_code` on `customer_profiles`, extended `LoyaltyTransactionType` enum (`EARNED` | `REDEEMED` | `REFERRAL_BONUS`)

**Admin dashboard additions needed for v2:** loyalty program config (points-per-peso rate, redemption cap), referral reward config, and a view for tracking referral conversions.

**Apps involved:** Customer app (loyalty balance display, redeem-at-checkout UI, referral code sharing) · Admin dashboard (program config, referral tracking) · Backend (points/referral logic, schema). **Provider app is not involved** — this design rewards customers only, matching the original flowchart's "Earn Rewards / Refer & Earn" section. If the client wants a provider-side incentive program later, that'd be a separate design.
