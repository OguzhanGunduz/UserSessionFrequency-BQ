# UserSessionCount-


WITH prep AS (
  SELECT
    user_pseudo_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'session_start' AND key = 'ga_session_id') AS session_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'session_start' AND key = 'ga_session_number') AS session_number,
  FROM
    xxx.analytics_xxx.events_20210601
  WHERE
    (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'session_start' AND key = 'ga_session_id') IS NOT NULL
  ORDER BY
    session_number
),

prep_pageview AS (
  SELECT
    user_pseudo_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE event_name = 'page_view' AND key = 'ga_session_id') AS session_id,
    CASE WHEN event_name = 'page_view' THEN 1 ELSE 0 END AS pageview
  FROM
    xxx.analytics_xxx.events_20210601
  WHERE
    event_name = 'page_view'
),

prep_session_count AS (
  SELECT
    user_pseudo_id,
    session_id,
    FIRST_VALUE(session_number) OVER (PARTITION BY user_pseudo_id ORDER BY session_number DESC) AS max_session_number
  FROM
    prep
  ORDER BY
    max_session_number, user_pseudo_id
)

SELECT
  prep_session_count.max_session_number,
  COUNT(DISTINCT CONCAT(prep_session_count.user_pseudo_id, prep_session_count.session_id)) AS sessions,
  count(prep_pageview.pageview) AS total_pageviews,
FROM
  prep_session_count
JOIN
  prep_pageview ON prep_session_count.user_pseudo_id = prep_pageview.user_pseudo_id
WHERE
prep_session_count.user_pseudo_id="1003822581.1612175456"
GROUP BY
  prep_session_count.max_session_number
HAVING
  prep_session_count.max_session_number IS NOT NULL
ORDER BY
  prep_session_count.max_session_number;
