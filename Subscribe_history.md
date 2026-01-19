# [Multiply](https://solvit.space/coding/2997)

# Description 
Given a subscriptions table with a history of all subscription periods for each client.
Write an SQL query that converts the subscription history into a chronological list of events.

Example source data:
| customer_id | membership_start_date | membership_end_date | membership_status |
|-------------|-----------------------|---------------------|-------------------|
| 115         | 2020-01-01            | 2020-02-15          | Free              |
| 115         | 2020-02-15            | 2020-03-15          | Paid              |
| 115         | 2020-03-15            | 2020-04-01          | Non-member        |
| 115         | 2020-04-01            | 2020-10-01          | Paid              |

Basic rules for transitions:
Free → Paid: 'Convert'
Paid → Free: 'ReverseConvert'
Paid → Non-member: 'Cancel'
Free → Non-member: 'Cancel'
Non-member → Paid: 'ColdStart'
Non-member → Free: 'WarmStart'
Paid → Paid: 'Renewal'
Free → Free: 'Renewal'

Example result:
| customer_id | change_date | event      |
|-------------|-------------|------------|
| 115         | 2020-01-01  | WarmStart  |
| 115         | 2020-02-15  | Convert    |
| 115         | 2020-03-15  | Cancel     |
| 115         | 2020-04-01  | ColdStart  |
| 115         | 2020-10-01  | Cancel     |

The fields in the resulting table are: customer_id, change_date, event.
Sort the results in ascending order by customer_id, then by change_date.

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
