# Technical Exercise: Support Data Scenario — Answers

## 1. First response time by team

```sql
SELECT
    a.team,
    AVG(t.first_response_minutes) AS avg_first_response_minutes,
    COUNT(*) AS tickets_closed
FROM tickets t
JOIN agents a ON t.agent_id = a.agent_id
WHERE t.closed_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY a.team
ORDER BY avg_first_response_minutes;
```

I filtered on `closed_at` (not `opened_at`) since the ask is specifically about tickets closed in the window. I also included a ticket count alongside the average — an average built on a handful of tickets isn't as trustworthy as one built on hundreds, and that context matters before anyone acts on the number. One assumption worth flagging: this includes tickets currently in `reopened` status as long as they have a `closed_at` in range; I'd confirm with the team whether that's the intended definition of "closed."

## 2. Agents with above-average reopen rates

```sql
-- Step 1 (inner query "ar"): reopen rate per agent
-- Step 2 (inner query "ta"): average of those per-agent rates, grouped by team
-- Step 3 (outer query): keep only agents whose rate beats their team's average
SELECT
    ar.name,
    ar.team,
    ar.reopen_rate,
    ta.team_avg_reopen_rate
FROM (
    SELECT
        t.agent_id,
        a.name,
        a.team,
        SUM(CASE WHEN t.reopened_count > 0 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS reopen_rate
    FROM tickets t
    JOIN agents a ON t.agent_id = a.agent_id
    GROUP BY t.agent_id, a.name, a.team
) ar
JOIN (
    SELECT
        team,
        AVG(reopen_rate) AS team_avg_reopen_rate
    FROM (
        SELECT
            t.agent_id,
            a.team,
            SUM(CASE WHEN t.reopened_count > 0 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS reopen_rate
        FROM tickets t
        JOIN agents a ON t.agent_id = a.agent_id
        GROUP BY t.agent_id, a.team
    ) agent_rates
    GROUP BY team
) ta ON ar.team = ta.team
WHERE ar.reopen_rate > ta.team_avg_reopen_rate
ORDER BY ar.team, ar.reopen_rate DESC;
```

I defined a "reopened ticket" as `reopened_count > 0` — a ticket that bounced back multiple times still counts once toward the rate, since the question is about how many of an agent's tickets didn't stick the first time, not total reopen events. The team average here is the average of individual agents' rates, not (team reopens / team tickets) — worth calling out since the two can diverge if agents handle very different volumes. I recompute the per-agent rate twice (once for the listing, once inside the team-average step) instead of reusing it — a little repetitive, but it keeps each piece of the query readable as its own step.

## 3. CSAT trend by category, by month

```sql
SELECT
    t.category,
    DATE_TRUNC('month', c.submitted_at) AS month,
    AVG(c.score) AS avg_csat,
    COUNT(*) AS responses
FROM csat_responses c
JOIN tickets t ON c.ticket_id = t.ticket_id
WHERE c.submitted_at >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '3 months'
GROUP BY t.category, DATE_TRUNC('month', c.submitted_at)
ORDER BY t.category, month;
```

I used `submitted_at` rather than the ticket's `opened_at`, since the question is about when the score was given, not when the underlying ticket started. Response count is included for the same reason as Q1 — a monthly average from 4 responses shouldn't be read the same way as one from 400.

## 4. Digging in

Beyond these three tables, I'd want the free-text side of things: agent notes or a resolution/reason code on Billing tickets, and the open-text field on CSAT responses if one exists. Right now the schema tells us that  CSAT dropped, not why — the reason codes and comments are the most direct read on that. I'd also pull a change log for the billing system, refund process, or pricing, so I can line up any recent change against the timing of the drop.

If there's any ticket-classification or AI tooling available (even something lightweight, like running a sample of ticket text through an LLM to bucket it into issue types), I'd lean on it here specifically — manually reading hundreds of Billing tickets to spot a pattern doesn't scale, but an automated first pass at tagging them by underlying issue would let me confirm a theory in hours instead of days, and turn "CSAT dropped 12%" into "62% of that drop traces to refund-timing questions" quickly.

Of the existing tables, I'd start with `tickets`. It already carries category, priority, `reopened_count`, and `first_response_minutes`, which is enough to slice the problem several ways (by agent, by day, by reopen status) before requesting anything new — I might find the answer without waiting on new data at all.

## 5. Testing a theory

Theory: The CSAT drop isn't about the quality of individual interactions — it's driven by a spike in reopened Billing tickets tied to refund-timing confusion. Customers are told a refund is processing, aren't given a clear timeframe, and contact us again before it lands. That extra round-trip is what's tanking the score, not the resolution itself.

```sql
SELECT
    DATE_TRUNC('month', opened_at) AS month,
    COUNT(*) AS total_tickets,
    SUM(CASE WHEN reopened_count > 0 THEN 1 ELSE 0 END) AS reopened_tickets,
    SUM(CASE WHEN reopened_count > 0 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS reopen_rate
FROM tickets
WHERE category = 'Billing'
  AND opened_at >= CURRENT_DATE - INTERVAL '4 months'
GROUP BY DATE_TRUNC('month', opened_at)
ORDER BY month;
```

I pulled 4 months instead of 3 so the month of the drop has a 3-month baseline to compare against. If reopen rate jumps sharply in the drop month relative to that baseline, it supports the theory — it doesn't prove causation on its own, but it's enough to justify pulling the reason codes/comments from Q4 to confirm the "refund timing" detail specifically.

## 6. What I'd actually do

Assuming the query confirms it — reopens in Billing spiked, and most trace back to refund-timing questions — here's what I'd do in the next week:

First, I'd read through a pool of the actual reopened tickets to make sure the pattern holds up in the details and isn't an artifact of how tickets get tagged. Then I'd tell the team plainly what's going on: customers aren't unhappy with how the ticket was handled, they're unhappy that they had to ask twice — so the fix isn't "be nicer," it's "don't make them come back."

On process, I'd make two changes. First, a reactive one: update the Billing macro so the first response always includes a concrete refund timeframe instead of a generic "it's processing" — that alone should cut a chunk of the reopens. Second, and more important long-term: a proactive communication step so the question never becomes a ticket at all. Instead of waiting for the customer to ask, we'd push a status update at the moments that matter — e.g., an automatic note when a refund is actually issued, or a scheduled check-in during the expected window. If we don't have automated messaging for this yet, I'd start with a manual version (a macro agents send at ticket close: "here's when to expect it, here's how you'll know it landed") while we scope the automated version with product. The point isn't just to lower the reopen rate on tickets we already have — it's to shrink Billing volume overall, which frees agents up for the tickets that actually need a person (disputes, edge cases) instead of status checks we could have prevented.

To know if it worked, I'd track three things weekly for the following 2–4 weeks: reopen rate on Billing tickets (should drop), total Billing ticket volume (should also drop, not just shift), and Billing CSAT (should recover). If reopens drop but total volume doesn't, that tells me the proactive piece isn't landing yet even if the reactive fix is working.

## 7. Reporting up and coaching down

To my manager: "The Billing CSAT drop traces mainly to a spike in reopened tickets around refund-timing confusion — customers are contacting us again before their refund lands, which is a communication gap, not a quality-of-service problem. We're rolling out a clearer refund-timeline macro this week plus a proactive status update so customers don't have to ask, and I'll have reopen rate, ticket volume, and CSAT numbers back to you in about two weeks to confirm it's working."

In a 1:1 with a specific agent: Much more concrete and example-driven. I'd pull 2–3 of their actual tickets with a refund-related reopen, walk through them together, and ask how they currently explain timing to customers. I'd introduce the updated macro as a tool to close things out in one touch instead of two, not as a correction — the framing is "here's something that'll make your day easier and your numbers better," not "you did this wrong."

The difference between the two: the manager conversation is short, forward-looking, and about the plan and the numbers — no ticket-level detail. The coaching conversation is specific, uses real examples, and is about building a skill collaboratively rather than assigning blame for something the agent didn't cause (the process, not the agent, created the gap).
