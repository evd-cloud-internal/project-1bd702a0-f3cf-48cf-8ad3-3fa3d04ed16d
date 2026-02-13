---
name: Home
assetId: fac58792-1273-4b3a-affa-4d01e083acc0
type: page
---

# Company Dashboard

{% date_grain_selector
    title="Choose a grain"
    id="date_grain"
    preset_values=["day", "week", "month"]
    default_value="week"
/%}

{% range_calendar
    id="time_range"
    default_range="last 3 months"
    title="Choose a range"
/%}

```sql current_active_children
WITH max_date AS (
  SELECT MAX(period_end_date) max_date FROM active_children
)
SELECT 
    *
FROM active_children
WHERE period_end_date = (SELECT max_date FROM max_date)
AND period_type = 'day'
```

{% big_value
    text_size="2xl"
    title="Monthly Active Chilcren"
    data="current_active_children"
    value="sum(active_children)"
/%}



```sql mau_qaf_trend
SELECT
    c.period_identifier,
    c.active_children as monthly_active_children,
    f.active_families as quarterly_active_families
FROM (
    SELECT period_identifier, sum(active_children) as active_children
    FROM active_children
    WHERE period_type = {{date_grain}}
    GROUP BY period_identifier
) c
LEFT JOIN (
    SELECT period_identifier, sum(active_families) as active_families
    FROM active_families
    WHERE period_type = {{date_grain}}
    GROUP BY period_identifier
) f ON c.period_identifier = f.period_identifier
ORDER BY c.period_identifier
```

{% line_chart
    data="mau_qaf_trend"
    x="period_identifier"
    y="max(monthly_active_children)"
    y2="max(quarterly_active_families)"
    data_labels={
        position="above"
    }
    y2_axis_options={
        labels=false
        gridlines=false
        title=""
    }
    y_axis_options={
        ticks=false
        gridlines=false
        title=""
    }
    title="Growth Trend"
    subtitle="Monthly Active Children vs Quarterly Active Families"
/%}

```sql new_parents
SELECT
    *
FROM new_partner_parent_profiles
```

```sql conversion_rate
SELECT
    *
FROM parent_conversion
```

```sql cohort_retention
SELECT
    cohort_date,
    month_number,
    bank_identifier,
    cohort_size,
    active_users,
    CASE WHEN is_period_complete THEN active_users END AS active_users_confirmed,
    CASE WHEN is_period_complete THEN cohort_size END AS cohort_size_confirmed
FROM month_n_retention
ORDER BY cohort_date
```

{% row  %}



{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="New Parents from partners"
    subtitle="Signups over time"
    date_grain={{date_grain}}
/%}



{% line_chart
    data="conversion_rate"
    x="signup_date"
    y="sum(converted_30d) / sum(signups)"
    title="Parent Conversion Rate (30d)"
    subtitle="% of signups who convert within 30 days"
    date_grain="week"
/%}

{% /row %}

```sql available_periods
SELECT DISTINCT month_number
FROM month_n_retention
WHERE month_number BETWEEN 1 AND 3
ORDER BY month_number
```

{% row  %}

{% repeat id="month_num" data="available_periods" column="month_number" %}

{% combo_chart
    data="cohort_retention"
    x="cohort_date"
    where="month_number = {{ month_num }}"
    title="M{{month_num}} Retention"
    y_fmt="pct1"
    height=250
%}
    {% line y="sum(active_users) / sum(cohort_size)"
           options={type="dashed" color="#3b82f6" opacity=0.4} /%}
    {% line y="sum(active_users_confirmed) / sum(cohort_size_confirmed)"
           options={type="solid" color="#3b82f6"} /%}
{% /combo_chart %}


{% /repeat %}

{% /row %}





