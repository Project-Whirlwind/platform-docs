# MindBlossom marketing strategy and operating runbook

Status: Draft for review. Author: Fable, 2026-07-07.
Tracking: platform-docs#3.

Operating constraint up front: **no paid spend beyond small test budgets until
the safety and max-user thresholds land (mindblossom#55).** Driving signups
into an app with no user cap, no per-user AI budget, and (until mindblossom#77)
an open SSRF is how you turn a marketing win into a cost or security incident.
Sequence: ship #77 and #55, then open the taps.

---

## 1. Positioning

**One line:** MindBlossom is your personal AI second brain for everything that
lands in your inbox and texts, and it is owned by its users.

**The angles, ranked by expected pull:**

1. **Capture from anywhere, recall on demand.** Text or forward anything to
   Blossom; it saves, tags, and answers. This is the concrete, demoable value
   and the top of every funnel.
2. **Owned by you, not private equity.** Perplexity and Poke are PE-owned;
   Honey is PayPal's (Peter Thiel). MindBlossom returns the value to users
   (revenue-share/co-op intent via revenue-ledger). This is the differentiator
   that earns loyalty and press, but it is a *belief* sell — lead with utility,
   close with ownership. Never claim payouts that do not exist yet.
3. **Price-watching that texts you.** Concrete, viral-adjacent, already built
   (price-watcher + comm-gateway). Good for a specific audience (deal hunters)
   and demos.

**Voice:** plain, direct, a little warm — the same as Blossom. No hype. The
audience that responds to "not owned by PE" can smell marketing voice.

---

## 2. Funnel

Landing (`mindblossom.app`) → early access → registration → onboarding →
first captured link (activation) → weekly active capture (retention) →
free-to-Pro conversion.

Instrument every step. Top of funnel already exists: the `early_access_signups`
table and the admin dashboard. What to add:

| Step | Metric | Where measured |
|------|--------|----------------|
| Landing view → email submit | early-access conversion rate | landing analytics + `early_access_signups` |
| Signup → onboarding complete | activation-1 | `users.onboarded_at` |
| Onboarding → first captured link | **activation-2 (the north star)** | first entry per user |
| Week N retention | weekly active capture | entries per user per week |
| Free → Pro | conversion rate | `users.plan` transitions (Stripe billing, PR #75) |
| Pro churn | monthly churn | `plan_cancel_at_period_end` + subscription.deleted |

**North star: activation-2 (first captured link within 24h of signup).** It is
the moment the product's value becomes real; optimize the funnel toward it, not
toward raw signups.

---

## 3. Channels (test small, kill fast)

Each channel gets a small test budget, a hypothesis, and a kill criterion. Do
not scale any channel before it beats the CAC ceiling (Section 4).

| Channel | Hypothesis | Test budget | Kill criterion |
|---------|------------|-------------|----------------|
| **Communities** (Reddit r/productivity, r/selfhosted, HN, indie forums) | The ownership + second-brain story resonates with privacy/ownership-minded users; near-zero CAC | Time, not $ | No signups from 3 genuine posts/comments |
| **Search ads** (Google) | Intent exists for "AI note capture", "SMS to notes", "forward email to AI" | $300 over 2 weeks | CAC > ceiling after 2 weeks, or activation-2 < 30% |
| **Content/SEO** | "Second brain", "forward email to notes", price-tracking how-tos rank | Time | No organic signups after 6 published pieces |
| **Social** (short demo videos of texting Blossom) | The capture demo is inherently showable | $200 boosted | CPM fine but activation-2 < 25% |
| **Referral** (once retained users exist) | Owned-by-users story makes people evangelize | Product work | < 0.15 invites per active user |

**Keyword clusters to track (search):**
- Capture: "text to notes app", "sms note taking", "forward email to notes",
  "email to second brain".
- Category: "AI second brain", "personal knowledge assistant", "Notion AI
  alternative", "Mem alternative".
- Ownership/anti-PE: "privacy AI assistant", "user owned AI", "open alternative
  to Perplexity/Poke", "Honey alternative".
- Deals: "price drop alert text", "price tracker sms".

Track branded terms separately so you do not pay to capture demand you already
earned.

---

## 4. Metrics and levers

**Core metrics:**
- **CAC** per channel. Ceiling = Pro LTV × 0.33. At $9/mo, assume (conservatively)
  6-month average retention → LTV ≈ $54 → **CAC ceiling ≈ $18**. Re-derive once
  real churn data exists; do not trust the assumption past the first cohort.
- **Activation-2** (north star, Section 2).
- **Retention:** weekly active capture, and free→Pro conversion by cohort.
- **Unit economics:** per-user AI + comms cost (from the #55 usage accounting)
  vs revenue. This is why #55 blocks paid scale — you cannot compute LTV
  honestly without knowing per-user cost.

**Levers, ranked by expected impact:**
1. **Activation onboarding.** Get the user to capture their first link inside
   onboarding (send a sample, or prompt them to forward one email). Biggest
   lever; cheapest.
2. **Time-to-value in the demo.** The landing should show the text-Blossom loop
   in the hero, not describe it.
3. **Pro trigger placement.** Surface the Pro upgrade at the moment a free user
   hits the tag/auto-tag cap (the gate already exists), not on a settings page.
4. **Referral loop** once retention is proven.
5. **Channel mix** — only after 1-3 are working; acquiring users into a leaky
   funnel wastes spend.

---

## 5. Weekly operating cadence

Every week:
- **Read:** signups, activation-2 rate, weekly active capture, free→Pro,
  per-channel CAC and spend, per-user cost trend (once #55 lands).
- **Decide:** any channel past its kill criterion gets cut; any channel under
  the CAC ceiling with positive activation gets a bit more budget.
- **Ship:** one funnel improvement aimed at the weakest step (usually
  activation early on).
- **Kill:** stop defending a channel that missed its criterion two weeks
  running. The point of small test budgets is cheap, fast negative results.

Monthly: re-derive the CAC ceiling from real churn; review Pro cohort
retention; decide whether the ownership/co-op story is ready to move from
"belief close" to a concrete revenue-share announcement (gated on
revenue-ledger being real).

---

## 6. Sequencing (what to do in what order)

1. Ship mindblossom#77 (SSRF) and #55 (thresholds + max-user cap). Non-negotiable
   before paid acquisition.
2. Instrument the funnel (Section 2 table). You cannot optimize what you cannot
   see.
3. Nail activation in onboarding (lever 1) with the small organic/community
   audience you already have.
4. Only then turn on paid test budgets, one channel at a time, honoring kill
   criteria.
5. Add referral once retention is proven and the ownership story has a concrete
   hook.

---

## 7. Open questions for the reviewer

- CAC ceiling assumes 6-month retention and 33% CAC/LTV. Comfortable, or set a
  tighter ceiling until real data lands?
- How hard to lead with the anti-PE/ownership angle now vs after there is a real
  revenue-share mechanism? Recommendation: utility-led everywhere, ownership as
  the closer, no payout claims until revenue-ledger is live.
- Deal-hunter (price-watcher) audience: treat as a separate funnel/landing, or
  fold into the main second-brain story? Recommend a dedicated landing section
  and keyword set, same signup.
