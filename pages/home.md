---
name: Home
assetId: fac58792-1273-4b3a-affa-4d01e083acc0
type: page
---

{% image
  url="https://gimi-org.github.io/gimi-analytics/images/white-logo.png"
  description="Gimi Logo"
  max_width=150
/%}

# Partner Growth KPI trends
Get a clear view of growth trends of active users from partners.


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
AND period_type = 'DAY'
```

```sql new_parents_30d
SELECT
    signup_date,
    new_parent_profiles
FROM new_partner_parent_profiles
WHERE signup_date >= today() - 30
ORDER BY signup_date
```

```sql conversion_rate_30d
SELECT
    signup_date,
    signups,
    converted_30d
FROM parent_conversion
WHERE is_30d_complete = true
AND signup_date > (SELECT max(signup_date) - 30 FROM parent_conversion WHERE is_30d_complete = true)
ORDER BY signup_date
```

```sql m1_retention_complete
SELECT
    cohort_date,
    active_users,
    cohort_size
FROM month_n_retention
WHERE month_number = 1
AND is_period_complete = true
ORDER BY cohort_date
```

# KPI summary

{% big_value
    text_size="3xl"
    title="Monthly Active Children"
    data="current_active_children"
    value="sum(active_children)"
    fmt="num2k"
/%}

{% big_value
    text_size="3xl"
    title="New Parents"
    data="new_parents_30d"
    info="New Parent Last 30 days"
    value="sum(new_parent_profiles)"
    fmt="num0"
    sparkline={
        type="bar"
        x="signup_date"
    }
/%}

{% big_value
    text_size="3xl"
    title="Parent Conversion Rate"
    data="conversion_rate_30d"
    value="sum(converted_30d) / sum(signups)"
    fmt="pct1"
    info="Weighted average: total conversions / total signups over last 30 days of completed periods"
    sparkline={
        type="line"
        x="signup_date"
    }
/%}

{% big_value
    text_size="3xl"
    title="M1 Retention (Completed)"
    data="m1_retention_complete"
    value="sum(active_users) / sum(cohort_size)"
    fmt="pct1"
    sparkline={
        type="line"
        x="cohort_date"
        date_grain="month"
    }
    info="Month 1 retention rate across all completed cohort periods"
/%}

```sql active_users
SELECT
    c.period_end_date,
    bank_identifier,
    c.active_children,
    f.active_families
FROM (
    SELECT period_end_date, 
    sum(active_children) as active_children, 
    bank_identifier
    FROM active_children
    WHERE period_type = UPPER({{date_grain}})
    GROUP BY period_end_date, bank_identifier
) c
LEFT JOIN (
    SELECT 
        period_end_date, 
        sum(active_families) as active_families,
        bank_identifier
    FROM active_families
    WHERE period_type = UPPER({{date_grain}})
    GROUP BY period_end_date, bank_identifier
) f ON (c.period_end_date = f.period_end_date AND c.bank_identifier = f.bank_identifier)
ORDER BY c.period_end_date
```

```sql bank_kpis
WITH max_date AS (
    SELECT MAX(period_end_date) as max_date FROM active_children WHERE period_type = 'DAY'
),
active AS (
    SELECT
        bank_identifier,
        sum(active_children) as active_children
    FROM active_children
    WHERE period_end_date = (SELECT max_date FROM max_date)
    AND period_type = 'DAY'
    GROUP BY bank_identifier
),
new_parents AS (
    SELECT
        bank_identifier,
        sum(new_parent_profiles) as new_parents_30d
    FROM new_partner_parent_profiles
    WHERE signup_date >= today() - 30
    GROUP BY bank_identifier
),
conversion AS (
    SELECT
        bank_identifier,
        sum(converted_30d) as converted_30d,
        sum(signups) as signups_30d
    FROM parent_conversion
    WHERE is_30d_complete = true
    AND signup_date > (SELECT max(signup_date) - 30 FROM parent_conversion WHERE is_30d_complete = true)
    GROUP BY bank_identifier
),
retention AS (
    SELECT
        bank_identifier,
        sum(active_users) as m1_active_users,
        sum(cohort_size) as m1_cohort_size
    FROM month_n_retention
    WHERE month_number = 1
    AND is_period_complete = true
    GROUP BY bank_identifier
)
SELECT
    bank_identifier,
    active_children,
    new_parents_30d,
    converted_30d,
    signups_30d,
    m1_active_users,
    m1_cohort_size
FROM active
FULL OUTER JOIN new_parents USING (bank_identifier)
FULL OUTER JOIN conversion USING (bank_identifier)
FULL OUTER JOIN retention USING (bank_identifier)
ORDER BY active_children DESC
```

{% details
    title="Per bank KPIs"
%}

{% table
    data="bank_kpis"
%}

    {% dimension
        value="bank_identifier"
        title="Bank"
    /%}
    {% measure
        value="sum(active_children)"
        title="Active Children"
        fmt="num0"
        viz="bar"
        sort="desc"
    /%}
    {% measure
        value="sum(new_parents_30d)"
        title="New Parents (30d)"
        fmt="num0"
        viz="bar"
    /%}
    {% measure
        value="sum(converted_30d) / sum(signups_30d) as conversion_rate"
        title="Conversion Rate (30d)"
        fmt="pct1"
        viz="color"
    /%}
    {% measure
        value="sum(m1_active_users) / sum(m1_cohort_size) as m1_retention"
        title="M1 Retention"
        fmt="pct1"
        viz="color"
    /%}

{% /table %}
{% /details %}

# Active users
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
    fmt="num2k"
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
    fmt="num2k"
/%}

{% /combo_chart %}

{% row  %}

{% line_chart
    data="active_children"
    x="period_end_date"
    y="sum(active_children)"
    title="ABN Amro"
    where="bank_identifier = 'abn-amro-nl' AND period_type = UPPER({{date_grain}})"
    y_fmt="num2k"
    date_grain={{date_grain}}
    date_range={
        date="period_end_date"
        range="{{time_range}}"
    }
/%}

{% line_chart
    data="active_children"
    x="period_end_date"
    y="sum(active_children)"
    series="bank_identifier"
    title="Nordea"
    where="bank_identifier IN ('nordea-se', 'nordea-no', 'nordea-dk') AND period_type = UPPER({{date_grain}})"
    y_fmt="num2k"
    date_grain={{date_grain}}
    date_range={
        date="period_end_date"
        range="{{time_range}}"
    }
/%}

{% line_chart
    data="active_children"
    x="period_end_date"
    y="sum(active_children)"
    series="bank_identifier"
    title="Other Banks"
    where="bank_identifier NOT IN ('abn-amro-nl', 'nordea-se', 'nordea-no', 'nordea-dk') AND period_type = UPPER({{date_grain}})"
    y_fmt="num2k"
    date_grain={{date_grain}}
    date_range={
        date="period_end_date"
        range="{{time_range}}"
    }
/%}

{% /row %}



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

# Partner distribution

## Overall

{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="New Parent Profiles"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% line_chart
    data="conversion_rate"
    x="signup_date"
    y="sum(converted_30d) / sum(signups)"
    y_fmt="pct1"
    title="Parent Conversion Rate (30d)"
    subtitle="% of parent profiles who convert within 30 days"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

## ABN Amro

{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="New Parent Profiles"
    where="bank_identifier = 'abn-amro-nl'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% line_chart
    data="conversion_rate"
    x="signup_date"
    y="sum(converted_30d) / sum(signups)"
    y_fmt="pct1"
    title="Parent Conversion Rate (30d)"
    where="bank_identifier = 'abn-amro-nl'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

## Nordea

{% row  %}

{% stack  %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="New Parents"
    where="bank_identifier = 'nordea-se'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% line_chart
    data="conversion_rate"
    x="signup_date"
    y="sum(converted_30d) / sum(signups)"
    y_fmt="pct1"
    title="Conversion (30d)"
    where="bank_identifier = 'nordea-se'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /stack %}
{% /row %}

#### Nordea SE



#### Nordea NO

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="New Parents"
    where="bank_identifier = 'nordea-no'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% line_chart
    data="conversion_rate"
    x="signup_date"
    y="sum(converted_30d) / sum(signups)"
    y_fmt="pct1"
    title="Conversion (30d)"
    where="bank_identifier = 'nordea-no'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

#### Nordea DK

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="New Parents"
    where="bank_identifier = 'nordea-dk'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% line_chart
    data="conversion_rate"
    x="signup_date"
    y="sum(converted_30d) / sum(signups)"
    y_fmt="pct1"
    title="Conversion (30d)"
    where="bank_identifier = 'nordea-dk'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

## Other Banks

{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    series="bank_identifier"
    title="New Parent Profiles"
    where="bank_identifier NOT IN ('abn-amro-nl', 'nordea-se', 'nordea-no', 'nordea-dk')"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% line_chart
    data="conversion_rate"
    x="signup_date"
    y="sum(converted_30d) / sum(signups)"
    y_fmt="pct1"
    series="bank_identifier"
    title="Parent Conversion Rate (30d)"
    where="bank_identifier NOT IN ('abn-amro-nl', 'nordea-se', 'nordea-no', 'nordea-dk')"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

# Child retention

{% row %}

{% combo_chart
    data="cohort_retention"
    x="cohort_date"
    where="month_number = 1"
    title="M1 Retention"
    y_fmt="pct0"
    date_grain={{date_grain}}
    date_range={
        date="cohort_date"
        range="{{time_range}}"
    }
%}
    {% line y="sum(active_users) / sum(cohort_size)" options={type="dashed"} /%}
    {% line y="sum(active_users_confirmed) / sum(cohort_size_confirmed)" options={type="solid"} /%}
{% /combo_chart %}

{% combo_chart
    data="cohort_retention"
    x="cohort_date"
    where="month_number = 2"
    title="M2 Retention"
    y_fmt="pct0"
    date_grain={{date_grain}}
    date_range={
        date="cohort_date"
        range="{{time_range}}"
    }
%}
    {% line y="sum(active_users) / sum(cohort_size)" options={type="dashed"} /%}
    {% line y="sum(active_users_confirmed) / sum(cohort_size_confirmed)" options={type="solid"} /%}
{% /combo_chart %}

{% combo_chart
    data="cohort_retention"
    x="cohort_date"
    where="month_number = 3"
    title="M3 Retention"
    y_fmt="pct0"
    date_grain={{date_grain}}
    date_range={
        date="cohort_date"
        range="{{time_range}}"
    }

%}
    {% line y="sum(active_users) / sum(cohort_size)" options={type="dashed"} /%}
    {% line y="sum(active_users_confirmed) / sum(cohort_size_confirmed)" options={type="solid"} /%}
{% /combo_chart %}

{% /row %}







