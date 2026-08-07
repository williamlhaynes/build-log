# Build log

Newest first. What shipped, real numbers, what broke.

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
