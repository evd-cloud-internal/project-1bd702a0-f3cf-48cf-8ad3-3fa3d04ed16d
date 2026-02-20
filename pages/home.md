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

# North Star KPI trends
Get a clear view of growth trends of active users from partners.

- See trends in Active users, new parents, conversion and retention
- Drill down to see what is driving the growth


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

{% row  %}

{% big_value
    text_size="3xl"
    title="New Parents"
    data="new_parents_30d"
    info="New Parent Last 30 days"
    value="sum(new_parent_profiles)"
    fmt="num0"
/%}

{% big_value
    text_size="3xl"
    title="Parent Conversion Rate"
    data="conversion_rate_30d"
    value="sum(converted_30d) / sum(signups)"
    fmt="pct0"
    info="Weighted average: total conversions / total signups over last 30 days of completed periods"
/%}

{% big_value
    text_size="3xl"
    title="M1 Retention"
    data="m1_retention_complete"
    value="sum(active_users) / sum(cohort_size)"
    fmt="pct0"
    info="Month 1 retention rate across all completed cohort periods"
/%}

{% /row %}



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

```sql active_children_last_point
SELECT
    period_end_date,
    active_children,
    toString(active_children) as label
FROM (
    SELECT
        period_end_date,
        sum(active_children) as active_children
    FROM active_children
    WHERE period_type = UPPER({{date_grain}})
    GROUP BY period_end_date
    ORDER BY period_end_date DESC
    LIMIT 1
)
```

```sql active_users_last_points
WITH ranked AS (
    SELECT
        period_end_date,
        bank_identifier,
        sum(active_children) as active_children,
        ROW_NUMBER() OVER (PARTITION BY bank_identifier ORDER BY period_end_date DESC) as rn
    FROM active_children
    WHERE period_type = UPPER({{date_grain}})
    GROUP BY period_end_date, bank_identifier
)
SELECT
    period_end_date,
    bank_identifier,
    active_children,
    toString(active_children) as label,
    CASE bank_identifier
        WHEN 'abn-amro-nl' THEN 'abn'
        WHEN 'nordea-se' THEN 'SE'
        WHEN 'nordea-no' THEN 'NO'
        WHEN 'nordea-dk' THEN 'DK'
        WHEN 'icabanken-se' THEN 'ica'
        WHEN 'lansforsakringar-ostgota-se' THEN 'LF'
        ELSE bank_identifier
    END as short_name
FROM ranked
WHERE rn = 1
ORDER BY bank_identifier
```

```sql abn_last_point
SELECT * FROM {{active_users_last_points}}
WHERE bank_identifier = 'abn-amro-nl'
```

```sql nordea_dk_last
SELECT * FROM {{active_users_last_points}}
WHERE bank_identifier = 'nordea-dk'
```

```sql nordea_no_last
SELECT * FROM {{active_users_last_points}}
WHERE bank_identifier = 'nordea-no'
```

```sql nordea_se_last
SELECT * FROM {{active_users_last_points}}
WHERE bank_identifier = 'nordea-se'
```

```sql ica_last
SELECT * FROM {{active_users_last_points}}
WHERE bank_identifier = 'icabanken-se'
```

```sql lf_last
SELECT * FROM {{active_users_last_points}}
WHERE bank_identifier = 'lansforsakringar-ostgota-se'
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


{% accordion variant="well" %}
    {% accordion_item
        title="Per Bank KPIs"
        icon="trending-up"        
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
        fmt="pct0"
        viz="color"
    /%}
    {% measure
        value="sum(m1_active_users) / sum(m1_cohort_size) as m1_retention"
        title="M1 Retention"
        fmt="pct0"
        viz="color"
    /%}

{% /table %}
    {% /accordion_item %}
{% /accordion %}

# Active users

{% line_chart
    data="active_children"
    title="Monthly Active Children"
    subtitle="Children active in the last 30 days"
    x="period_end_date"
    y="sum(active_children)"
    y_fmt="num0"
    y_axis_options={
        title_position="side"
        labels=true
    }
    
    date_grain={{date_grain}}
    where="period_type = UPPER({{date_grain}})"
    date_range={
        range={{time_range}}
        date="period_end_date"
    }
    chart_options={
        top_padding=20
    }
%}
    {% reference_point
        data="active_children_last_point"
        x="period_end_date"
        y="active_children"
        label="label"
        color="#75C6FF"
        label_options={
            position="top"
            color="#75C6FF"
        }
        symbol_options={
            shape="circle"
            size=10
            color="#75C6FF"
        }
    /%}
{% /line_chart %}

{% row  %}

{% line_chart
    data="active_children"
    x="period_end_date"
    y="sum(active_children)"
    title="ABN AMRO - Active users"
    subtitle="ABN AMRO Children active in the last 30 days"
    where="bank_identifier = 'abn-amro-nl' AND period_type = UPPER({{date_grain}})"
    y_fmt="num2k"
    date_grain={{date_grain}}
    date_range={
        date="period_end_date"
        range="{{time_range}}"
    }
%}
    {% reference_point
        data="abn_last_point"
        x="period_end_date"
        y="active_children"
        label="label"
        color="#75C6FF"
        symbol_options={
            shape="circle"
            size=8
            color="#75C6FF"
        }
        label_options={
            position="left"
            color="#75C6FF"
            fmt="num2k"
        }
    /%}
{% /line_chart %}

{% line_chart
    data="active_children"
    x="period_end_date"
    y="sum(active_children)"
    series="bank_identifier"
      chart_options={
        series_colors={
            "nordea-no"="#FFCDB9"
            "nordea-se"="#FFDF6F"
            "nordea-dk"="#75C6FF"
        }
    }
    title="Nordea"
    where="bank_identifier IN ('nordea-se', 'nordea-no', 'nordea-dk') AND period_type = UPPER({{date_grain}})"
    y_fmt="num2k"
    date_grain={{date_grain}}
    date_range={
        date="period_end_date"
        range="{{time_range}}"
    }
%}
    {% reference_point
        data="nordea_dk_last"
        x="period_end_date"
        y="active_children"
        label="label"
        color="#75C6FF"
        symbol_options={
            shape="circle"
            size=8
            color="#75C6FF"
        }
        label_options={
            position="top"
            color="#75C6FF"
        }
    /%}
    {% reference_point
        data="nordea_dk_last"
        x="period_end_date"
        y="active_children"
        label="short_name"
        color="#75C6FF"
        symbol_options={
            size=0
        }
        label_options={
            position="left"
            color="#75C6FF"
        }
    /%}
    {% reference_point
        data="nordea_no_last"
        x="period_end_date"
        y="active_children"
        label="label"
        color="#FFCDB9"
        symbol_options={
            shape="circle"
            size=8
            color="#FFCDB9"
        }
        label_options={
            position="top"
            color="#FFCDB9"
        }
    /%}
    {% reference_point
        data="nordea_no_last"
        x="period_end_date"
        y="active_children"
        label="short_name"
        color="#FFCDB9"
        symbol_options={
            size=0
        }
        label_options={
            position="left"
            color="#FFCDB9"
        }
    /%}
    {% reference_point
        data="nordea_se_last"
        x="period_end_date"
        y="active_children"
        label="label"
        color="#FFDF6F"
        symbol_options={
            shape="circle"
            size=8
            color="#FFDF6F"
        }
        label_options={
            position="top"
            color="#FFDF6F"
        }
    /%}
    {% reference_point
        data="nordea_se_last"
        x="period_end_date"
        y="active_children"
        label="short_name"
        color="#FFDF6F"
        symbol_options={
            size=0
        }
        label_options={
            position="left"
            color="#FFDF6F"
        }
    /%}
{% /line_chart %}

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
    chart_options={
        top_padding=20
    }
%}
    {% reference_point
        data="ica_last"
        x="period_end_date"
        y="active_children"
        label="label"
        color="#75C6FF"
        symbol_options={
            shape="circle"
            size=8
            color="#75C6FF"
        }
        label_options={
            position="top"
            color="#75C6FF"
        }
    /%}
    {% reference_point
        data="ica_last"
        x="period_end_date"
        y="active_children"
        label="short_name"
        color="#75C6FF"
        symbol_options={
            size=0
        }
        label_options={
            position="bottom"
            color="#75C6FF"
        }
    /%}
    {% reference_point
        data="lf_last"
        x="period_end_date"
        y="active_children"
        label="label"
        color="#FFCDB9"
        symbol_options={
            shape="circle"
            size=8
            color="#FFCDB9"
        }
        label_options={
            position="top"
            color="#FFCDB9"
        }
    /%}
    {% reference_point
        data="lf_last"
        x="period_end_date"
        y="active_children"
        label="short_name"
        color="#FFCDB9"
        symbol_options={
            size=0
        }
        label_options={
            position="bottom"
            color="#FFCDB9"
        }
    /%}
{% /line_chart %}

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
    y_fmt="pct0"
    title="Parent Conversion Rate (30d)"
    subtitle="% of parent profiles who convert within 30 days"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="ABN Amro - New Parent Profiles"
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
    y_fmt="pct0"
    title="ABN Amro - Conversion Rate (30d)"
    where="bank_identifier = 'abn-amro-nl'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="Nordea - New Parent Profiles"
    where="bank_identifier IN ('nordea-se', 'nordea-no', 'nordea-dk')"
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
    y_fmt="pct0"
    title="Nordea - Conversion Rate (30d)"
    where="bank_identifier IN ('nordea-se', 'nordea-no', 'nordea-dk')"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

{% accordion variant="well" %}

{% accordion_item title="Nordea Branches" icon="trending-up" %}

{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="Nordea SE - New Parents"
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
    y_fmt="pct0"
    title="Nordea SE - Conversion (30d)"
    where="bank_identifier = 'nordea-se'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="Nordea NO - New Parents"
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
    y_fmt="pct0"
    title="Nordea NO - Conversion (30d)"
    where="bank_identifier = 'nordea-no'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="Nordea DK - New Parents"
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
    y_fmt="pct0"
    title="Nordea DK - Conversion (30d)"
    where="bank_identifier = 'nordea-dk'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

{% /accordion_item %}

{% /accordion %}

{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="Other Banks - New Parent Profiles"
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
    y_fmt="pct0"
    title="Other Banks - Conversion Rate (30d)"
    where="bank_identifier NOT IN ('abn-amro-nl', 'nordea-se', 'nordea-no', 'nordea-dk')"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

{% accordion variant="well" %}

{% accordion_item title="Per Bank Details" icon="trending-up" %}


{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="ICA Banken - New Parents"
    subtitle="New Parent sign-ups from ICA Banken"
    where="bank_identifier = 'icabanken-se'"
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
    y_fmt="pct0"
    title="ICA Banken - Conversion (30d)"
    where="bank_identifier = 'icabanken-se'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}


{% row %}

{% line_chart
    data="new_parents"
    x="signup_date"
    y="sum(new_parent_profiles)"
    y_fmt="num0"
    title="Länsförsäkringar Östgöta - New Parents"
    where="bank_identifier = 'lansforsakringar-ostgota-se'"
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
    y_fmt="pct0"
    title="Länsförsäkringar Östgöta - Conversion (30d)"
    where="bank_identifier = 'lansforsakringar-ostgota-se'"
    date_grain={{date_grain}}
    date_range={
        date="signup_date"
        range="{{time_range}}"
    }
/%}

{% /row %}

{% /accordion_item %}

{% /accordion %}

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







