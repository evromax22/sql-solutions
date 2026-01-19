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

# Key points
1. Basic transition rules have been updated to include an additional case: 'Non-member','Non-member','Non-member' (these cases exist in the data).
2. Use CTE for transition rules mapping
3. Use ROW_NUMBER() to sequence periods
4. Handle first/last statuses with COALESCE and FULL JOIN.
    A FULL JOIN is needed for two extreme cases.
    After FULL JOIN (st1.rnk = st2.rnk + 1) for example 115:
        
        st1 (next period)	st2 (current period)	What it represents
        rnk=2 (Paid)	rnk=1 (Free)	Transition 1→2
        rnk=3 (Non-member)	rnk=2 (Paid)	Transition 2→3
        rnk=4 (Paid)	rnk=3 (Non-member)	Transition 3→4
        NULL	rnk=4 (Paid)	Last record!
        rnk=1 (Free)	NULL	First record!

    Why NULL rows are needed:
    I. st1 = NULL, st2 = last record → transition to Non-member
    st2.membership_end_date = 2020-10-01
    st2.status = 'Paid'
    st1.status = NULL → COALESCE(NULL, 'Non-member') = 'Non-member'
    Rule: 'Paid' → 'Non-member' = 'Cancel'
    
    II. st2 = NULL, st1 = first record → start of history
    st1.membership_start_date = 2020-01-01
    st1.status = 'Free'
    st2.status = NULL → COALESCE(NULL, 'Non-member') = 'Non-member'
    Rule: 'Non-member' → 'Free' = 'WarmStart'
5. COALESCE treats NULL as 'Non-member' for edge cases. WHERE excludes 'Non-member'→'Non-member' events since they represent no actual status change.

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
