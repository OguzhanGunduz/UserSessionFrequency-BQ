# User Session Count 

*You can find the SQL query and its explanation below.

```SQL
-- CTE to prepare session-related information
WITH prep AS (
  SELECT
    user_pseudo_id,
    -- Extract session_id from the 'session_start' event parameters
    (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'session_start' AND key = 'ga_session_id') AS session_id,
    -- Extract session_number from the 'session_start' event parameters
    (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'session_start' AND key = 'ga_session_number') AS session_number,
  FROM
    xxx.analytics_xxx.events_20210601 --- "xxx" part, you can add your id
  -- Filter out rows where ga_session_id is not null
  WHERE
    (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'session_start' AND key = 'ga_session_id') IS NOT NULL
  -- Order by session_number
  ORDER BY
    session_number
),
```

```SQL
-- CTE to prepare page_view-related information
prep_pageview AS (
  SELECT
    user_pseudo_id,
    -- Extract session_id from the 'page_view' event parameters
    (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'page_view' AND key = 'ga_session_id') AS session_id,
    -- Assign 1 to the page_view if the event is a 'page_view', otherwise 0
    CASE WHEN event_name = 'page_view' THEN 1 ELSE 0 END AS pageview
  FROM
    xxx.analytics_xxx.events_20210601  --- "xxx" part, you can add your id
  -- Filter rows where the event_name is 'page_view'
  WHERE
    event_name = 'page_view'
),
```

```SQL
-- CTE to prepare session counts
prep_session_count AS (
  SELECT
    user_pseudo_id,
    session_id,
    -- Use the FIRST_VALUE window function to get the max session number for each user
    FIRST_VALUE(session_number) OVER (PARTITION BY user_pseudo_id ORDER BY session_number DESC) AS max_session_number
  FROM
    prep
  -- Order by max_session_number and user_pseudo_id
  ORDER BY
    max_session_number, user_pseudo_id
)
```

```SQL
-- Main query to get aggregated results
SELECT
  prep_session_count.max_session_number,
  -- Count distinct concatenated user_pseudo_id and session_id for unique sessions
  COUNT(DISTINCT CONCAT(prep_session_count.user_pseudo_id, prep_session_count.session_id)) AS sessions,
  -- Count pageviews
  COUNT(prep_pageview.pageview) AS total_pageviews
FROM
  prep_session_count
-- Join with prep_pageview based on user_pseudo_id
JOIN
  prep_pageview ON prep_session_count.user_pseudo_id = prep_pageview.user_pseudo_id
-- Filter by a specific user_pseudo_id
WHERE
  prep_session_count.user_pseudo_id = "1003822581.1612175456"
GROUP BY
  prep_session_count.max_session_number
-- Filter out null max_session_numbers
HAVING
  prep_session_count.max_session_number IS NOT NULL
-- Order by max_session_number
ORDER BY
  prep_session_count.max_session_number;
```
