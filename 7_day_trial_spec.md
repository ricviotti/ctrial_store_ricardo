<!-- 
Have minimum 3 labels
1. Priority: Low, Mid, High?
2. Level: 1, 2, 3, 4, 5?
3. Code: Html/CSS, Backend, Database?
4. Platform: Discord, Mobile?
5. A/B Test
Projects: Add to the most relevant project
-->

**Labels:** Priority: High, Level: 3, Code: Frontend/Backend, Platform: Web+Mobile, A/B Test

## Prerequisite
- Stripe account with **Link** enabled and test keys available  
- Feature flag service ready (e.g., LaunchDarkly/Config) — key: `paywall_trial_v1`  
- Analytics schemas ready (Segment/GA4/Amplitude/DB) with events below  
- Legal text approved for **Free Trial** + **Auto-renewal** disclosures  
- Subscription status API endpoint available (`isSubscriber`, `isInTrial`) 

## Summary
Introduce a **7-day free trial** for Colonist subscriptions (Plus/Premium/Elite) with:
- A new **carousel-style paywall** on the Store page  
- Additional **trial CTAs** surfaced on high-traffic screens (Main Menu + Profile)  
- Logic to **hide trial CTA** if user already has an active subscription or ongoing trial  
- Checkout routed via **Stripe Link** (web) and native WebView (mobile)  
- A/B test measuring uplift in paid conversion and trial engagement

## Details
* **Current Behavior:**
- Only “Subscribe” button (no trial)  
- No purchase entry points outside the Store (users must navigate manually)

* **Expected Behavior:**
  - New paywall with **“Try 7 Days Free”** CTA (teal/blue), pricing toggle, “Best Value” badge, and pagination dots
  - Trial starts without upfront charge; auto-renews at plan price after 7 days
  - Checkout via **Stripe Checkout + Link**; entitlement sync back to Colonist account
  - A/B Test: `control` (no trial) vs `trial_v1` (7-day trial)
  - Additional CTAs:
    - “TRY 7 DAYS PREMIUM FREE” button displayed on:
      - **Main Menu** (below user card)
      - **Profile Screen** (below user card)
      - **Victory Screen**
    - CTA hidden for users with `isSubscriber = true` or `isInTrial = true`
    - Clicking CTA deep-links to the Store paywall (variant = trial_v1)

## Key Knowledge
* **Problem to solve:** Low paid conversion (0.13–0.36% new→paid) on first session; high friction to purchase.
* **Goal:** Increase paid conversion by **~25%** and achieve **~20–25% YoY MRR uplift** after accounting for slightly higher first-cycle churn.
* **Reasons behind changes (feedback/research):**
  - Trial reduces upfront risk; industry data shows material lift in trial→paid conversion.
  - Surfacing CTAs on top-level screens increases funnel entry volume.  
* **Assumptions:**
  - Trial start rate from paywall: ~5%
  - Trial→paid: ~20% (conservative)
  - First-cycle churn +30% relative vs direct payers
  - Price points: Plus $6.99–$8.99/mo; Premium $14.99; Elite $19.99 (web/mobile vary)
* **Risks:**
  - Higher first-cycle churn; chargebacks if disclosures unclear
  - Apple external-link/checkout compliance (if applicable)
  - Incorrect CTA visibility logic (showing to subscribers)
* **Data, UI, design, functionality flows:**
  - **Events (all platforms):**
    - `paywall_viewed` { tier, variant, source, city, device }
    - `trial_start_clicked` { tier, price, variant }
    - `trial_started` { tier, price, provider: "stripe" }
    - `trial_converted_paid` { tier, price, cycle: 1 }
    - `subscription_renewed` { tier, cycle_n }
    - `subscription_canceled` { tier, reason, cycle_n }
  - **Funnel KPI:** paid conversion lift (%), trial start rate, trial→paid, 1-cycle churn, net MRR.
* **File to edit:**
  - Web: `web/src/pages/Store/Paywall.tsx`, `web/src/components/Paywall/TrialCTA.tsx`
  - Mobile: `apps/mobile/src/screens/StoreScreen.tsx`
  - Backend: `services/billing/stripeRoutes.ts`, `services/entitlements/entitlements.ts`
* **Function to edit:**
  - `createCheckoutSession(tier, isTrial)`
  - `handleStripeWebhook(event)` → `customer.subscription.created/updated`, trial end handling
  - `grantEntitlement(userId, tier, expiresAt)`
* **Edge Cases:**
  - Multiple trials per user (enforce 1 per account)  
  - Cancel during trial → access remains until trial end  
  - Failed charge at renewal → 24h grace period  
  - Upgrades during trial → trial ends immediately; paid starts  
  - Signed-out users clicking CTA → redirect to login → return to paywall

## Actions
- [ ] **Design**: Finalize CTA color & states (Primary: `#1ECBE1`, text `#FFFFFF`, pressed `#1AA7BF`), “Best Value” badge, pagination dots (active `#FFFFFF`, inactive `#FFFFFF` @ 30%).
- [ ] **Frontend – Web**: Implement `PaywallV2` with carousel (Plus/Premium/Elite), trial CTA, annual toggle, disclosure line (“No charge today. Auto-renews in 7 days.”).
- [ ] **Frontend – Mobile**: Mirror `PaywallV2` in React Native; open Stripe Checkout in in-app browser/WebView.
- [ ] **Backend – Stripe**: Add product/prices with `trial_period_days: 7`; implement `createCheckoutSession(..., trial=true)`.
- [ ] **Webhooks**: Handle `checkout.session.completed`, `customer.subscription.trial_will_end`, `invoice.payment_succeeded/failed`; update entitlements accordingly.
- [ ] **Entitlements**: Source of truth in DB; nightly reconciler to match Stripe status.
- [ ] **Feature Flag**: Gate `trial_v1` to 50% of eligible traffic (new users only).
- [ ] **Analytics**: Ship events listed above with user/tier metadata; add dashboard tiles for trial funnel & cohort churn.
- [ ] **Copy & Legal**: Localize CTA + disclosures (EN/ES/PT/DE). Add Terms & cancellation link.
- [ ] **QA**: Test flows: start/cancel trial, convert at day 7, failure retries, cross-platform sign-in, one-trial enforcement.
- [ ] **Rollout**: 10% → 25% → 50% with monitoring; rollback plan via flag.
- [ ] **Post-Launch Analysis**: 14 & 30-day readouts vs control (conversion, churn, net MRR).

## References
* Screenshots:
  
**Store Paywalls**
<img width="1170" height="2532" alt="paywall_elite" src="https://github.com/user-attachments/assets/29a2bbe4-da80-4286-8003-89163455caa0" />
<img width="1170" height="2532" alt="paywall_plus" src="https://github.com/user-attachments/assets/b7362769-5fb1-4117-a3c2-83137c5c687d" />
<img width="1170" height="2532" alt="paywall_premium" src="https://github.com/user-attachments/assets/546ae70f-0d3b-4ca8-a5e3-dc6e8879141d" />

**Main Screen Trial**
<img width="1170" height="2532" alt="Main_Screen_with_Trial" src="https://github.com/user-attachments/assets/0c2a7ce6-70c4-41d2-8abb-5be6f72937b4" />

**Profile Screen Trial**
<img width="1170" height="2532" alt="Profile_Screen_Trial" src="https://github.com/user-attachments/assets/3bbdcdaf-1496-4959-a536-daaa440b8e16" />

**Victory Screen**
<img width="1170" height="2600" alt="Victory Screen (1)" src="https://github.com/user-attachments/assets/bdf32fd5-1321-4922-a905-75e295fe403f" />


* Benchmarks:
  - RevenueCat — trial conversion insights (median ~45%): https://www.revenuecat.com/blog/growth/app-trial-conversion-rate-insights
  - RevenueCat (2023) — State of Subscription Apps: Industry benchmarks showing ~38% average trial→paid conversion for mobile apps.  
    https://www.revenuecat.com/state-of-subscription-apps-2023/
  - Duolingo Case Study (Nico Bottaro, Medium) — 7 Lessons on How Duolingo Increased Premium Users by 176% (7-day free trial cited as key growth lever).  
    https://medium.com/%40nicobottaro/monetization-7-lessons-on-how-duolingo-increased-premium-users-by-176-from-3-to-8-8-42e8d63b58f2
  - Lenny Rachitsky Newsletter — What is a Good Free-to-Paid Conversion Rate? (7-day trials outperform shorter or no-trial setups).  
    https://www.lennysnewsletter.com/p/what-is-a-good-free-to-paid-conversion
 

* Internal funnel (July): Web 0.13%, Discord 0.14%, Mobile 0.36% purchased at least once (screens provided)

## Reviewer
- **PM:** @ricardo (scope, KPIs, experiment design)
- **Dev Owner:** @frontend-dev, @backend-dev
- **Reviewer:** @payments-reviewer (Stripe), @qa-lead, @design-lead
- **QA:** @qa-owner
