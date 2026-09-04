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

5. Testing a theory. Pick one specific theory for what might be driving the drop. State your theory, then write a query using the tables above that would help confirm or rule it out.

6. What you'd actually do. Say your query in Question 5 showed that reopened tickets in Billing spiked, and most of the reopens trace back to one specific issue type (e.g., refund timing questions). What would you actually do with that in the next week? Be concrete. What would you say to the team, what (if anything) would you change in a process or macro, and how would you know if it worked?

7. Reporting up and coaching down You need to update your own manager on this in two sentences, and separately coach one agent on it in a 1:1. How would those two conversations differ?
