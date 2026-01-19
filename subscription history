# [Multiply](https://solvit.space/coding/2997)

# Description 
This code does not execute properly. Try to figure out why.

# My Solution

```sql

with transitions as (
    SELECT 'Free' status_from ,'Paid' status_to, 'Convert' event
    union all 
    SELECT 'Paid','Free','ReverseConvert'
    union all 
    SELECT 'Paid','Non-member','Cancel'
    union all 
    SELECT 'Free','Non-member','Cancel'
    union all 
    SELECT 'Non-member','Paid','ColdStart'
    union all 
    SELECT 'Non-member','Free','WarmStart'
    union all 
    SELECT 'Non-member','Non-member','Non-member'
    union all 
    SELECT 'Paid','Paid','Renewal'
    union all 
    SELECT 'Free','Free','Renewal' ),
status_rnk as (
    SELECT *, 
        ROW_NUMBER() OVER (
            PARTITION BY customer_id ORDER BY membership_start_date) rnk
    FROM subscriptions)

select *
from(
    SELECT 
    coalesce(st1.customer_id, st2.customer_id) as customer_id,
    coalesce(st1.membership_start_date, st2.membership_end_date) as change_date, transitions.event
        FROM status_rnk st1
        FULL JOIN status_rnk st2 ON st1.customer_id = st2.customer_id
             AND st1.rnk = st2.rnk + 1
        inner join transitions on coalesce(st1.membership_status, 'Non-member') = transitions.status_to AND
        coalesce(st2.membership_status, 'Non-member') = transitions.status_from)
WHERE event != 'Non-member'
order by 1,2
```
