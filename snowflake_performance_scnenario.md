# ⚙️ Snowflake Performance Diagnosis Exercise

## 📘 Scenario

You’ve inherited a view in Snowflake that support a core business dashboard used in ThoughtSpot `V_USER_ENGAGEMENT_SUMMARY`. The dashboard has recently become sluggish, with some queries taking over 60 seconds to run.

You’ve been asked to investigate and propose a plan to refactor these views for **performance and maintainability**. Initial investigation suggests that there is another view, `V_SESSION_BASE_METRICS` that is referenced within the SQL code.

---

## 🧠 Your Task

Using the materials below:
1. Identify **any performance issues** in the current view logic.
2. Propose **concrete refactors** or architecture changes to improve performance.
3. If time allows, suggest **preventative practices** or process improvements to avoid similar issues in the future.

✅ Submission Tips
- Focus on _why_ the view is slow, not just what it’s doing.
- Propose both quick refactors and scalable architectural changes.
- Bonus points for thinking like a Snowflake performance engineer.

Estimated time: **15 minutes**

---

## 🧾 SQL Code

### `V_USER_ENGAGEMENT_SUMMARY`

```sql
CREATE OR REPLACE VIEW ANALYTICS.V_USER_ENGAGEMENT_SUMMARY AS
WITH session_click_counts AS (
    SELECT 
        se.session_id,
        COUNT(*) AS click_count
    FROM prod.session_events se
    WHERE se.event_type = 'click'
    GROUP BY se.session_id
)
SELECT 
    sbm.user_id,
    AVG(sbm.session_duration) AS avg_session_duration,
    SUM(sbm.total_events) AS total_events,
    SUM(scc.click_count) AS total_clicks,
    sbm.region
FROM ANALYTICS.V_SESSION_BASE_METRICS sbm
LEFT JOIN session_click_counts scc ON sbm.session_id = scc.session_id
WHERE sbm.account_type = 'paid'
GROUP BY sbm.user_id, sbm.region;
```

### `V_SESSION_BASE_METRICS`

```sql
CREATE OR REPLACE VIEW ANALYTICS.V_SESSION_BASE_METRICS AS
WITH user_session_data AS (
    SELECT 
        s.session_id,
        s.user_id,
        s.start_time,
        s.end_time,
        u.region,
        u.account_type
    FROM prod.user_sessions s
    JOIN prod.users u ON s.user_id = u.id
),
session_events_summary AS (
    SELECT 
        se.session_id,
        COUNT(*) AS total_events
    FROM prod.session_events se
    GROUP BY se.session_id
)
SELECT 
    usd.session_id,
    usd.user_id,
    DATEDIFF('second', usd.start_time, usd.end_time) AS session_duration,
    ses.total_events,
    usd.region,
    usd.account_type
FROM user_session_data usd
JOIN session_events_summary ses ON usd.session_id = ses.session_id;
```


---
##  Entity-Relationship Diagram (ERD)

```
+------------------+     +-------------------+
|   users (u)      |     |  session_events   |
|------------------|     |-------------------|
| id (PK)          |     | session_id (FK)   |
| region           |     | event_type        |
| account_type     |     | timestamp         |
+------------------+     +-------------------+
           |                        ^
           |                        |
           v                        |
+---------------------+            |
|  user_sessions (s)  |------------+
|---------------------|
| user_id (FK)        |
| session_id (PK)     |
| start_time          |
| end_time            |
+---------------------+
```

---
## Query Execution Plan

```yaml
┌─────────────────────────────────────────────────────────────┐
│ Node 1: TableScan                                            │
│ Source: PROD.USER_SESSIONS                                  │
│ Partitions Scanned: 1000 of 1000                            │
│ Bytes Scanned: 2.4 GB                                       │
│ Bytes Spilled to Disk: 0 B                                  │
│ % Total Time: 20%                                            │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ Node 2: TableScan                                            │
│ Source: PROD.SESSION_EVENTS                                 │
│ Bytes Scanned: 1.8 GB                                       │
│ Bytes Spilled to Disk: 32 MB                                │
│ % Total Time: 18%                                            │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ Node 3: Join                                                 │
│ Type: HASH JOIN                                              │
│ Distributed Join Type: BROADCAST                            │
│ Join Key: USER_ID                                           │
│ Bytes Spilled to Local Storage: 64 MB                        │
│ Bytes Spilled to Remote Storage: 0 B                         │
│ % Total Time: 12%                                            │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ Node 4: CTE Materialization                                  │
│ CTE: session_click_counts                                    │
│ Bytes Scanned: 1.2 GB                                       │
│ Bytes Spilled to Disk: 48 MB                                 │
│ % Total Time: 8%                                             │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ Node 5: Filter                                               │
│ Condition: ACCOUNT_TYPE = 'paid'                            │
│ Records Filtered: 65%                                       │
│ % Total Time: 7%                                             │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ Node 6: Aggregation                                          │
│ Group By: USER_ID, REGION                                    │
│ Aggregates: AVG, SUM                                         │
│ Output Rows: ~2.1 million                                    │
│ Bytes Spilled to Disk: 128 MB                                │
│ % Total Time: 25%                                            │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ Node 7: Result                                               │
│ Output: V_USER_ENGAGEMENT_SUMMARY                            │
│ Result Cache Used: No                                       │
│ Bytes Returned to Client: 140 MB                             │
│ Network I/O: 40 MB                                           │
│ % Total Time: 10%                                            │
└─────────────────────────────────────────────────────────────┘
```
