# AI Cost & Reliability Modeler — AI Meeting Summarizer
**Feature modeled:** 2026-03-20

---

## COST MODEL

**Token assumptions (flagged):**
- Input: 2,000-word transcript ≈ 2,600 tokens + ~200 token system prompt = ~2,800 tokens per call
- Output: 200-word summary ≈ 260 tokens per call
- Pricing assumption: Claude claude-sonnet-4-6 at $3.00 / 1M input tokens, $15.00 / 1M output tokens *(verify current pricing at anthropic.com — rates change)*

**Usage math:**
- 3 meetings/user/week × 4.33 weeks/month = ~13 API calls/user/month

| Scale | Monthly API calls | Estimated monthly cost | Cost per user |
|-------|------------------|----------------------|---------------|
| Current (500 users) | 6,500 | ~$88 ($54 input + $25 output) | ~$0.18 |
| 10x growth (5,000 users) | 65,000 | ~$877 | ~$0.18 |
| 100x growth (50,000 users) | 650,000 | ~$8,775 | ~$0.18 |

**Viability verdict: ECONOMICALLY SAFE at all three scales.**
At $0.18/user/month, the AI cost is roughly 1–2% of a typical $10–20/month SaaS seat price. Cost per user stays flat as you scale because usage pattern is per-user, not shared. No red flag at any scale in this model.

*The risk is not current cost — it's price sensitivity to model cost increases. See Reliability Risks.*

---

## COST OPTIMIZATION OPTIONS

**1. Enable prompt caching on the system prompt**
- What it is: Anthropic's prompt caching discounts repeated prompt prefixes by ~90%. Your system prompt (instructions for how to summarize, format, extract owners) is identical on every call.
- Estimated savings: 5–10% of total cost (system prompt is ~200 of ~2,800 input tokens — small but free to implement)
- Tradeoff: Minor implementation work. No quality impact.

**2. Pre-process transcripts before sending to the API**
- What it is: Run a lightweight local filter that strips filler content (small talk, "um", "uh", repeated phrases, scheduling logistics) before the transcript hits the API. Target: reduce 2,000-word input to ~1,200–1,400 words.
- Estimated savings: 30–40% on input token cost (input is already the smaller cost driver vs. output, so net ~15–20% total)
- Tradeoff: Requires a pre-processing step. Risk of stripping content that was actually decision-relevant — needs testing and a conservative filter threshold.

**3. Route short or low-content meetings to a smaller model**
- What it is: If the transcript is under 500 words or flagged as "no structured content" (see interaction design edge case), route to a cheaper, smaller model (e.g., Claude Haiku) instead of Sonnet.
- Estimated savings: 60–70% on calls routed to the smaller model. If 20% of meetings qualify, that's ~12–14% total savings.
- Tradeoff: Lower-quality summaries for edge cases. Must define the routing threshold carefully and test output quality at the boundary.

**4. Async batch processing with off-peak scheduling**
- What it is: Since meeting summaries are non-blocking (user navigates away while it generates), queue low-priority summaries and process them during off-peak API hours. Anthropic does not currently offer time-of-day pricing, but this positions you for batch API pricing if/when it becomes available — and reduces peak concurrency costs in your own infrastructure.
- Estimated savings: 0% today on API cost; potential 30–50% if batch pricing becomes available. Immediate benefit: lower infrastructure cost from peak load reduction.
- Tradeoff: Summaries may not be ready when users return immediately after a meeting. Acceptable if notification UX is solid.

---

## LATENCY PROFILE

**For a 2,800-token input / 260-token output call to Claude claude-sonnet-4-6:**

- Best case: 1,500 ms (low API load, short output, fast network)
- Typical case: 3,000–5,000 ms
- Worst case: 12,000–20,000 ms (high API load, degraded performance period)

**Damage threshold for this feature: ~15 seconds**
This feature is designed async — the user logs a meeting and navigates away while the summary generates. This makes it unusually tolerant of latency compared to inline AI features. A 5-second generation time is invisible to the user. A 10-second generation time is also acceptable as long as the notification UX works correctly (in-app + email when ready).

**What happens if AI response takes 10 seconds:**
In this design — nothing visible to the user. The loading state persists in the background and a notification fires when done. The only failure is if the user returns to the timeline within 10 seconds expecting the summary to be there — they see "Generating summary…" and wait. Not damaging unless it becomes the norm.

**What actually damages UX:** Summaries that take more than 5 minutes, or that silently fail without surfacing the "Summary unavailable" fallback. Latency is not the risk here — silent failure is.

---

## RELIABILITY RISKS

**1. Full API outage (1–4 hours)**
- Impact: All meeting summaries logged during the outage fail to generate. Users return to timeline and see either a broken loading state or no summary card. If the failure is silent, users don't know to retry.
- Mitigation: Implement a job queue with retry logic — failed generation attempts are retried automatically when the API recovers, with no user action required. Surface a clear "Summary unavailable — we'll retry shortly" state so users know the system knows it failed.

**2. Degraded API performance (slow responses)**
- Impact: Summaries take 30–120 seconds instead of 3–5 seconds. Async design absorbs most of this — users won't notice unless they're watching the loading state. At 50,000 users and 650,000 calls/month (~22,000 calls/day), a degraded period means a backlog builds in your job queue.
- Mitigation: Set a generation timeout at 60 seconds. Anything that exceeds it moves to a "retry" state. Monitor queue depth as a leading indicator of API health — a growing queue before error rates spike gives you early warning.

**3. Model behavior changes without notice**
- Impact: Anthropic updates Claude claude-sonnet-4-6's behavior and summaries start formatting differently, extracting owners less reliably, or producing longer/shorter output than expected. Users notice inconsistency across meetings. Trust erodes gradually, not suddenly — which makes it harder to detect.
- Mitigation: Run your golden dataset (see eval framework) after every model update notification. Subscribe to Anthropic's model changelog. Add automated output structure validation — if the summary doesn't contain the expected sections (Key Decisions, Action Items, Overview), flag it before displaying to the user.

**4. Anthropic raises prices by 3x**
- Impact: Monthly cost at 50,000 users goes from ~$8,775 to ~$26,325. Cost per user rises from $0.18 to ~$0.53/month. Still not economically unviable at typical SaaS pricing, but it meaningfully compresses margin and creates pressure to pass cost on or reduce quality.
- Mitigation: This is the most important long-term risk to model. Two actions: (1) implement prompt caching and transcript pre-processing now so you have headroom before a price increase forces a rushed optimization; (2) abstract your LLM provider behind a provider interface so you can switch to a competing model (GPT-4o, Gemini) within days, not months, if pricing becomes untenable.

---

## SINGLE POINTS OF FAILURE

**Single point 1: Anthropic API availability**
- Dependency: 100% of summary generation depends on a single external API with no fallback.
- Fallback: The feature degrades gracefully to "Add your own notes" manual mode — which is the right product behavior — but you have no AI fallback. If Anthropic has a multi-hour outage during a heavy meeting day, a meaningful portion of your users will have no summaries for that day.
- What to do: Either integrate a secondary LLM provider as a fallback (called only when Anthropic returns errors), or accept the degradation and ensure the manual fallback UX is excellent.

**Single point 2: The transcript input pipeline**
- Dependency: If transcript ingestion breaks (file parsing fails, encoding issue, upload times out), the AI never gets called and the user sees a silent failure.
- Fallback: Currently none described. Every input failure needs to surface the same "Add meeting notes" prompt so users have a manual path.
- What to do: Validate transcript input at ingestion time, not at AI call time. Surface errors immediately when the transcript is logged, not after the user waits for a summary.

**Single point 3: The job queue**
- Dependency: If summaries are processed async via a queue, the queue infrastructure is a single point of failure. A queue backlog, crash, or misconfiguration means summaries never generate — and users may not know why.
- Fallback: Dead letter queue for failed jobs + monitoring alert if queue depth exceeds a threshold.

---

## RECOMMENDATIONS

1. **Build retry logic and a dead letter queue into the async job system before launch** — at 500 users and 6,500 calls/month, a 1-hour Anthropic outage affects ~270 meetings; without automated retry, those summaries are permanently lost and users have no recourse.

2. **Abstract your LLM provider behind a single interface now, not after your first price increase** — at 50,000 users you will spend ~$105,000/year on inference; vendor lock-in at that scale gives Anthropic full pricing leverage over your margin, and switching under pressure takes months without a provider abstraction layer already in place.

3. **Implement output structure validation before displaying summaries to users** — model behavior drift is invisible without it; add a check that every generated summary contains the required sections (Key Decisions, Action Items, Overview) and falls within expected token range, and route any summary that fails validation to a "Summary unavailable" fallback rather than displaying malformed output.

---

*Assumptions made: Claude claude-sonnet-4-6 priced at $3.00/1M input, $15.00/1M output tokens. Token estimates based on ~1.3 tokens/word. Usage assumes even distribution across 4.33 weeks/month. Growth to 50,000 users assumes proportional usage pattern (3 meetings/user/week maintained). Verify current API pricing before using cost figures in financial planning.*
