# Longevity Life Academy (LLA) Blueprint: Technical Handover

Owner: Omri Gitter, GiTA Agency (gitter.omri@gita-agency.com). Written 2026-11-02 (Asia/Jerusalem).
Purpose: let a different operator continue work on the LLA Blueprint web surfaces with zero questions.
This document contains no secrets, tokens or passwords. All credentials live in Cloudflare Worker secrets, Vercel env vars and Omri's credential vault.

Sources: `github.com/gitteromri-ux/lla-course-checkout` (HANDOVER.md, worker/CLOUDFLARE-INTEGRATION-SPEC.md, worker/README.md, dist/checkout.html, dist/assets/eteacher-ecomm.js, dist/_worker.js) and `github.com/Longevity-Academy/Longevity-Academy.github.io` (checkout/, blueprint/, assets/eteacher-ecomm.js, vercel.json), plus the julie-masterclass repo metadata. Items marked UNVERIFIED could not be re-read from code at writing time and must be confirmed before acting on them.

---

## 1. Surfaces: live URLs, repos, deploy methods

| Surface | URL | Repo / branch | Served by | Deploy method |
|---|---|---|---|---|
| Blueprint funnel + self-service checkout (dev/relaunch host) | https://longevitylifeacademy.pages.dev/ and https://longevitylifeacademy.pages.dev/checkout.html | https://github.com/gitteromri-ux/lla-course-checkout, branch `master`, folder `dist/` | Cloudflare Pages project `longevitylifeacademy` (account id `55eb74f4002b7237e393bd6980a1676a`) | DIRECT UPLOAD of `dist/` via Pages API (upload-token, check-missing, upload, upsert-hashes, POST deployments). NOT git auto-deploy. Pushing to GitHub changes nothing live. Script in repo HANDOVER.md ("Deploy — Cloudflare Pages"). |
| Same bundle on Worker host (legacy mirror) | https://lla-checkout.gitter-omri.workers.dev/ | same `dist/` | Cloudflare Worker `lla-checkout` (static assets) | Worker assets upload + PUT `workers/scripts/lla-checkout` (HANDOVER.md "Deploy — Worker"). |
| Main site | https://www.longevitylifeacademy.com/ | https://github.com/Longevity-Academy/Longevity-Academy.github.io, branch `main` (CNAME in repo root) | GitHub Pages | Git push to the default branch auto-publishes. `vercel.json` in root is a static build config only. |
| Main-site checkout | https://www.longevitylifeacademy.com/checkout/ | same repo, `checkout/index.html` | GitHub Pages | git push |
| Blueprint landing + checkout on main site | https://www.longevitylifeacademy.com/blueprint/ and https://www.longevitylifeacademy.com/blueprint/checkout.html | same repo, `blueprint/index.html`, `blueprint/checkout.html`, shared `assets/eteacher-ecomm.js` | GitHub Pages | git push |
| Enroll staging page | https://www.longevitylifeacademy.com/enroll-staging.html | same repo, `enroll-staging.html` | GitHub Pages | git push |
| Julie masterclass page | Julie masterclass site served from https://github.com/Longevity-Academy/julie-masterclass (`index.html`, `assets/`, `privacy.html`, `support/`); GitHub Pages. Golden tag: `tracking-golden-20260928`. Latest commits: `bf3b14d`, `ce0caf8`, `270106d` (October-date copy removals only; "retain November 14 and all protected logic"). | GitHub Pages | git push |
| Masterclass attendee offer (unlisted guest page) | path `/blueprint-attendees-7q4x/` | Was added to Longevity-Academy.github.io in commit `4030535` ("Replace unlisted guest page", 2026-10-27) with `blueprint-attendees-7q4x/attendee-form.js`, `blueprint/checkout.html`, `blueprint/assets/eteacher-ecomm.js`, and then REVERTED in `b4435d5` (2026-10-27). The matching attendee offer + checkout now live on pages.dev: lla-course-checkout commit `ac35c28` "Match masterclass attendee offer and checkout on pages.dev; preserve 48h+ coupon window" (2026-10-27 19:53 UTC); `dist/attendee-form.js`, `dist/attendee-pricing.js`, `dist/attendee-pricing.css`, `dist/exact-attendee-copy.js`. Verify the exact public URL with Omri before sharing it. | Cloudflare Pages direct upload | see row 1 |
| Meta CAPI relay | https://lla-ac-events.vercel.app/api/meta/capi | Vercel project `ac-events` (prj_3v7g4Xwve6OmXrcPCVQOWJOiN2SK) | Vercel serverless | Vercel deploy (requires Omri's deploy permission; see commit `5897dcd` note "awaiting Vercel deploy permission") |
| CRM proxy Worker (shared with all eTeacher sites) | https://eteacher-leads-proxy.gitter-omri.workers.dev | source in `lla-course-checkout/worker/src` (spec + README) | Cloudflare Worker `eteacher-leads-proxy` | `wrangler deploy`. HANDOVER.md marks this Worker as "NOT TOUCH" for casual edits: it holds the CRM secrets used by longevitylifeacademy.com. |

Cache rule: every delivered link must carry a cache-buster (`?v=<timestamp>`). Verify both pages.dev and workers.dev serve identical `dist/index.html` when deploying the funnel.

Compare links: https://github.com/gitteromri-ux/lla-course-checkout/commits/master and https://github.com/Longevity-Academy/Longevity-Academy.github.io/commits/main

---

## 2. Checkout architecture

### 2.1 Three-step flow (browser never calls the CRM directly)

```
Browser -> Cloudflare Worker eteacher-leads-proxy -> eTeacher CRM
Payment report: Browser -> Worker -> Cloudflare Queue -> queue consumer -> CRM SetECommercePayment
```

| Step | Worker route | CRM endpoint | What happens |
|---|---|---|---|
| 1. Enroll form (lead + order) | `POST /api/lead/ecomm` (`/staging/api/lead/ecomm` for staging) | `/Students/AddNewECommerceLeadAndOrder` | Creates lead + order, returns `{orderid, stid}` |
| 2. Checkout load | `POST /api/checkout/details` | `/Students/GetECommerceFirstPaymentDetails` | CRM mints the Airwallex PaymentIntent and returns `paymentId`, amount, currency, Airwallex `intent_id` + `client_secret`, PayPal SDK URL |
| 3. Gateway result | `POST /api/checkout/payments/crm` | `/Students/SetECommercePayment` (queued, retried on 408/429/5xx/timeout) | Reports success or failure; browser gets `{status:"Queued", redirect}` |

Legacy `POST /leads/{staging|production}` -> `/Students/AddNewLead` is the website lead path (`mode:'lead'` in `window.LLA_ECOMM_CONFIG`); `mode:'ecomm'` is the full 3-route flow.

Source: `worker/README.md` and `worker/CLOUDFLARE-INTEGRATION-SPEC.md` in https://github.com/gitteromri-ux/lla-course-checkout

### 2.2 AddNewECommerceLeadAndOrder payload (from `dist/assets/eteacher-ecomm.js`)

| Field | Value | Note |
|---|---|---|
| `ProductID` | `26` | Longevity product, on every lead |
| `MainAbroadCourseId` | `258` | The Longevity Blueprint |
| `AbroadCourseId` | `1774` | class-level course id (eTeacher dev, 2026-08-05); semester 2273 "2026-10 - Longevity" |
| `LanguageId` | `101` | English |
| `CountryIsoCode` / `CountryIsoCodeByIp` | always `US` | PRICING CURRENCY GUARD (2026-09-09): CRM prices by country; an IL lead was minted an ILS intent (`int_nlpd6cws2hm4v98p9p1`). Product is USD only, so the order is always created as US; real country goes to AdminNotes and the E.164 phone. |
| `FirstName`, `LastName`, `Email`, `MobilePhone` (E.164), `State`, `StateProvinceRegion` | from form | US state auto-derived from NANPA area code |
| `LandingPage`, `UserAgent`, `ReferringSite`, `QueryString` | from browser | campaign is read from QueryString only as `cid=<id>` |
| `DynamicParameters` | click ids + utm_* + `google_campaign_id=116641` | 30-day persistence of gclid/fbclid/msclkid/ttclid/li_fat_id/wbraid/gbraid/utm_* |
| `AdminNotes` | `Buyer country: XX | US State: XX | SELF-SERVICE ECOMM CHECKOUT: The Longevity Blueprint · Start: … · SMS consent: … · Promo: …` | max 4000 chars |
| `CampaignID` | not sent | does not exist on this endpoint (ignored); email LPs use 117600 |
| `FirstPayment`, `NumberOfPayments`, `LeftToPay` | see 2.4 | |

### 2.3 GetECommerceFirstPaymentDetails and SetECommercePayment

- Details response gives `paymentId`, `amountCharged`, `currency`, `airwallex.dropIn {intent_id, client_secret, currency, mode:'recurring', recurringOptions:{next_triggered_by:'merchant'}}`, `airwallex.config.env` (`prod` or `demo`) and PayPal SDK URL. The CRM, not the site, mints the Airwallex intent; the intent carries `card_input_via: ecommerce` (never replace this value).
- The checkout validates the details against the expected quote (course ids 258/1774, USD, amount and installment count must match one of the allowed pairs) before mounting any payment form (`dist/checkout.html` "CURRENCY GUARD, HARD BLOCK MODE").
- `SetECommercePayment` report payload (required): `StudentId, OrderId, PaymentId, PaymentMethod ('Card' | 'PayPal'), AmountCharged, IsApproved`. Optional: `AuthCode, DeclineReason, ErrorCode, ErrorMessage, GatewayResponse, TransactionId, Last4Digits, CreditCardNumberLength, BIN, CCToken, ExpirationDateMMYY, CCHolderName, PayPalPayerId, PayPalPaymentId, PayPalOrderId, PreferredCourseId, RequestId`. Raw card fields (`Pan, CardNumber, Cvv, Cvc, SecurityCode`) are rejected by the Worker.
- PayPal path: PayPal Smart Buttons (same client id as eTeacher's other schools, currency USD, `disable-funding=credit,card`), capture must be `COMPLETED` in USD, then `SetECommercePayment` with `PaymentMethod 'PayPal'`.

### 2.4 FirstPayment / NumberOfPayments semantics

- `FirstPayment` = amount charged today.
- `NumberOfPayments` sent to the CRM = REMAINING installments after the first payment (`plan === 'upfront' ? 1 : numberOfPayments - 1`), so a 5-payment plan is sent as `NumberOfPayments: 4`. The CRM echoes back `NumOfPayments` and `AmountToCharge`.
- `LeftToPay` = remaining total (e.g. 4 x 169 = 676).
- Known CRM behaviour (verified live, order 8026893): the CRM can keep the catalogue price (FirstPayment 179 x 5) even when the buyer chose upfront; the site therefore re-validates the quote and refuses mismatches.

Allowed quote pairs hard-coded in `eteacher-ecomm.js` (`firstPayment / remaining / leftToPay`): 79/4/676, 169/4/676, 179/4/716, 799/0/0, 799/1/0, 149/4/596.

### 2.5 Current offers and promo codes

| Offer | Today | Then | Total | Notes |
|---|---|---|---|---|
| Standard (promo / masterclass price) | $79 | 4 x $169 | $755 | `LLA_PROMO` active, plan monthly, remaining 4, USD |
| Standard list monthly | $169 | 4 x $169 | $845 | config `monthly: {firstPayment:169, numberOfPayments:5, leftToPay:676}` |
| Upfront | $799 | none | $799 | `upfront: {firstPayment:799, numberOfPayments:1}`; Airwallex intent example `int_sgpvzggzxhm4mqvafja` |
| VIP (masterclass attendees) | code-driven | | | Julie VIP Masterclass included, "$99 value, yours with code JULIE249" |

Promo codes: `JULIE249` (Julie price contract) and `MASTERCLASS-VIP` (credit code, same JULIE249 price contract, no deadline). Email links may pre-apply with `?code=<valid code>` (commit `0a04ccc`). Older plans in the HANDOVER.md gates ($279 x 5, 18 weeks, $200 credit) are historical and superseded; do not reintroduce them.

Commits: `5897dcd` (MASTERCLASS-VIP accepted; synced to Pages deployment 850b444b), `0a04ccc` (?code= pre-apply; Pages deployment d2261859), `6de1cbc` (checkout audit copy), `ac35c28` (attendee offer parity on pages.dev).

### 2.6 Environments

| | Staging | Production |
|---|---|---|
| Trigger | `?env=staging` on the page -> `LLA_ECOMM_CONFIG.env='staging'` | default |
| Worker prefix | `/staging/api/...` | `/api/...` |
| CRM | eTeacher staging CRM | production CRM |
| Airwallex | DEMO account ("eTeacher Demo PA"), test cards valid, `env:'demo'` | prod ("ETEACHER EU LIMITED"), `env:'prod'` |
| Tracking | `LLA_LIVE=false`, no Meta events, relay no-op | live |

Worker secrets live only at Cloudflare level (`wrangler secret put`): `CRM_AUTH_SECRET, CRM_STAGING_AUTH_SECRET, CRM_CF_ACCESS_CLIENT_ID, CRM_CF_ACCESS_CLIENT_SECRET, CRM_STAGING_CF_ACCESS_CLIENT_ID, CRM_STAGING_CF_ACCESS_CLIENT_SECRET, PAYPAL_SERVER_SECRET, AIRWALLEX_API_KEY`. Non-secret vars: `CRM_BASE_URL, PAYPAL_CLIENT_ID, AIRWALLEX_SDK_URL, PAYMENTS_MODE, ALLOWED_ORIGINS`. The CRM is called with an `Authorization: token <secret>` header plus Cloudflare Access headers, added server-side in the Worker; the browser never sees them. Never commit `.dev.vars`, `.env` or a `wrangler.toml` with real values.

---

## 3. Airwallex Drop-In configuration (`dist/checkout.html`)

- SDK: `https://static.airwallex.com/components/sdk/v1/index.js`, `enabledElements:['payments']`, `locale:'en'`, `env` = `demo` (staging) or `prod`.
- Drop-In: `intent_id` + `client_secret` from CRM, `currency:'USD'` force-set, `mode:'recurring'`, `recurringOptions:{ next_triggered_by:'merchant', currency:'USD' }` for installment plans (the merchant triggers later charges). For the one-payment upfront plan use `next_triggered_by:'payment'` (customer-initiated, no mandate).
- Methods: `methods`, `payment_methods` and `paymentMethods` all set to `['card','paypal']`.
- USD guard: the form is never mounted if the intent currency is not USD or the amount/installments differ from the expected quote; the buyer is told to contact an advisor.
- Apple Pay / Google Pay history: Google Pay was rendered by the Drop-In (googlepay method) and an `apple-developer-merchantid-domain-association` file was added for domain registration under `/.well-known/` (lla-com commit `4030535`, later reverted). Both wallets were removed from checkout methods: lla-course-checkout `280e385` "Remove Google Pay and Apple Pay from checkout methods only" and Longevity-Academy.github.io `6b698ef` (2026-10-13) "remove Apple Pay and Google Pay from Airwallex Drop-In (card + PayPal only). Main site /checkout/ and /blueprint/checkout.html only; Julie masterclass untouched." Re-enabling requires Apple domain verification per host and Omri's written approval.
- After `onSuccess`: Meta Purchase fires, then the CRM report is sent through the Worker, then the confirmation state shows `Order #<orderid> · Payment of $<amount> USD received`.

---

## 4. Tracking and Golden CAPI

| Item | Value |
|---|---|
| Meta pixel / dataset | `1440305917310328` (init with advanced matching em, ph, fn, ln, st, country `us`, `external_id` = `lla_xid_v1` from localStorage) |
| Server relay | `https://lla-ac-events.vercel.app/api/meta/capi` (Vercel project `ac-events`); test event code historically `TEST75180` |
| GTM | container `GTM-PMRKXXJV` (has a broken Custom HTML tag that injects `<--->`; the page has a client-side cleaner as a bandage) |
| GA4 vs Meta | GA4/GTM dataLayer events (`lla_identity_ready`, CTA events) are analytics only. Meta pixel + CAPI are the conversion source of truth; never count GA4 and Meta as the same thing. |

Event scheme (browser pixel and CAPI share the same `eventID` so Meta dedupes):

| Event | When | eventID |
|---|---|---|
| PageView | page load | generated per view |
| ViewContent | product fold viewed | generated |
| AddToCart | enroll step 1 submitted (lead+order created) | generated |
| InitiateCheckout | checkout.html loaded with valid details | generated |
| Purchase | ONLY after Airwallex `onSuccess` or PayPal capture `COMPLETED` | `eventID` = CRM order id (`orderid`), with `value`, `currency:'USD'`, `content_ids:[LLA_CONTENT_ID]`, `content_type:'product'`, `order_id`, `payment_id` |

No pre-purchase Meta event fires on checkout mount any more (removed to stop inflated counts). Counter custom events were removed from the pixel (commit `8b313e9`, 2026-09-25, live CF deployment `543adb13`, rollback `a5414f7c`).

fbc / fbp preservation: `dist/_worker.js` sets 90-day `_fbc`/`_fbp` cookies server-side (`8b313e9`); checkout rebuilds `fbc` as `fb.1.<first-seen ms>.<fbclid>` when missing; `__llaRefreshClickPayload` refreshes click ids on relayed payloads except AddToCart/InitiateCheckout/Purchase (`dist/assets/lla-relay-receipts.js`). Without fbc, Ads Manager shows 0 purchases.

Tester suppression: `?llatest=1` or `?qa=1` or a tester email (example.com, test.com, mailinator, eteacher*, gita-agency, anything matching `gitter`, `test|qa|demo` prefixes) sets `lla_tester` and silences fbq + relay on every page (sticky); `?llatest=0` clears. `?env=staging` also disables tracking.

Julie Golden files (Longevity-Academy/julie-masterclass, frozen at tag `tracking-golden-20260928`): `julie-meta-tracking.js`, `julie-attribution.js`, `julie-payment-safety.js`. Freeze rule: these three files and the checkout/payment logic on that site are not edited by anyone; the last three commits touched copy only and state "preserve all checkout and Golden tracking scripts". Any change requires Omri's literal authorization sentence and a before/after diff against the tag.

---

## 5. DO-NOT-TOUCH rules

1. No change to any Golden / tracking / payment file without Omri's literal authorization sentence in the chat. Protected: `dist/checkout.html`, `dist/assets/eteacher-ecomm.js`, `dist/_worker.js`, `dist/assets/lla-relay-receipts.js`, `dist/assets/julie-*.js` (lla-course-checkout); `checkout/index.html`, `blueprint/checkout.html`, `assets/eteacher-ecomm.js`, `assets/ai-attribution.js` (Longevity-Academy.github.io); `julie-meta-tracking.js`, `julie-attribution.js`, `julie-payment-safety.js` (julie-masterclass, tag `tracking-golden-20260928`); the Vercel relay `ac-events`; the Worker `eteacher-leads-proxy`; GTM-PMRKXXJV tags; Meta dataset 1440305917310328 settings.
2. No CRM object creation without approval: no test leads/orders in production, no fake emails. Only consented test contact 7179031 (gitter.omri@gmail.com) may be used in ActiveCampaign.
3. No write on page visit: nothing is sent to the CRM until the buyer submits the enroll form. Demo state (no ids) writes nothing.
4. Never replace `card_input_via` (must stay `ecommerce`, minted by the CRM).
5. Never change currency: USD only, `CountryIsoCode:'US'` always, guard stays in hard-block mode.
6. Never create duplicate orders: reuse `stid`/`orderid` from storage; only rebuild an order when the quote changed; never re-fire Purchase (duplicate Meta events are "past trauma").
7. Purchase only after gateway success; `eventID` = CRM order id.
8. ActiveCampaign: fields 214-220 protected; never delete or create AC objects without Sandra's sign-off; sender `Longevity Life Academy <admissions@longevitylifeacademy.com>`.
9. Never echo the Meta CAPI token, Cloudflare token, CRM auth or PayPal secret into chat, docs or commits.
10. Read approvals never authorize uploads: permission to read/audit a repo, Worker, Vercel project or Meta dataset is not permission to deploy, push, or change settings. Each upload needs its own explicit go.
11. Content gates: no em/en dashes, emojis or exclamation points on LLA pages; never claim live Q&A with Julie; only scarcity line allowed: "When a session fills, it closes."; session dates never hard-coded (token-driven); Harel Tayeb never under a Julie headline.
12. Funnel changes deploy to BOTH pages.dev and workers.dev; always cache-bust links.

---

## 6. Known open issues and incidents (facts, dated)

| Date | Incident / issue | Status |
|---|---|---|
| 2026-08-04/05 | Missing `MainAbroadCourseId` caused server NRE on AddNewECommerceLeadAndOrder; fixed by always sending 258/1774/101. | Closed |
| 2026-09-09 | IL buyer was minted an ILS Airwallex intent (`int_nlpd6cws2hm4v98p9p1`). Fix: order always created as US; hard USD guard before mount. | Closed, guard live |
| 2026-09-25 | Attribution fixes: server-set 90d `_fbc/_fbp`, sticky test-device exclusion, counter events removed. CF deployment `543adb13`; rollback point `a5414f7c`. Commit `8b313e9`. | Live |
| 2026-10-13 | Apple Pay and Google Pay removed from Drop-In on main site (`6b698ef`) and pages.dev (`280e385`). | Live |
| 2026-10-27 | Payment-record linkage incident on the attendee guest page: commits `4030535` (new guest page + checkout + hotfix state) and `025f925` ("Harden guest form receipts and correct verified remaining-installment contract") were pushed to Longevity-Academy.github.io and then both REVERTED the same day (`34a81a9` at 10:16 UTC, `b4435d5`) after audit. Attendee offer parity was then shipped on pages.dev instead (`ac35c28`, 19:53 UTC). Rollback audit: compare https://github.com/Longevity-Academy/Longevity-Academy.github.io/compare/6b698ef...b4435d5 (should be net zero on protected files). | Reverted; confirm main-site checkout files equal pre-`4030535` state before any new work |
| Open | Installment recurring-consent concern: the Drop-In is mounted with `mode:'recurring'`, `next_triggered_by:'merchant'`, i.e. the buyer consents to merchant-initiated charges. It is NOT verified in code whether the eTeacher CRM auto-charges the remaining 4 installments against the saved Airwallex consent or sends payment links instead. Treat as UNVERIFIED; ask eTeacher (Ehud) for the written contract before changing copy such as "4 x $169 charged monthly". | Open |
| Open | Whether CRM honours `NumberOfPayments` as REMAINING installments in every path (order 8026893 showed catalogue 179 x 5 despite upfront choice). Site-side validation blocks mismatches. | Open, mitigated |
| Open | GTM-PMRKXXJV broken Custom HTML tag injecting `<--->`; page cleaner is a bandage. | Open |
| Open | Meta domain verification meta tag for www.longevitylifeacademy.com `<head>`. | Open (verify current state) |
| Open | Vercel relay changes for LGV_Product_Event Regular/VIP handling awaited Omri's deploy permission (`5897dcd`). | Verify deployed |
| Open | ActiveCampaign automations: Sandra's sign-off required; fields 214-220 untouched. | Open |

---

## 7. Operating rules for working with Omri

1. Before any work: state a time budget (minutes) and a plan; no clarifying questions when the ask is clear.
2. During work: report numeric pace ("40% done, 6 min left"); no silent stalls.
3. 100% scope, never cut scope, never ask for extensions; if something is impossible, say so with evidence.
4. Audit before "done": verify live with curl/screenshots (desktop 1440 and mobile 412), check both hosts, check no protected file changed (`git diff <tag|sha> -- <protected files>` must be empty).
5. Handoffs: direct deep links (cache-busted) plus paste-ready text; never navigation instructions.
6. Bucket every claim as VERIFIED or UNVERIFIED.
7. No secrets in chat, docs or commits.

---

## 8. Quick-start commands (read-only first)

```bash
git clone https://github.com/gitteromri-ux/lla-course-checkout lla-dev
git clone https://github.com/Longevity-Academy/Longevity-Academy.github.io lla-com
git clone https://github.com/Longevity-Academy/julie-masterclass julie
# Golden freeze check
git -C julie diff tracking-golden-20260928 --stat -- assets
# Live parity check
curl -s "https://longevitylifeacademy.pages.dev/?v=$(date +%s)" | md5sum
curl -s "https://lla-checkout.gitter-omri.workers.dev/?v=$(date +%s)" | md5sum
# Staging checkout (demo Airwallex, no live charge, no tracking)
open "https://longevitylifeacademy.pages.dev/checkout.html?env=staging&llatest=1"
```

Key commits: https://github.com/gitteromri-ux/lla-course-checkout/commit/8b313e9 (attribution), https://github.com/gitteromri-ux/lla-course-checkout/commit/5897dcd (MASTERCLASS-VIP), https://github.com/gitteromri-ux/lla-course-checkout/commit/280e385 (wallets removed), https://github.com/gitteromri-ux/lla-course-checkout/commit/ac35c28 (attendee offer on pages.dev), https://github.com/Longevity-Academy/Longevity-Academy.github.io/commit/6b698ef (wallets removed, main site), https://github.com/Longevity-Academy/Longevity-Academy.github.io/commit/025f925 and https://github.com/Longevity-Academy/Longevity-Academy.github.io/commit/b4435d5 (2026-10-27 incident and revert).

---

## 9. Addendum from Omri's session memory (facts the code alone does not show)

| Date | Fact | Source |
|---|---|---|
| 2026-09-17 | pages.dev checkout went live with Airwallex Drop-In. Omri completed a real-card full purchase at $279.80 that day. That order was created in one-time `mode:'payment'`, so no card consent exists for it; the CRM sends a payment link for each later installment. Treat it as manual. | memory projects.web_checkout.payment_integration; session 49da238d |
| 2026-09-22 19:51 UTC | Recurring consent for installments went live: .dev commits 39a043b and 445e6a9 (https://github.com/gitteromri-ux/lla-course-checkout/compare/f17e805...445e6a9), .com commit 80dc8a3 (https://github.com/Longevity-Academy/Longevity-Academy.github.io/compare/1f789a3...ffd4a3f). Any order paid between 2026-09-17 and that moment shares the one-time exposure. Live-verified on all three checkouts on 2026-11-02. | this handover |
| 2026-09-23 | Airwallex intents are minted by eTeacher CRM via GetECommerceFirstPaymentDetails, never by the Worker. Reference IIBS intent returned `payment_method_options {card:{card_input_via:"ecommerce"}}`. Never replace it with applepay. | memory projects.web_checkout.airwallex_intent_ownership |
| 2026-09-22 | Vercel relay production deployment dpl_ET6KZD4gwmBJCWs8gWCRU86Qa4xz for project prj_3v7g4Xwve6OmXrcPCVQOWJOiN2SK verified READY; checkout copies post to /api/meta/capi with dataset 1440305917310328. | memory projects.web_checkout.lla_event_infrastructure |
| 2026-09-23 | GTM-PMRKXXJV v22 used a case-sensitive Click Text contains "Enroll Now" rule for GA4 Clicked_on_enroll_now and Meta Clicked_on_Enroll; the .dev site fires Enroll_CTA_Click separately. Button-label coverage is not event receipt; GA4 is not Meta. | memory projects.marketing.lla_tracking_audits |
| 2026-09-28 to 2026-10-26 | Julie Golden freeze: release julie-meta-20260927-4, tag tracking-golden-20260928. Golden = one attributed real-customer transaction chain (campaign -> page -> eTeacher order -> payment success -> Meta Purchase), not certification of every later release. On 2026-10-15 commit 5e176ea changed julie-meta-tracking.js without the authorization sentence; the 2026-10-17 rollback restored the 2026-10-15 production build. On 2026-10-29 commits ce0caf8 and bf3b14d removed October 27 copy only; five protected tracking/payment assets stayed byte-identical and the relay alias still pointed to Golden deployment dpl_DQogoR... | knowledge projects/lla-julie-tracking; memories julie_masterclass_audit_20261029 |
| 2026-10-27 | eTeacher NumberOfPayments means REMAINING installments: FirstPayment 149 + NumberOfPayments 4 + LeftToPay 596 = 149 now plus four payments of 149 = 745. Final production verification: MASTERCLASS-VIP created order 8063265 for student 11683843, class 170277; fbc/fbp preserved. | memories lla_guest_page_audit_20261027, lla_guest_final_verification_20261027 |
| Standing | Approved pages.dev offer: $169 x 5 = $845 base; JULIE249 and MASTERCLASS-VIP select $79 + 4 x $169 = $755; upfront $799. Blueprint relaunch VIP selected variant: $129 + 4 x $229 = $1,045 (publication gated on validated checkout). Staging CRM order mapping: ProductID 26, MainAbroadCourseId 258, AbroadCourseId 1774, PreferredCourseId -1, LanguageId 101, IsTrial 0, cid=116641 in QueryString, CampaignID omitted. | knowledge projects/lla-course-checkout, projects/lla-blueprint-relaunch |
| Standing | Blueprint payment-record incident: multiple separately identified payment records inside each customer's CRM record (one per installment), not duplicate customer profiles. Unresolved; inspect per-customer payment identifiers before concluding. | knowledge projects/lla-blueprint-payment-record-incident |
| Standing | Evidence rules: returned IDs, byte-identical files, API acceptance or events_received are never proof of a paid checkout, attributed conversion or a verified Golden chain. Say VERIFIED or UNVERIFIED explicitly. | memory preferences.evidence.payment_enrollment, clickid_evidence_guardrails |
| Standing | Omri will not accept repeated Airwallex credential popups; reuse existing authorized access and uploaded payment-team material. Every Enroll CTA opens the pop-up checkout only, never a lead/application form. Reference ecommerce flow is IIBS (Biblical Hebrew -> Level 1 -> Get started right now), not Rosen. | memories preferences.agent.airwallex_access_reuse, projects.web_checkout.lla_course_flow |

---

## 10. GOLDEN RECURRING rule (frozen 2 Nov 2026, rank equal to Golden CAPI)

**Rule:** On every LLA Blueprint checkout, the Airwallex Drop-In for any plan with more than one payment MUST receive `customer_id` (from the CRM response `Airwallex.customerId`), `mode: 'recurring'`, `recurringOptions` and `payment_consent` with `next_triggered_by: 'merchant'` and `merchant_trigger_reason: 'unscheduled'`, exactly as eTeacher's `GetECommerceFirstPaymentDetails.AirwallexOptions` prescribes. Pay-in-full stays `mode: 'payment'`. No agent changes, removes, reorders or redeploys these lines without Omri's literal written authorization in that conversation. Any release touching a checkout file must prove these hashes intact (or intentionally changed with his sentence) and re-verify the live HTML. "Recurring works" is only claimable with an Airwallex readout showing `payment_consent_id` on the intent.

| Surface | Commit / tag | File | sha256 |
|---|---|---|---|
| www.longevitylifeacademy.com/checkout/ | Longevity-Academy.github.io `492bb0ee445d6d977d82a64d5e7ddbaac5081770`, tag `golden-recurring-20261102` | checkout/index.html | f29ca1a96445d8d9f1d06f3a94b157dfb62e466caf50b2df4026e7643dcc9040 |
| www.longevitylifeacademy.com/blueprint/checkout.html | same | blueprint/checkout.html | ad45102f8ac4cf21d17e38aa315bdaba8ea4d96866143fa6542584fbb94c4363 |
| longevitylifeacademy.pages.dev/checkout and /blueprint/checkout.html | lla-course-checkout `08cd3251c11a4e35291c91d59778231ebdf5d826`, tag `golden-recurring-20261102`, Cloudflare Pages deployment 916f29e4 | dist/checkout.html, dist/blueprint/checkout.html | 9a77ceb89f8ca0fc7d2132dfb34d834a11ae76f0d3d86e85450f17850b853fba |

Live hashes matched these values at 2026-11-02 20:20 UTC on all four URLs.

**Evidence:** staging CRM order 8068308 (student 11687346), intent `int_sgpvb7qw4hnsmm4fgv1`, customer `cus_sgpvb7qw4hnsmm3kwjv`. Before fix: `customer_payment_consents: []`. After confirming with the fixed payload: `payment_consent_id: cst_sgpvb7qw4hnsn6coqxb`, `payment_consent.next_triggered_by: merchant`, method `mtd_sgpvb7qw4hnsn6cooly` attached to the customer. Diffs: https://github.com/Longevity-Academy/Longevity-Academy.github.io/commit/492bb0ee445d6d977d82a64d5e7ddbaac5081770 and https://github.com/gitteromri-ux/lla-course-checkout/commit/08cd3251c11a4e35291c91d59778231ebdf5d826

**Recorded fact:** installment orders paid before 2 Nov 2026 20:00 UTC have no consent on file; remaining installments are link-billed until eTeacher attaches consents.

**Check command:**
```bash
for u in https://www.longevitylifeacademy.com/checkout/ https://www.longevitylifeacademy.com/blueprint/checkout.html https://longevitylifeacademy.pages.dev/checkout https://longevitylifeacademy.pages.dev/blueprint/checkout.html; do echo "$(curl -sL "$u?x=$RANDOM" | sha256sum | cut -c1-64) $u"; done
```
