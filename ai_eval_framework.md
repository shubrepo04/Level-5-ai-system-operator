# AI Eval Framework — Support Ticket Triage AI
**Feature evaluated:** 2026-03-20

---

## WHAT WE ARE EVALUATING

An AI that reads incoming support tickets and generates a summary, resolution category, and priority level before a human agent sees the ticket — working well means the AI saves agents meaningful time without creating errors that increase handle time or harm customers.

---

## THE 4 EVAL DIMENSIONS

### 1. Correctness

**Is the AI output actually right?**

*What we measure:*
- **Category accuracy:** % of tickets where AI-suggested category matches the category the agent ultimately uses (after any override). This is our primary correctness signal because agents are the ground truth.
- **Priority accuracy:** % of tickets where AI-suggested priority (High / Medium / Low) matches the priority the agent sets at ticket close.
- **Summary completeness:** Does the AI summary contain the key issue, affected user/account, and any error codes or specific details mentioned in the ticket? Measured via weekly human spot-check sample.

*Target scores:*
- Category accuracy: ≥ 85%
- Priority accuracy: ≥ 80%
- Summary completeness: ≥ 90% (human-reviewed sample of 50 tickets/week)

*How we sample:*
- Automatic: log every agent override as an implicit correctness signal (override = AI was wrong)
- Manual: weekly random sample of 50 tickets — a human reviewer scores each summary on a 3-point scale (Complete / Partially complete / Missing key info)
- Priority audit: weekly review of all tickets that were escalated after initial assignment — did AI mark them Low or Medium?

---

### 2. Relevance

**Is the AI output useful for what the agent actually needed?**

*What we measure:*
- **Summary usefulness:** Do agents read the AI summary before responding, or do they scroll past it and re-read the raw ticket? Measured via UI event logging (did the agent expand/scroll the ticket after seeing the summary?).
- **Category relevance:** Is the suggested category specific enough to route the ticket to the right sub-queue, or is it too broad to be useful?
- **Time-to-first-response:** Are agents responding faster on tickets with AI suggestions vs. without? (A/B or pre/post comparison)

*Signals that output was irrelevant:*
- Agent overrides the category within 5 seconds of opening the ticket (didn't read it, just changed it)
- Agent opens the raw ticket immediately after the summary without any pause (summary was not useful enough to stop at)
- Category is "General" more than 15% of the time (AI is falling back to a catch-all rather than being specific)
- Time-to-first-response is not improving vs. baseline

---

### 3. Safety

**Can the AI output cause harm?**

*Specific risks for this feature:*

1. **Incorrect priority downgrade:** AI marks a High priority ticket as Low. Agent accepts without reviewing. Customer with a critical issue waits too long. This is the highest-severity failure mode — it has direct customer harm potential.

2. **Incorrect category routing:** Ticket is routed to the wrong team. Resolution is delayed. Customer frustration compounds.

3. **Summary omits sensitive information:** A ticket mentions a security issue, data breach concern, or regulatory complaint. AI summary doesn't flag it. Agent proceeds with a standard resolution. Escalation is missed.

4. **PII exposure in summaries:** AI summary reproduces verbatim sensitive customer data (account numbers, passwords, health information) in a field that may be logged, indexed, or visible to more people than the original ticket.

5. **Bias in priority assignment:** AI consistently rates tickets from certain account types or regions as lower priority. This could create systematic service inequality that is invisible without an explicit audit.

*Checks that catch risks before users see them:*
- **Priority floor rule:** Any ticket containing keywords associated with security, legal, compliance, or data breach is automatically flagged High regardless of AI output. Rule-based override, not AI.
- **PII scrubbing:** Summary field passes through a PII detection layer before display. Detected PII is replaced with `[REDACTED]`.
- **Equity audit:** Monthly analysis of priority distribution by customer segment, region, and account tier. Flag if any segment is consistently rated 10%+ lower than average.
- **Escalation mismatch log:** Any ticket that is manually escalated after AI assigned Low or Medium is flagged for review. Reviewed weekly.

---

### 4. User Trust

**Are agents actually trusting and using the AI output?**

*Behavioral signals that show trust:*
- Agent accepts AI category without override
- Agent accepts AI priority without override
- Agent's first response time is faster than pre-AI baseline
- Agent does not re-open the raw ticket immediately after seeing the summary
- Agent uses the summary language in their response (copy/paste or paraphrase detected)

*Behavioral signals that show distrust:*
- Override rate > 30% on category or priority — agents are consistently correcting the AI
- Agents override within 2 seconds of ticket open — not reading the suggestion, just dismissing it
- Agent feedback (if collected) rates summaries as "Not helpful"
- Agents report in qualitative feedback that they check the raw ticket every time regardless of the summary

*Trust floor:* If the override rate exceeds 35% for two consecutive weeks, the AI is not trusted enough to be useful. Investigate root cause before continuing to show suggestions.

---

## EVAL METHODS

**Human review: YES**
- *Why:* Summary quality cannot be measured automatically. A human must assess whether the summary is complete and accurate. Weekly sample of 50 tickets is the minimum viable human review cadence for this feature.
- *Who does it:* A senior support agent or QA reviewer — not the same agents using the tool daily.

**LLM as judge: YES (secondary)**
- *Why:* Can scale the summary completeness check beyond the 50-ticket human sample. Use an LLM to score summaries against a rubric: "Does this summary include the core issue, the affected account, and any specific error details?" Score 1-3. Flag any scoring below 2 for human review.
- *Caution:* LLM-as-judge cannot evaluate correctness of category or priority — only a human with domain context can do that.

**User feedback signals: YES (primary ongoing signal)**
- *Why:* Agent overrides are the most valuable real-time signal we have. Every override is a labeled data point: AI suggested X, human chose Y. This data should be logged, analyzed weekly, and used to retrain or recalibrate.
- *Implementation:* Log category override, priority override, and time-between-open-and-override for every ticket. Build a weekly override report.

**A/B testing: YES (at launch and after major changes)**
- *Why:* We need a baseline comparison to know whether the AI is actually helping. Run an A/B at launch: 50% of tickets get AI suggestions, 50% don't. Compare time-to-first-response, handle time, and CSAT.
- *After launch:* Re-run A/B whenever the model changes significantly or when trust signals degrade.

**Golden dataset: YES (quarterly)**
- *Why:* Build a set of 100–200 tickets with known correct categories and priorities, verified by senior agents. Run AI outputs against this dataset quarterly to track accuracy over time and catch model drift.
- *Maintenance:* Update the golden dataset quarterly to include new ticket types and edge cases.

---

## METRICS DASHBOARD

| Metric | What it measures | Target | Red flag threshold |
|--------|-----------------|--------|--------------------|
| Category acceptance rate | % of tickets where agent keeps AI category | ≥ 85% | < 70% for 2 consecutive weeks |
| Priority acceptance rate | % of tickets where agent keeps AI priority | ≥ 80% | < 65% for 2 consecutive weeks |
| Priority downgrade miss rate | % of escalated tickets that AI marked Low/Medium | < 5% | > 10% in any week |
| Summary completeness score | Human-reviewed score (1–3) on 50-ticket sample | ≥ 2.5 avg | < 2.0 avg in any review |
| Time-to-first-response delta | Avg agent response time with AI vs. baseline | ≥ 10% faster | Slower than baseline for 2 weeks |
| Override within 2 seconds | % of overrides done without reading | < 10% | > 25% (agents dismissing without reading) |
| Escalation mismatch rate | Escalated tickets AI rated Low/Medium | < 5% | > 10% |
| PII detection trigger rate | % of summaries where PII scrubbing activated | Monitor for trend | Sharp spike = prompt injection risk |

---

## WHEN TO ACT

**Investigate:**
- Category or priority acceptance rate drops below target for one week
- Any week with escalation mismatch rate > 5%
- Summary completeness score drops below 2.2 in human review
- Override-within-2-seconds rate exceeds 20% (agents not trusting the tool)

**Roll back the feature (show no AI suggestion):**
- Priority acceptance rate drops below 65% for two consecutive weeks — AI is wrong more often than it helps
- Escalation mismatch rate exceeds 10% in any single week — a safety threshold, not a performance threshold
- PII scrubbing failures detected in production (any confirmed case of unredacted PII in summaries)
- Agents submit formal complaints that the AI is slowing them down, not helping (qualitative trigger — escalate to PM review immediately)

**Retrain or change the model:**
- Golden dataset accuracy drops more than 5 percentage points from the launch baseline (model drift)
- Category accuracy on a specific ticket type falls below 70% (model has a systematic blind spot)
- New ticket types emerge (product launches, outages, policy changes) that the model has never seen — proactive retraining, not reactive
- Equity audit finds a segment consistently rated 10%+ below average priority — retraining required with bias correction

---

## EVAL CADENCE

| Evaluation type | Cadence | Owner |
|----------------|---------|-------|
| Override rate report (category + priority) | Daily automated | Engineering / Data |
| Escalation mismatch review | Weekly | Support Lead |
| Human summary spot-check (50 tickets) | Weekly | QA Reviewer |
| Override-within-2-seconds trend | Weekly | PM |
| A/B performance review | At launch, then per major model change | PM + Data |
| Golden dataset accuracy run | Quarterly | Engineering |
| Equity audit (priority by segment) | Monthly | PM + Support Lead |
| Full eval framework review (are these metrics still the right ones?) | Quarterly | PM |
