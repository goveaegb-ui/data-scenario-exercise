Quick summary of issue: CSAT score dropped 12% in the Billing category, no change in headcount or ticket volume.
1. First response time by team Write a query returning the average first_response_minutes for tickets closed in the last 30 days, grouped by agent team.

SELECT a.team, ROUND(AVG(t.first_response_minutes),2) AS avg_first_response_minutes
FROM tickets t
JOIN agents a
ON t.agent_id = a.agent_id
WHERE t.closed_at >= CURRENT_DATE - INTERVAL '30 days'
AND t.first_response_minutes IS NOT NULL
GROUP BY a.team
ORDER BY avg_first_response_minutes ASC;

3. Agents with above-average reopen rates Write a query returning each agent's reopen rate (reopened tickets ÷ total tickets they handled) for agents whose rate is higher than their own team's average.

4. CSAT trend by category Write a query returning the average CSAT score per category, per month, for the last 3 months.

5. Digging in Beyond the three tables above, what additional data would you want to pull to understand the Billing drop — and which of the existing tables would you start with, and why?

6. Testing a theory Pick one specific theory for what might be driving the drop. State your theory, then write a query using the tables above that would help confirm or rule it out.

7. What you'd actually do Say your query in Question 5 showed that reopened tickets in Billing spiked, and most of the reopens trace back to one specific issue type (e.g., refund timing questions). What would you actually do with that in the next week? Be concrete. What would you say to the team, what (if anything) would you change in a process or macro, and how would you know if it worked?

8. Reporting up and coaching down You need to update your own manager on this in two sentences, and separately coach one agent on it in a 1:1. How would those two conversations differ?
