---
name: Home
assetId: fac58792-1273-4b3a-affa-4d01e083acc0
type: page
---

# Partner Growth trends
Get a clear view of growth trends of active users from partners.


{% date_grain_selector
    title="Choose a grain"
    id="date_grain"
    preset_values=["day", "week", "month"]
    default_value="week"
/%}

{% range_calendar
    id="time_range"
    default_range="last 6 months"
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


```sql active_users
SELECT
    c.period_end_date,
    c.active_children,
    f.active_families
FROM (
    SELECT period_end_date, sum(active_children) as active_children
    FROM active_children
    WHERE period_type = {{date_grain}}
    GROUP BY period_end_date
) c
LEFT JOIN (
    SELECT period_end_date, sum(active_families) as active_families
    FROM active_families
    WHERE period_type = {{date_grain}}
    GROUP BY period_end_date
) f ON c.period_end_date = f.period_end_date
ORDER BY c.period_end_date
```

{% combo_chart
    data="active_users"
    x="period_end_date"
    title="Active users"
    date_range={
        date="period_end_date"
        range="{{time_range}}"
    }
%}

{% line
    y="sum(active_children)"
    options={
        width=3
    }
    data_labels={
        position="left"
    }
/%}
{% line
    y="sum(active_families)"
    options={
        opacity=0.4
        markers={
            shape="none"
        }
        type="dashed"
    }
/%}

{% /combo_chart %}

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
```

## New partner parents



{% row  %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="New Parent Profiles from partners"
    subtitle="Signups over time"
    date_grain={{date_grain}}
    info="New Parent Profiles from parters"
/%}



{% /row %}



## New Parent Conversion rate

{% row  %}

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

## Cohort retention trends

{% row  %}

{% repeat id="month_num" data="available_periods" column="month_number" %}

{% combo_chart
    data="cohort_retention"
    x="cohort_date"
    where="month_number = {{ month_num }}"
    title="M{{month_num}} Retention"
    y_fmt="pct0"
    date_grain={{date_grain}}
    date_range={
        date="cohort_date"
        range="{{time_range}}"
    }
    
%}
    {% line y="sum(active_users) / sum(cohort_size)"
           options={type="dashed"
           } 
           /%}
    {% line y="sum(active_users_confirmed) / sum(cohort_size_confirmed)"
           options={type="solid"} /%}
{% /combo_chart %}


{% /repeat %}

{% /row %}





