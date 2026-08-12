# Build log

Newest first. What shipped, real numbers, what broke.

## 2026-08-12 — Real market data in, checkout wired, nobody has paid yet

**What shipped**
- FlippaDrive's underwriting engine now runs on live market data from a licensed data provider. The mock provider is demoted to an explicit fallback behind a provider module: VIN decode + active-listing comps, a 7-day valuation cache keyed per VIN, a monthly budget gate that degrades to demo mode instead of overspending, and a per-endpoint cost ledger where every call — including cache hits at $0 and failed attempts — writes a row.
- The full Stripe stack, live in a dedicated FlippaDrive account: 4 products, 4 prices (every one with a `lookup_key` and `tax_behavior: exclusive`, per the recipe extracted from SudsOps), 2 subscription payment links, real price IDs wired into checkout with a server-side allowlist, secret key rotated to the new account, customer portal configured. Enterprise came off self-serve entirely — it now says "Contact for pricing" because that tier should be a conversation.
- Trust pages that match the trust pitch: About rewritten to say plainly it's a solo founder with a risk-management background, an essential-cookies-only cookie policy, and a consent banner with two equal buttons — Essential Only and Enable Analytics. Analytics is gated on that flag in the tracking code itself: no consent, no identifiers, no localStorage writes.
- Vendor selection: two market-data vendors compared on published pricing and terms. One quoted a fixed monthly subscription for a workload the other prices metered at a small fraction of that; production runs on the metered vendor's entry tier, a free trial of the other is in flight for an accuracy bake-off, and caching terms are in writing — persisted report data may be re-shown to the customer who bought it. Names and rates stay out of the public log.

**Numbers (real ones only)**
- Market data cost per fresh underwrite: under half a cent; a cache hit costs $0.000
- The placeholder ledger rates were 14x too high until checked against the vendor's published fee schedule
- 42 active comps on the first working live underwrite (2020 Ford Fusion S, exact-trim match, no widening needed)
- 4 Stripe products, 4 prices, 2 payment links, 0 test purchases so far
- 15.7 Lovable credits across 8 agent runs, 240 founder-minutes, 3 work blocks over 6 days
- $0 revenue — the checkout is live and no one has been through it

**What broke**
- The first production underwrite came back wearing the demo banner. The comps search queried active listings by exact VIN — which matches at most the one physical car being underwritten — so the 3-comp minimum tripped and every real VIN on earth would have fallen back to sample data. Fix: comps come from the decoded year/make/model/trim, widening to ±1 year when a trim is thin, with confidence downgraded whenever widening was needed.
- Same failure exposed a ledger hole: when the provider threw, the billable HTTP calls it had already made were discarded with the error. Real API calls happened, zero rows written, spend showed $0.00. Failed attempts now log their calls and cost before the fallback runs.
- The first draft of this entry named both data vendors and their per-call rates. William caught it after it was committed. The entry was rewritten within the hour and the content skill now bans vendor names and vendor pricing from every public surface.
- The SKU catalog workbook could not be written in place — Excel's lock rides through the OneDrive mount — so the corrected file lives under an `-UPDATED` name waiting for a manual rename. Third session this workbook has fought back.
- The Linear connector invalidated itself in the middle of session close-out, and an agent build timed out at the 180-second MCP limit while the build itself finished fine — both look like failures, neither was, both cost a verification round-trip.
- The COO review verdict on the session: Revise. Correct call — a billing stack with zero completed purchases is wiring, and it stays "wiring" until one real payment clears end to end.

## 2026-08-08 — Outlook was reading my auth emails and spending the tokens

**What shipped**
- Found why sign-in was dead on FlippaDrive: Supabase issues one one-time token per auth email, and `{{ .Token }}` (the six digits) and `{{ .ConfirmationURL }}` (the link) are the same token. Outlook Safe Links fetches the URL the moment mail lands, which spends the token before a human touches it. The logs caught Microsoft hitting `/verify` 18 seconds after send, then my own click returning `otp_expired`.
- Fix that holds: emails now carry `{{ .TokenHash }}` in a link to a page I own, `/auth/confirm`, which calls `verifyOtp` only when someone presses a button. A scanner loading that page spends nothing. Verified against the same Outlook mailbox that had failed twice — one POST `/verify` from my own IP, no Microsoft address anywhere near it.
- Magic-link sign-in added next to password sign-in, on the same click-gated pattern.
- Duplicate signups now say so. Supabase returns HTTP 200 with an empty `identities` array when the address already exists and creates nothing, so the app was showing people a success screen for an account that was never made.
- Killed a `getUser()` call firing on every anonymous page load. It threw a 403 each time, logged the event anyway, and dragged the project's success rate to 72%. The anon key is a JWT with no `sub` claim, so the guard is a `sub` check before the call. Same bug found in a second edge function.
- RLS pass across the Supabase estate: audited eight projects, rewrote 31 policies to wrap row-independent functions in `select` so Postgres caches them per statement instead of per row, moved role tests into `TO` clauses, and indexed two policy filter columns.
- Auth email moved onto Resend SMTP on my own verified sending domain, with branded templates.

**Numbers (real ones only)**
- 8 Supabase projects audited, 4 needed work, 3 were already clean
- 31 RLS policies rewritten, 2 indexes added on policy filter columns
- Supabase's published benchmark for the wrap alone: 179ms → 9ms; unindexed to indexed, 171ms → under 0.1ms
- 18 seconds from send to Microsoft spending the token; 26 seconds on the second attempt
- 2 Lovable commits, 7.5 credits
- 240 active minutes
- One empty database found: zero users, zero rows, last migration May 2025, sitting there under the good name while the live one wore a "2" suffix
- $0 revenue — FlippaDrive still isn't selling

**What broke**
- I kept the link in the email as a "fallback" for anyone who couldn't use the code. The link and the code were the same token, so the fallback was the thing killing the primary path. Told William to keep it, watched it break his reset, removed it.
- Before the logs came back I floated a theory that the custom domain was serving a stale bundle. The request payload said otherwise — the client was pointed at the right database the whole time. Guessing ahead of the evidence cost a round trip.
- A `referer: http://localhost:3000` in the auth log sent me hunting for a dev origin baked into production. It was the project's Site URL leaking into a log field. Real problem, wrong diagnosis.
- Two migration writes came back "No approval received" and did nothing, which looks identical to a silent failure until you check that the database is untouched.
- The COO review at close-out had no subagent tool available on this surface, so it is self-assessment wearing a reviewer's hat. Verdict was Revise: the signup path still has not been exercised end to end by a second address.

## 2026-08-06 — Domains, de-slop, and a favicon that would not die

**What shipped**
- blog.flippadrive.com and www.flippadrive.com live on custom domains (Cloudflare DNS, Lovable hosting). The blog no longer shows a lovable.app URL anywhere.
- Killed an entire hallucinated blog living on the main site's production build — 12 AI-seeded posts with invented case studies and profit claims. `/blog` now redirects to the real blog.
- Ten-item marketing cleanup on flippadrive.com: fake phone/address/support hours off the contact page, fabricated partner testimonials deleted, fake "73/100 spots filled" scarcity deleted, fake user counts out of the share bar, careers page for a one-person company removed, feature pages consolidated around the one thing that matters (VIN → underwrite → verdict). Footer now links NHTSA VIN Decoder and NMVTIS — things buyers actually use.
- Brand favicon + OG image wired on both sites, with per-post og:image on the blog.
- Stripe: new FlippaDrive account onboarding underway; extracted the SudsOps product/price/payment-link pattern from the live API into a reusable setup recipe + a five-venture SKU catalog workbook; added the missing `lookup_key` to the live SudsOps price.

**Numbers (real ones only)**
- 12 fabricated blog posts removed from production
- 10 cleanup items executed in one agent pass
- 12.5+ Lovable credits burned across seven agent runs (one run's cost went unreported)
- 1 live Stripe API write (`lookup_key` on the SudsOps price)
- $0 revenue — FlippaDrive isn't selling yet

**What broke**
- The favicon shipped wrong twice. The brand doc claimed an icon asset existed; it didn't. The blog agent then downloaded the main site's *old* published favicon (the Lovable heart) believing it was our mark, and generated the full icon set from it. Verification checked HTTP 200s instead of pixel content, so it got called fixed while production served a pink heart at every size. William caught it. Canvas pixel-sampling settled the argument, and Lovable's CDN then kept serving stale files at the old paths, forcing a `-v2` filename rename to bust the cache.
- One agent instruction silently died on a dropped connection and had to be re-sent after the message log showed it never arrived.
- An agent copied the homepage's known bad "INVESTIGATE" badge into a new component — verdict vocabulary is locked to buy/negotiate/pass — caught and fixed same session.
