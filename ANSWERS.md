Quick summary of issue: CSAT score dropped 12% in the Billing category, no change in headcount or ticket volume.
1. First response time by team. Write a query returning the average first_response_minutes for tickets closed in the last 30 days, grouped by agent team.
```
SELECT a.team, ROUND(AVG(t.first_response_minutes),2) AS avg_first_response_minutes
FROM tickets t
JOIN agents a
ON t.agent_id = a.agent_id
WHERE t.closed_at >= CURRENT_DATE - INTERVAL '30 days'
AND t.first_response_minutes IS NOT NULL
GROUP BY a.team
ORDER BY avg_first_response_minutes ASC;
```
2. Agents with above-average reopen rates. Write a query returning each agent's reopen rate (reopened tickets ÷ total tickets they handled) for agents whose rate is higher than their own team's average.
```
WITH agent_metrics AS (
    SELECT 
        a.agent_id,
        a.name,
        a.team,
        COUNT(t.ticket_id) AS total_tickets,
        ROUND(
            SUM(CASE WHEN t.reopened_count > 0 THEN 1 ELSE 0 END)::numeric 
            / NULLIF(COUNT(t.ticket_id), 0), 
            4
        ) AS agent_reopen_rate
    FROM agents a
    JOIN tickets t 
        ON a.agent_id = t.agent_id
    GROUP BY a.agent_id, a.name, a.team
    HAVING COUNT(t.ticket_id) > 0
),
team_benchmarks AS (
    SELECT 
        agent_id,
        name,
        team,
        total_tickets,
        agent_reopen_rate,
        ROUND(AVG(agent_reopen_rate) OVER (PARTITION BY team), 4) AS team_avg_reopen_rate
    FROM agent_metrics
)
SELECT 
    agent_id,
    name,
    team,
    total_tickets,
    agent_reopen_rate,
    team_avg_reopen_rate
FROM team_benchmarks
WHERE agent_reopen_rate > team_avg_reopen_rate
ORDER BY team, agent_reopen_rate DESC;
```
3. CSAT trend by category. Write a query returning the average CSAT score per category, per month, for the last 3 months.
```
SELECT 
    t.category,
    DATE_TRUNC('month', c.submitted_at)::date AS csat_month,
    COUNT(c.response_id) AS total_responses,
    ROUND(AVG(c.score), 2) AS avg_csat_score
FROM tickets t
JOIN csat_responses c 
    ON t.ticket_id = c.ticket_id
WHERE c.submitted_at >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '2 months'
GROUP BY t.category, DATE_TRUNC('month', c.submitted_at)
ORDER BY t.category, csat_month ASC;
```
4. Digging in. Beyond the three tables above, what additional data would you want to pull to understand the Billing drop — and which of the existing tables would you start with, and why?

First, since there were no changes in staffing headcount or ticket volume, it would be important to query deeper into the 'Billing' category within the 'tickets' table. I would do a JOIN with the 'csat_responses' table to see if there is a specific subcategory or issue that is causing the drop in CSAT scores. This could help with determining if the problem lies within a reopen rate, long resolution time, etc.

As for additional data I would want to pull, I think it's important to gather the qualitative CSAT feedback. By analyzing the customer's comments, it can help identify any pain points within the product itself or with the agent support. For example, maybe there is a trend where some specific agents are not providing a solution for the root cause, but just giving generic responses that don't solve the issue. 

I would also like to query data on specific macros or canned responses used within the Billing tickets to determine if the information is innacurate or outdated.

5. Testing a theory. Pick one specific theory for what might be driving the drop. State your theory, then write a query using the tables above that would help confirm or rule it out.

Theory: The 12% drop in Billing CSAT is driven by a drop in First Contact Resolution (FCR). Customers are receiving incomplete initial responses, leading to a spike in ticket reopens, and customers who experience reopens rate their satisfaction significantly lower. In support operations, customers rarely complain about a resolution if it's handled right the first time. They complain when they have to follow up repeatedly.
```
SELECT 
    DATE_TRUNC('month', t.opened_at)::date AS ticket_month,
    COUNT(t.ticket_id) AS billing_ticket_volume,
    -- Reopen Rate across all Billing tickets
    ROUND(
        SUM(CASE WHEN t.reopened_count > 0 THEN 1 ELSE 0 END)::numeric 
        / NULLIF(COUNT(t.ticket_id), 0), 
        4
    ) AS billing_reopen_rate,
    -- CSAT: Single-Touch (FCR) vs. Reopened Tickets
    ROUND(AVG(CASE WHEN t.reopened_count = 0 THEN c.score END), 2) AS avg_csat_first_contact_resolved,
    ROUND(AVG(CASE WHEN t.reopened_count > 0 THEN c.score END), 2) AS avg_csat_reopened
FROM tickets t
LEFT JOIN csat_responses c 
    ON t.ticket_id = c.ticket_id
WHERE t.category = 'Billing'
  AND t.opened_at >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '2 months'
GROUP BY DATE_TRUNC('month', t.opened_at)
ORDER BY ticket_month ASC;
```
6. What you'd actually do. Say your query in Question 5 showed that reopened tickets in Billing spiked, and most of the reopens trace back to one specific issue type (e.g., refund timing questions). What would you actually do with that in the next week? Be concrete. What would you say to the team, what (if anything) would you change in a process or macro, and how would you know if it worked?

First, I would audit a sample of the reopened tickets to see the type of information the agents are providing and if they are using a macro or canned responses per a specific process they are following. I will assume that the macro indicates that the refund timing is around 10 business days. However, per the audit, the reopened tickets are regarding refund timing for Bank of America, which has a fictional refund timing of 20 business days. So the customer comes back after the 10 day mark, asking about their refund, continuously re-opening the ticket. 

Having identified the issue, we will need to update the macro to flag refund timing for Bank of America. We would also need to update both public help centers and our own internal knowledge bases to ensure this information is updated.

Regarding briefing the team, I would send a Slack message flagging the issue, indicating that we're updating the information in the macro and all relevant information sources. I would also acknowledge that this is an issue that has also affected the team and that with these changes we expect for them to have less friction. I would follow up in our team meeting and also ask for them to flag if there is a snag with the changes.

To know that these changes worked, I would do another round of ticket audits, check in with the agents, and do another query of reopen rate on tickets as well as First Contact Resolution rate. Longer term, the Billing CSAT score should recover from the 12% drop.

7. Reporting up and coaching down You need to update your own manager on this in two sentences, and separately coach one agent on it in a 1:1. How would those two conversations differ?

To the manager:
The 12% Billing CSAT drop was driven by an influx of ticket reopens caused by our refund macro understating settlement timelines for institutions like Bank of America. We updated customer facing docs and internal macros with accurate 20-day expectations, and we expect daily reopen rates to drop immediately with Billing CSAT recovering within 2–3 weeks (details and tracking linked in this brief one-pager).

1:1 session with agent:
I noticed your reopen rate on Billing tickets increased recently, especially around refund requests. Let's pull up two of these tickets together to see what the customers were experiencing.

When we tell a customer their refund takes 10 days, but their bank actually takes 20, they panic on day 11 and write back in. That creates a frustrated customer and forces you to handle the exact same inquiry twice.

We just rolled out an updated refund macro that flags these institution-specific timelines upfront. Let’s start using this on all your refund tickets this week to set clear expectations on day one and save you from handling repeat replies.

Please let me know how it goes and we can check in on our next 1:1.
