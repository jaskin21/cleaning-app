# Cleaning Business App — Project Decisions

_Client project. Two-sided marketplace: Customers book cleaning services, Providers fulfill them._

## 1. Product Overview

Based on the original flowchart, the app covers five phases:

1. **Promotion & Discovery** — marketing funnel (ads, QR, referrals) into app download
2. **Sign Up / Register** — separate flows for Customer (fast: OTP → profile → address) and Provider (heavier: OTP → profile → documents → background check → admin approval)
3. **Booking & Service Flow** — customer books → providers notified by location/service type → first to accept gets the job → live status (en route → arrived → in progress → completed)
4. **Payment Flow** — fee breakdown → payment method → charge → receipt → payout to provider
5. **Ratings, Reviews & Loyalty** — post-service reviews, loyalty points, repeat booking, referrals

## 2. Platform Decision

**Status: Decided**

| App | Platform | Why |
|---|---|---|
| Customer app | React Native (Expo) | Client leans toward mobile; equal priority with provider side; booking/payment UX benefits from native polish |
| Provider app | React Native (Expo) | Needs reliable push notifications ("new job available") and background GPS ("provider en route") — both weak/unreliable on web |
| Admin dashboard | Next.js (web) | Admins work at a desk; approving providers, managing disputes/bookings/payments is a desktop workflow |
| Backend/API | Shared (Node/Express or Next.js API routes) | Serves all three clients from one source of truth |

**Context that drove this decision:**
- Client's expectation: leaned mobile, left the final call to us
- Both Customer and Provider sides are equally high priority from day one
- Budget/timeline: flexible, can be phased — so no pressure to cut to a web-only MVP first

## 3. Still To Decide (next conversation)

- [ ] Database choice (Postgres via Supabase/Neon, etc.)
- [ ] Auth approach (OTP provider, session/JWT strategy)
- [ ] Real-time job matching & notifications (Firebase Cloud Messaging, WebSockets, etc.)
- [ ] Payments provider (Stripe Connect for two-sided payouts, or local PH payment rails)
- [ ] Background check / document verification flow (manual admin review vs 3rd-party API)
- [ ] Detailed data model (Users, Bookings, Services, Payments, Reviews)
- [ ] MVP scope cut — which of the 5 phases ship in v1 vs later
- [ ] Phased timeline / milestones

## 4. MVP Build Order (from earlier discussion)

1. Narrow MVP scope (defer loyalty/referrals, automate less on background checks)
2. Define data model & core entities
3. Finalize stack (DB, auth, payments)
4. Design Customer + Provider dashboards/screens
5. Build booking + notification loop first (highest-risk, highest-value)
6. Wire up payments last
7. Ship, test with real users, iterate
