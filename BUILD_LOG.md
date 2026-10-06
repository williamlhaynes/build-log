# Build log

Newest first. What shipped, real numbers, what broke.

## 2026-10-06 — A campaign inbox that drafts every reply and sends none without my tap

**What I built**
My Atlanta Pride privacy campaign (Oct 10-11, 2026) prints one contact address on its cards and pages. I want every message answered fast, and nothing going out that I didn't read. A message arrives, a language model sorts it into a lane and drafts a reply from an answer library I approved, and I get one alert with the draft and a review link. I edit or don't, tap Send, and the reply leaves from the campaign address, threaded under theirs. Nothing sends without that tap.

The desk is the first build under a rule I now hold my own agents to: every action with an outside effect gets a yes, a no, or a step up before it runs, and leaves a record I can show later. I wrote the rule on 2026-10-05 after three actions ran past my authority while the agent was signed in the whole time (details under What broke).

Status: built and tested; sending domain verified on 2026-10-06; deploy pending as of 2026-10-06. This entry gets its link once the desk answers live mail.

**How it works**
- Cloudflare Email Routing hands each message for the campaign address to a Worker. Mail from machines (auto-replies, bounces, mailing lists, the campaign's own domain) is dropped before anything is stored.
- The Worker stores the message in D1 and asks the model for a lane, a one-line summary and a draft. Four of the eight lanes (privacy requests, member help, press, personal or safety) are marked mine: the draft holds the line and I answer personally.
- The alert reaches my inbox through Resend with a signed review link that expires after 14 days.
- Send claims the message first, so a second tap cannot send twice, then the exact text in the box goes out through Resend with threading headers.
- A cron every 15 minutes retries failed drafts and alerts and deletes anything older than 90 days.

**Take it: seven checks for any agent wired to real accounts**
Each one is a yes or no.
1. Does every action with an outside effect wait for a person's yes? Ours: posts, deploys, DNS, money and new access wait. Drafts and reads don't.
2. Does an approval cover only what you saw? Ours: what goes out is exactly the text in the box at the tap. An edit means a new approval.
3. Can the agent give itself new access? Ours: no. New keys, repo access or tools need their own yes.
4. When a yes leaves a value open, does the agent ask about that value? Ours: yes, on that value only.
5. Does "unknown" ever count as yes? Ours: never.
6. Can you take a yes back before the action runs? Ours: yes, and the action does not run.
7. Does every decision leave a record you could hand someone? Ours: what it touched, the decision, the time.

Two patterns worth copying. One approval, one send:

```js
const claim = await env.DB.prepare(
  "UPDATE messages SET status='sending' WHERE id=? AND status IN ('drafted','held')"
).bind(id).run();
if (claim.meta.changes !== 1) return alreadyHandled();
```

Never draft a reply to a machine:

```js
const auto = get('auto-submitted');
if (auto && auto.toLowerCase() !== 'no') return 'skip';
if (/^(bulk|junk|list)$/i.test(get('precedence'))) return 'skip';
if (get('list-id') || get('list-unsubscribe')) return 'skip';
```

**What I checked before shipping**
- 79 of 79 local checks, including: replies leave only from the campaign address and never carry my company address; a tampered review link is refused; a second Send is refused; a failed send keeps the edited text and sends nothing; machine mail is skipped and not stored; inbound HTML is escaped; messages past 90 days are deleted; the health check names keys without printing them.
- The review page runs no JavaScript and sends `Content-Security-Policy: default-src 'none'`, `Referrer-Policy: no-referrer`, `noindex` and `no-store`. Checked at phone width in light and dark.
- Open and click tracking off on the sending domain. A send-only key limited to that one domain. DMARC at quarantine.
- Not yet checked: live mail end to end. That runs after the deploy.

**Numbers (real ones only)**
- 1 Worker (157 KB bundle), 1 database with 2 tables, 8 lanes, 11 facts in the answer library
- 5 DNS records added for sending, plus Email Routing's own
- 79 of 79 local checks
- Caps: 5 drafts per sender per day, 200 per day
- 0 messages received, 0 replies sent, $0 revenue. Not deployed yet

**What broke**
- On 2026-10-05, with the agent signed in throughout: a build log entry went public before I confirmed it, push access to the repo was attached without asking, and a tax setting got picked that my yes didn't cover. That is where the rule above came from.
- The first render put the reply box and buttons in fallback fonts. `font: 16px/1.5 inherit` is invalid CSS because `inherit` can't sit inside a shorthand, so the browser dropped the whole line. The screenshot caught it; no test did.
- A test that scans the answer library for a banned word failed on the rule that names the word. The rule text appears twice and `replace` only strips the first. `replaceAll` fixed it.
- Screenshots of the DNS page timed out in the browser, so the records went in through the dashboard's own API from the signed-in page.
- The remote shell on my machine turned out to be Linux with my Downloads folder mounted, so the first unpack command (written for PowerShell) failed.

**Next**
Deploy, route the campaign address to the desk, send a test from a phone, then run it through Pride weekend.

Look-back check, 2026-10-13: the next entry reports how many messages the desk received over Oct 10-11, how many replies went out, and how many went out without a tap (the target is zero).

## 2026-10-05 — Build In-House wish list: the asks are the lead gen

**What I built**
Small organizations carry the same proof obligations as large ones (a CMMC Level 2 assessment, a client security questionnaire, staff who charge their phones in airports) with less budget to meet them. The Build In-House wish list lets an organization post what it needs proven, anonymously by default, and lets a sponsor that benefits when the need is met fund the line. Build In-House does the work, the recipient keeps the findings, and the sponsor gets confirmation that it happened. The asks are the lead gen: each one is a stated need from an organization that chose to show up. Status: built and tested, deploy pending. This entry gets its link once the live page is verified.

**How it works**
- One Cloudflare Worker renders four audience pages (everyone, CISOs, IT directors, compliance leads) from one template, each with its own headline, line order, and default ask.
- The lines live in a D1 table, so adding, retiring, or reopening a line is a row change, not a redeploy.
- "Fund this" redirects to a Stripe Payment Link carrying a `client_reference_id` that names the line. Checkout returns to `/thanks?session_id={CHECKOUT_SESSION_ID}`, which puts the remaining open lines in front of the sponsor again, with the one they just funded moved last.
- Funded counts are recounted from Stripe's completed sessions, so day one needs no webhook and no stored secret.
- Scoped work (the pre-assessment evidence review) takes a pledge instead of a payment, because it has to be scoped with the recipient before anyone pays.

**Take it: six checks for an intake form regulated buyers will use**
Each one is a yes or no. Run them against your own form.
1. Does the form ask only for what you need to reply? Ours takes the need, organization type, size band, timing, and one email.
2. Does it tell people what not to send? Ours says no CUI, no contract numbers, no system names.
3. Is anonymous the default, and is it enforced on the server? Ours drops the organization name server-side unless the person chose to show it.
4. Does the page load anything from a third party? Ours runs no JavaScript, serves its own fonts, and sends `Content-Security-Policy: default-src 'none'` with `Referrer-Policy: no-referrer`.
5. Can you stop spam without a CAPTCHA vendor? Ours uses a hidden field and a minimum fill time.
6. Does the price wait for the order step? Ours appears only on the Stripe checkout page.

Two patterns worth copying. The "show my name" toggle needs no script:

```css
form:has(#listing-anon:checked) .orgname { display: none; }
```

And the redirect that tags every checkout with the line it funds:

```js
const u = new URL(item.payment_link_url);
u.searchParams.set('client_reference_id', 'wishlist_' + item.id);
return Response.redirect(u.toString(), 302);
```

**What I checked before shipping**
- 63 of 63 local checks: lane routing, spam drops, anonymous listing, Fund redirects, Stripe session verification, idempotent recounts, and copy rules (no prices, no banned phrases) across every rendered page.
- Every page at desktop and phone width, in light and dark.
- Session IDs are pattern-checked before any call to Stripe, and a repeated visit to the thank-you page cannot count a funding twice.
- No inline styles anywhere, because the CSP would block them.
- Not yet checked: the live page. That runs after the deploy.

**Numbers (real ones only)**
- 5 wish-list lines; 4 audience pages plus thank-you, ask-received, and pledge pages
- 4 live Stripe products with payment links; 1 database with 4 tables
- 232 KB Worker bundle, three fonts included
- Sourcing pass for the kits: 252 catalog listings captured, 175 kept after relevance filtering, 57 scored, 50 ranked; 2 kit parts the catalog does not carry (a USB data blocker and a phone-size faraday pouch)
- 120 founder-minutes; 77 minutes agent wall clock
- 0 asks, 0 pledges, $0 revenue. Not deployed yet

**What broke**
- I built it for the wrong door. The word "sponsor" sent it to mobilesecurity.guru, the build-in-public surface where GitHub Sponsors lives. William asked whether it should be Build In-House. The portfolio canon in Linear says paid engagements belong on the consulting surface, and every line on this list is a paid engagement billed by Build In-House. It was rebranded and retargeted before anything went live. A two-minute canon check at the start would have skipped the detour.
- The "Fund this" buttons rendered 300 pixels wide and spilled out of their column. The button variant class was `line`, which was also the class on each requisition row, a four-column grid, so the button inherited the grid. The first screenshot caught it; no test covered it.
- The bundler embedded the fonts with `Uint8Array.fromBase64`, which the local Node runtime does not have, so the first test run crashed on import. Pinning the build target to ES2022 brought back the portable decoder.
- The catalog search is fuzzy enough to be wrong. "Signal blocking bag" returned color-blocking pants and swimsuits, "webcam cover" returned beach cover-ups, and "USB data blocker" returned a charging adapter.
- A GPS "interference blocker" showed up beside the privacy gear. It is a jammer, illegal to sell or use in the US, and it stayed off the list.
- `pkill` matched its own shell and killed the command (exit 144). It has done this before.
- The payment links went live before the page did, and the two kits cannot ship until their missing parts are sourced.

**Next**
Deploy, verify the live page, then decide whether the two kit links pause until their parts are sourced. COO review verdict: Revise.

## 2026-09-29 — Deleted the Deploy button, shipped what actually runs

**What shipped**
- `mobile-security-guru/hyperagent` v0.1.0. For five weeks the README promised a deployable agent and carried a Deploy to Cloudflare button that deployed an empty repo. The button is gone. What ships instead is method: three seat folders (CISO, IT Director, Compliance), each with its questions, the exports to pull, and a paste-ready prompt, plus an `AGENTS.md` holding privacy rules any agent reads first. The buyer's own agent runs it against their own files. The output is a table of every action that runs through a phone, who may take it, the record that proves it, and UNKNOWN where no record does. We never receive the data. No code runs in the repo, which makes the privacy claim easy to check: read it.
- The only thing the kit suggests sending back is a one-line summary of counts, in an email subject, and only if the user chooses to. An inbox rule labels those on arrival. That is the whole engagement signal; nothing else reports home.
- mobilesecurity.guru had been quietly bouncing visitors to the main site. It now redirects to the repo, and the repo's description, website, and topics match the kit.
- The Sponsors goal and introduction were rewritten to match. Sponsor events now arrive by webhook into n8n and email me, with a flag whenever one of the ten capped Founding seats is taken.
- Role addresses: billing@ forwards a copy to the inbox I actually watch. contact@, the public email on the GitHub org, did not exist; now it does. Test messages to both landed.
- Three 90-second video scripts, one per seat, pointing at the kit.
- Drafted, not deployed: typed Cloudflare configs for the SOW gate and the roaming site (self-serve W-9, per-target pages, a JSON offer endpoint written for buyers' agents).

**Numbers (real ones only)**
- 17 files in the v0.1.0 commit; 1 release; 3 seat folders
- 1 blind test by a separate agent on made-up data: sent nothing out, used no names, flagged 4 unclear instructions; all 4 fixed before release
- 22 local tests passing on the undeployed Worker configs
- 31 days the Sponsors application has sat in review; 1 open support ticket, no human reply yet
- 120 founder-minutes; roughly 13 hours wall clock across two days, idle time included
- 0 sponsors, 0 stars, 0 forks, $0 revenue

**What broke**
- The Deploy button deployed nothing for five weeks. Nobody reported it, most likely because nobody clicked it.
- The public contact address on the GitHub org had no mailbox behind it. Mail sent there had nowhere to land.
- The Sponsors support ticket went in on Aug 29 with two template placeholders still in it: "[DATE]" and "[EMAIL]". A second ticket was closed as a duplicate of the first. Both details went in as a comment today.
- Sponsor notices were routed to an address nothing watched. It turned out to have never received a message, so nothing was lost. That was luck, not design.
- Finding the .guru forward took four stops: two Cloudflare rule pages, then the registrar's forwarding panel. The cause was the website host redirecting secondary domains to the primary one. A redirect rule at the edge runs first and overrides it.
- Direct git push was refused by the session's proxy, so the kit went up through the API as one commit. The n8n activation call failed with a 415 and the n8n editor would not render in the automated browser, so William flipped the switch by hand.
- The first save of the Sponsors introduction did nothing: setting the field programmatically never registered with the editor. Typing it did.
- COO review verdict: Escalate. Two follow-ups are open, including sponsor tiers that still promise support for software that is not built. Those go next.

## 2026-09-06 — The alert pipe was never connected

**What shipped**
- sow.buildinhouse.com is a gated statement of work on a Cloudflare Worker: public summary at the root, an email-and-code gate on `/full`. Every gate event (page view, code sent, code failed, access granted, document served, access requested by a non-allowlisted address) is supposed to POST to an n8n workflow that emails William for the ones worth reading. As of this morning, none of them had ever arrived. The Worker's webhook URL was a commented-out line in `wrangler.toml`, so the logging function returned before sending anything. Fixed: the production webhook URL is now a bound variable, and the deploy output lists it.
- One shared-secret header on both sides. The Worker sent the secret under one header name, the n8n credential was named for a different one, and the credential itself was attached to an unrelated W-9 request webhook rather than the gate webhook. The n8n workflow also had an IF node checking for a Bearer token the Worker never sends, which would have silently dropped every call that did get through. Now: same header name on both ends, Header Auth on the webhook the Worker actually calls, the dead IF node removed, and n8n returns 403 before the workflow runs if the header is wrong. The secret value never appeared in a file, a log, a response, or this chat.
- One latent crash fixed on the way past: the internal endpoint that serves the W-9 to the approval workflow referenced a variable before it was declared, so a successful auth would have thrown instead of returning the PDF.
- Two questions answered by reading instead of asking: the SOW gate and the W-9 request are separate webhooks in separate workflows, not a rename; and the Worker targets the production `/webhook/` path, not `/webhook-test/`.
- Verified against production, not a test URL: after deploy, a single page view produced n8n execution 10423 with Header Auth passed; a document view produced two more. Execution count went from zero, ever, to three.

**Numbers (real ones only)**
- 3 files changed (Worker source, wrangler config, workflow JSON); 1 n8n workflow republished; 1 Worker deploy
- 0 n8n executions before the deploy, 3 after, all successful
- 4 form submissions during the session hit the old Worker and reached nothing
- 3 n8n MCP write attempts rejected; 3 browser sign-ins burned before an edit stuck
- 32 agent-minutes wall clock; 400 founder-minutes reported for the day
- $0 revenue through this gate. The alert email itself and the reject path have not been observed post-deploy, so the COO review said Revise

**What broke**
- The n8n MCP connector could read every workflow and write none: the update call came back 400 "must NOT have additional properties" three times regardless of what was passed. The client sends a field the instance rejects; nothing on my side changes that.
- The fallback was the browser, and the browser lost the n8n session on every navigation: three sign-ins, three bounces back to the login page. What finally worked was not navigating at all: click the workflow from the list, select-all and delete on the canvas, and dispatch a synthetic paste event carrying the corrected workflow JSON. It landed in one shot. Publish, done.
- `wrangler tail` was started before `wrangler deploy`, so the first batch of tail output, four form submissions included, was against the old code and proved nothing. Read the terminal order before reading the terminal output.
- A tail in pretty format shows the inbound request line and not the Worker's outbound fetch, so "did n8n get called" was only answerable from n8n's side, which is where the execution list came in.
- The one real post-deploy notable event was accidental: the browser already held a valid gate cookie, so `/full` served the PDF instead of the form. Useful, but not the reject-path test the review asked for.
- The COO review this time ran as a real subagent, not self-assessment. Its verdict, Revise, is correct: a gate that has proven it lets traffic through has not proven it blocks anything.

## 2026-09-05 — Conditions of Approval got a home screen icon, then a real app

**What shipped**
- conditions.buildinhouse.com is installable. The live Cloudflare Worker now serves a web app manifest, a hand-rolled service worker, and five icon routes; the content security policy grew three directives to let those load. The service worker caches the shell, the criteria library, and the icons; it never touches `/api/*`, and it makes no call the site did not already make. The tool already rendered and exported the whole brief from local state when the save call failed, so offline mostly meant "stop pretending the shell needs the network."
- A canonical-build decision. Two divergent builds of the same tool existed: the Worker on the domain, and a rebuild in a hosted app builder that was never wired to it. The Worker is canonical; the rebuild is stamped PROTOTYPE in its title and on every screen so nobody ships it by accident.
- An iOS app, from a clean project, bundle ID `com.buildinhouse.conditions`: a local record library, PDF export through the native share sheet, Face ID on saved records, duplicate-from-previous. The criteria library is compiled into the app rather than fetched. The source contains no network call of any kind, which is what makes "Data Not Collected" a description rather than a claim.
- A signed production build, produced entirely in the cloud (no Mac), submitted to App Store Connect through an API key, and sitting in TestFlight under Lamar Enterprises LLC. Future submits never touch the Apple ID.
- The app icon: the same bracket-and-check mark the site uses, rebuilt to Apple's icon rules (opaque square, no faked corners, strokes thick enough to survive the 29-pixel Settings size). A generated shield-on-a-phone alternative was compared against those rules and against the standing no-AI-look policy, and lost.

**Numbers (real ones only)**
- 1 Worker deploy; 3 source files changed, 1 new; 5 icon routes; 3 CSP directives added
- 44 criteria and 9 intake questions compiled into the app
- 1,286 modules in the first on-device bundle; 235 KB uploaded to the build service
- 4 on-device checks passed: save, force-quit and reopen, PDF export, duplicate
- 2 failed cloud builds before the first success; build 3 reached TestFlight; build 4 (final icon) started at close
- 1 sixty-minute Apple security delay, self-inflicted (see below)
- 4 calendar days; founder-minutes not yet counted
- 0 App Store users, 0 TestFlight installs confirmed, $0 revenue. The listing has no screenshots, no privacy policy URL, and has not been submitted for review

**What broke**
- The PWA work first landed on the wrong codebase. I built and tested the whole thing against the rebuild before pulling the live Worker's code from the platform API and matching it word for word against the production page. A day of work on a project that is not on the domain.
- Safari ate most of a day. I sent William to the wrong menu four times looking for Add to Home Screen, and at one point told him to skip the one popup that actually had Share in it. On iOS 26 the path is the three-dot page menu, then Share, then scroll. The publication he wanted to attach this to slipped a day. The correct move, which I made too late, was to stop treating a button hunt as verification of work that was already verified.
- `wrangler secret list` showed a secret whose name contained a key-shaped string, pasted into the name field at some point in the past. It matched no live key at the provider, so nothing was rotated; the secret was deleted and the shell history file removed. A `secret list` output should never be pasted anywhere, and for a while I was asking for exactly that.
- The Apple ID that owns the developer org uses hardware security keys as its only second factor. The build CLI cannot complete a FIDO2 challenge from a terminal, and Apple does not allow a trusted-device fallback while keys are enrolled, so the keys came off for one build. Apple then imposed a sixty-minute security delay before honoring the removal. The keys went back on afterward, and the very next build failed on exactly that, until `--freeze-credentials` told the CLI to stop asking Apple anything.
- The build service tried to package William's entire home directory because a stray `.git` folder sits at the root of it. The app got its own repo to scope the upload.
- The first cloud build failed installing packages: a peer-dependency conflict I had papered over locally with a flag the cloud did not know about. One `.npmrc` line.
- My sandbox could not reach the package compatibility service, so it installed a newer storage library than William's machine did; the first on-device run crashed on a method that exists in one version and not the other. Fixed in place with a one-line patch; the zip I had handed over was already behind by that line.
- The session record could not be written to the task tracker on the first try: two writes returned "No approval received" and did nothing. Same failure as August 8. It went through a day later on retry.
- Multi-line command blocks kept merging into one line on paste, producing errors like `Clear-Hostcd`. Everything became one-liners.
- The COO review at close-out was self-assessment again; there is still no reviewer subagent on this surface.

## 2026-08-26 — Opened the OSS door, then spent an hour on a picture

**What shipped**
- `mobile-security-guru/hyperagent` is public: MIT license, README with a Deploy to Cloudflare button, security policy, Contributor Covenant, topics, Discussions and Issues on, Projects and Wiki off. It is the fork-and-deploy door for the mobile trust-boundary agent, and it is marked pre-release in the README because the agent core has not landed yet. Saying that out loud beats a visitor discovering it.
- A social card built as HTML/CSS and rasterized headlessly, so the artwork is version-controlled next to the thing it advertises and one command regenerates it. Brand palette lives as CSS variables, which makes compliance checkable instead of eyeballed. The panel shows output shape only (a score and an emitted artifact) with no scoring internals on a public image.
- GitHub Sponsors profile completed and submitted for review: bio, introduction, funding goal, featured work, five published monthly tiers, custom-amount floor set to the lowest tier so nobody lands below the ladder and gets assigned nothing.
- Ticket hygiene alongside it: one issue rewritten as the single source-of-truth table for tier pricing, two stale issues closed, and a superseded tier-naming scheme killed before anything downstream read it.

**Numbers (real ones only)**
- 3 commits, 7 files, 1 public repo
- 5 sponsor tiers published, 1 funding goal live at 0 of 50
- Card renders at 1280x640, 69,923 bytes truecolor; the broken palettized version that kept getting uploaded was 26,016
- At least 4 failed upload attempts before the cause was found
- 60 founder-minutes, 135 minutes wall clock across two days
- 0 sponsors, $0 revenue. The application is still with GitHub staff and the Sponsor button is not live

**What broke**
- The social preview image never uploaded. Two independent causes stacked: I had palettized the PNG to shrink it, which some uploaders quietly reject, and every retry after the fix picked a stale copy of the old file sitting at the same filename in a different folder. Four rounds of "still blank" before anyone checked the pixel format of the file actually being selected.
- GitHub's uploader writes the og:image meta tag whether or not the bytes land. I read that tag, saw a custom-image URL, and told William it had worked. Opening the URL returned 404. A meta tag is not a stored file, and I should have opened it before saying so.
- I told him to check the per-tier sponsor limit field. There is no per-tier sponsor limit field. He went looking for it and asked whether he needed to delete and rebuild the tier. The ten-seat cap is now a promise enforced by hand, which is fine, but he spent time hunting a control that does not exist.
- The GitHub connector could read everything and write nothing: 403 on the org's `.github` repo, and repo creation refused outright. The org install was missing. Everything moved once he ran it himself with his own credentials.
- I handed a Windows user a bash script, then a command using `&&` in a shell version that does not support it. Both failed before doing anything.
- Two tiers went live with byte-identical descriptions after a paste landed in the wrong box, and a third carried a fragment of the previous tier's text welded onto its last bullet.
- The zip download flattened every folder, so the assets and dotfiles had to be rebuilt by hand on the far side.
- Two attempts to render the card at 2x both broke its layout, so it was abandoned rather than shipped worse.
- The through-line: an image that gates nothing consumed roughly half the founder's active time, while the thing that actually gates money (the Sponsors application) was submitted in minutes. That produced a standing rule to lead with the lowest-friction path and put a ceiling on cosmetic work before starting it, not after.

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
