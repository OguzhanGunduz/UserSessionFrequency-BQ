# User Session Count 

* When you execute the provided query, you should see a table structured as follows:
![image](https://github.com/OguzhanGunduz/UserSessionCount-BQ/assets/89355322/c3c70e8e-647c-4125-81d9-4087c8bf424b)



## You can find the SQL query and its explanation below.

* In this query, a Common Table Expression (CTE) named "prep" is created to facilitate window functions. It retrieves user_pseudo_id, ga_session_id, and ga_session_number from the "2021-06-01" dated database, specifically focusing on sessions initiated with the 'session_start' event. Non-null values of ga_session_id are selected, and the results are ordered based on session_number.
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

* In this query, a Common Table Expression (CTE) named "prep_pageview" is crafted to facilitate window functions. It retrieves user_pseudo_id and ga_session_id from the "2021-06-01" dated database, focusing on events triggered by 'page_view'. For each 'page_view' event, a value of 1 is assigned, and the results are ordered based on the 'page_view' event.
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

* In this query, a Common Table Expression (CTE) named "prep_session_count" is created to facilitate window functions. It retrieves user_pseudo_id and session_id values from the "prep" data. If a user initiates multiple sessions on the same day, they can have multiple "session_number" values. To address this, the "session_number" values are sorted, and the first value is selected, then linked with user_pseudo_id using PARTITION. The results are further ordered by max_session_number and user_pseudo_id.
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

* Utilizing 'prep_session_count' and 'prep_pageview,' this Final Query computes the values for 'max_session_number,' 'sessions,' and 'total_pageviews.' Non-null values of 'max_session_number' are selected, and the results are ordered by 'max_session_number.' This allows us to specifically report on which session number each user reached during the specified time period.
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
GROUP BY
  prep_session_count.max_session_number
-- Filter out null max_session_numbers
HAVING
  prep_session_count.max_session_number IS NOT NULL
-- Order by max_session_number
ORDER BY
  prep_session_count.max_session_number;
```
