<DateRange 
name="date_range"
data={dates}
dates=Date
/>

```sql dates
select distinct Date from Supabase.defect
order by date asc
```

```sql query_name
SELECT
    date,
    LEFT(mc, 2) AS furnace,
    COUNT(defect_type) AS total_defects
FROM (
    SELECT date,mc, defect1 AS defect_type FROM Supabase.defect
    UNION all
    SELECT date, mc, defect2 FROM Supabase.defect
    UNION all
    SELECT date, mc, defect3 FROM Supabase.defect
) AS Defects
WHERE defect_type IS NOT NULL
and Date BETWEEN
    COALESCE(NULLIF('${inputs.date_range.start}', ''), (SELECT MIN(Date) FROM Supabase.defect))
        AND
        COALESCE(NULLIF('${inputs.date_range.end}', ''), (SELECT MAX(Date) FROM Supabase.defect))
        and furnace in ${inputs.selected_furnace.value}
GROUP BY Date, furnace
ORDER BY Date;

```

```sql query_name1
SELECT
    date,
    COUNT(defect_type) AS total_defects
FROM (
    SELECT date, defect1 AS defect_type FROM Supabase.defect
    UNION all
    SELECT date, defect2 FROM Supabase.defect
    UNION all
    SELECT date, defect3 FROM Supabase.defect
) AS Defects
WHERE defect_type IS NOT NULL
and Date BETWEEN
    COALESCE(NULLIF('${inputs.date_range.start}', ''), (SELECT MIN(Date) FROM Supabase.defect))
        AND
        COALESCE(NULLIF('${inputs.date_range.end}', ''), (SELECT MAX(Date) FROM Supabase.defect))
GROUP BY Date
ORDER BY Date;

```

## Number of Defects Per Furnace

<CalendarHeatmap
data={query_name1}
date=date
value=total_defects
subtitle="Daily defects"
colorScale={[
['rgb(99, 68, 63)', 'rgb(218,66,41)'],
['rgb(254,234,159)', 'rgb(254,234,159)']
]}
/>

<Dropdown name="selected_furnace" title="Select Furnace" defaultValue={["F1", "F2"]} multiple=true>
<DropdownOption valueLabel="F1" value="F1" />
<DropdownOption valueLabel="F2" value="F2" />
</Dropdown>

<LineChart step = true
data={query_name}  
x=date
y=total_defects
series = furnace
subtitle = "Another way of looking at it.."
markers=true
chartAreaHeight=250
sort=true
colorPalette={[
'#cb5752',
'#c88a87',
'#acdba1',
]}
/>
<Dropdown name="selected_machine" title="Select Machine" defaultValue={["F11","F13","F21","F22","F23"]} multiple=true>
<DropdownOption valueLabel="F11" value="F11" />
<DropdownOption valueLabel="F13" value="F13" />
<DropdownOption valueLabel="F21" value="F21" />
<DropdownOption valueLabel="F22" value="F22" />
<DropdownOption valueLabel="F23" value="F23" />
</Dropdown>

## Top 10 Defects

```sql testing2
SELECT defect_type, COUNT(*) AS defect_count
FROM (
    SELECT date, defect1 AS defect_type, mc FROM Supabase.defect
    UNION ALL
    SELECT date, defect2, mc FROM Supabase.defect
    UNION ALL
    SELECT date, defect3, mc FROM Supabase.defect
) AS AllDefects
WHERE defect_type IS NOT NULL
AND mc IN ${inputs.selected_machine.value}  -- Machine filter
AND date BETWEEN
    COALESCE(NULLIF('${inputs.date_range.start}', ''), (SELECT MIN(date) FROM Supabase.defect))
    AND
    COALESCE(NULLIF('${inputs.date_range.end}', ''), (SELECT MAX(date) FROM Supabase.defect))
GROUP BY defect_type
ORDER BY defect_count DESC
LIMIT 10;  -- Get top 10 defects

```

<BarChart
data={testing2}
x=defect_type
y=defect_count
swapXY= true
/>

## Action Plans

```sql testing3
SELECT mc, plans_type, COUNT(*) AS plans_count
FROM (
    SELECT mc, plan1 AS plans_type FROM Supabase.defect
    UNION ALL
    SELECT mc, plan2 FROM Supabase.defect
    UNION ALL
    SELECT mc, plan3 FROM Supabase.defect
) AS AllPlans
WHERE plans_type IS NOT NULL
GROUP BY mc, plans_type
ORDER BY plans_type DESC;

```

<BarChart
  data={testing3}
  x=plans_type
  y=plans_count
  swapXY= true
  series = mc
  groupMode = "stack"
/>

```sql defects
SELECT
    Date,
    defect_type,
    COUNT(defect_type) AS defect_count  -- Count of each unique defect per date
FROM (
    SELECT Date, defect1 AS defect_type FROM Supabase.defect
    UNION ALL
    SELECT Date, defect2 FROM Supabase.defect
    UNION ALL
    SELECT Date, defect3 FROM Supabase.defect
) AS Defects
WHERE defect_type IS NOT NULL
and Date BETWEEN
    COALESCE(NULLIF('${inputs.date_range.start}', ''), (SELECT MIN(Date) FROM Supabase.defect))
        AND
        COALESCE(NULLIF('${inputs.date_range.end}', ''), (SELECT MAX(Date) FROM Supabase.defect))
GROUP BY Date, defect_type
HAVING COUNT(defect_type) > 0  -- Only keep records where defect_count is greater than 0
ORDER BY Date, defect_count asc;

```

<BarChart
data={defects}
x="date"
y="defect_count"
series="defect_type"
chartAreaHeight= 300
/>

```sql testing4
WITH DefectData AS (
    SELECT date, defect1 AS defect_type FROM Supabase.defect
    UNION ALL
    SELECT date, defect2 FROM Supabase.defect
    UNION ALL
    SELECT date, defect3 FROM Supabase.defect
)
SELECT date, defect_type, COUNT(*) AS defect_count
FROM DefectData
WHERE defect_type IS NOT NULL
AND date BETWEEN
    COALESCE(NULLIF('${inputs.date_range.start}', ''), (SELECT MIN(date) FROM Supabase.defect))
    AND
    COALESCE(NULLIF('${inputs.date_range.end}', ''), (SELECT MAX(date) FROM Supabase.defect))
GROUP BY date, defect_type
ORDER BY date, defect_count DESC;

```

<LineChart 
  data={testing4} 
  x="date" 
  y="defect_count" 
  series="defect_type"
  showTooltipOnHover={true} 
  chartAreaHeight={300}
  step=true
/>
