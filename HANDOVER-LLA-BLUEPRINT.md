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
