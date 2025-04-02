<!-- <DateRange
name="date_range"
data={dates}
dates=Date
/> -->

<!-- -- ```sql dates
-- select distinct Date from Supabase.machine
-- order by date asc
-- ``` -->
<div class="w-[100%] mx-auto">

  ```sql speed
  SELECT
    speed_bpm,
    LEFT(mc, 2) AS furnace,
    ROUND((SUM(pack_quantity)::numeric / NULLIF(SUM(output_quantity), 0)) * 100, 2) AS avg_pack_eff_percent
  FROM
    Supabase.machine
  WHERE
    job_name NOT LIKE 'MC DRAINING'
    AND pack_quantity IS NOT NULL
    AND output_quantity IS NOT NULL
    AND LEFT(mc, 2) IN ${inputs.selected_furnace.value}
  GROUP BY
    speed_bpm, furnace
  ORDER BY
    speed_bpm;

  ```
  <div class="w-full text-xl font-normal leading-snug flex items-center justify-between h-14">
    <!-- Left Side: Title + Info -->
    <div class="flex items-center gap-2">
      Speed BPM vs Efficiency by Furnace
      <Info
        description="Does not include 'MC DRAINING' or rows with missing values."
        color="red"
      />
    </div>

    <!-- Right Side: Dropdown -->
    <Dropdown name="selected_furnace" title="Select Furnace" defaultValue={["F1", "F2"]} multiple={true}>
      <DropdownOption valueLabel="F1" value="F1" />
      <DropdownOption valueLabel="F2" value="F2" />
    </Dropdown>
  </div>

  <LineChart
    step={false}
    data={speed}
    xAxisTitle={true}
    yAxisTitle="Efficiency"
    x="speed_bpm"
    y="avg_pack_eff_percent"
    markers={true}
    chartAreaHeight={150}
    sort={true}
    series="furnace"
    subtitle="Shows how packing efficiency changes at different machine speeds, for each furnace."
    colorPalette={['#cb5752', '#c88a87', '#acdba1']}
  >
    <!-- ✅ Slot for custom title bar -->
    <div slot="title" class="w-full flex justify-between items-center">
      <!-- Left Title -->
      <div class="text-xl font-normal">
        Speed BPM vs Efficiency by Furnace
      </div>

      <!-- Right: Info + Dropdown -->
      <div class="flex items-center gap-3">
        <Info
          description="Does not include 'MC DRAINING' or rows with missing values."
          color="red"
        />
        <Dropdown name="selected_furnace" title="Select Furnace" defaultValue={["F1", "F2"]} multiple={true}>
          <DropdownOption valueLabel="F1" value="F1" />
          <DropdownOption valueLabel="F2" value="F2" />
        </Dropdown>
      </div>
    </div>
  </LineChart>



  ```sql job_trends1
  SELECT
    job_name,
    LEFT(mc, 2) AS furnace,
    ROUND((SUM(pack_quantity)::numeric / NULLIF(SUM(output_quantity), 0)) * 100, 2) AS avg_pack_eff_percent
  FROM
    Supabase.machine
  WHERE
    job_name LIKE 'JG%'
    AND pack_quantity IS NOT NULL
    AND output_quantity IS NOT NULL
    AND LEFT(mc, 2) IN ${inputs.selected_furnace.value}
  GROUP BY
    job_name, furnace
  ORDER BY
    furnace, job_name ASC;

  ```

  ```sql job_trends2
  SELECT
    job_name,
    LEFT(mc, 2) AS furnace,
    ROUND(AVG(speed_bpm)::numeric, 2) AS avg_speed_bpm
  FROM
    Supabase.machine
  WHERE
    job_name LIKE 'JG%'
    AND act_pack_eff_percent IS NOT NULL
    AND furnace IN ${inputs.selected_furnace.value}
  GROUP BY
    job_name, furnace
  ORDER BY
    furnace, job_name asc;

  ```

  ```sql helper7
  WITH job_last_entry AS (
    SELECT DISTINCT ON (job_name)
      job_name,
      date,
      mc,
      total_pack_qty
    FROM Supabase.machine
    WHERE job_name LIKE 'JG%' AND total_pack_qty IS NOT NULL
    ORDER BY job_name, date DESC
  ),
  matched_rows_for_avg AS (
    SELECT
      jle.job_name AS target_job,
      sm.speed_bpm,
      sm.pack_quantity,
      sm.output_quantity,
      jle.total_pack_qty
    FROM job_last_entry jle
    JOIN Supabase.machine sm
      ON sm.date = jle.date
      AND sm.mc = jle.mc
      AND sm.total_pack_qty = jle.total_pack_qty
  ),
  aggregated_final AS (
    SELECT
      target_job AS job_name,
      ROUND(AVG(speed_bpm)::numeric, 2) AS avg_speed_bpm,
      ROUND((SUM(pack_quantity)::numeric / NULLIF(SUM(output_quantity), 0)) * 100, 2) AS avg_pack_eff_percent,
      MAX(total_pack_qty) AS total_pack_qty
    FROM matched_rows_for_avg
    WHERE speed_bpm IS NOT NULL
      AND pack_quantity IS NOT NULL
      AND output_quantity IS NOT NULL
    GROUP BY target_job
  )

  SELECT *
  FROM aggregated_final
  ORDER BY job_name;

  ```

  ```sql helper8
  WITH job_signatures AS (
    SELECT
      job_name,
      date,
      mc,
      total_pack_qty
    FROM Supabase.machine
    WHERE job_name LIKE 'JG%' AND total_pack_qty IS NOT NULL
  ),
  job_mapped_rows AS (
    SELECT
      js.job_name AS true_job_name,
      sm.speed_bpm,
      sm.pack_quantity,
      sm.output_quantity,
      sm.pass_quantity
    FROM job_signatures js
    JOIN Supabase.machine sm
      ON js.date = sm.date
      AND js.mc = sm.mc
      AND js.total_pack_qty = sm.total_pack_qty
    WHERE sm.speed_bpm IS NOT NULL
      AND sm.pack_quantity IS NOT NULL
      AND sm.output_quantity IS NOT NULL
      AND sm.pass_quantity IS NOT NULL
  ),
  grouped_by_speed AS (
    SELECT
      true_job_name AS job_name,
      speed_bpm,
      ROUND((SUM(pack_quantity)::numeric / NULLIF(SUM(output_quantity), 0)) * 100, 2) AS calculated_efficiency,
      SUM(pass_quantity) AS total_pack_qty
    FROM job_mapped_rows
    GROUP BY true_job_name, speed_bpm
  )

  SELECT *
  FROM grouped_by_speed
  ORDER BY job_name, speed_bpm;

  ```


  <Grid cols={2} gapSize=lg>
    <div>
      <div class="text-xl font-normal leading-snug flex items-center gap-2 h-14">
      Job-Level Summary (Averaged Across All Speeds)
      </div>

      <span style="font-size: 0.875rem; color: #666;">
        Summarizes each job by calculating its average speed and efficiency, using the job’s final output record.
      </span>

      <DataTable data={helper7} wrapTitles={true} rowNumbers={true}>
        <Column id="job_name" align="center" />
        <Column id="avg_speed_bpm" contentType="colorscale" colorScale="#ce5050" align="center" />
        <Column id="avg_pack_eff_percent" contentType="colorscale" colorScale="#a85ab8" align="center" title="Avg Efficiency" />
        <Column id="total_pack_qty" contentType="colorscale" colorScale="#e3af05" align="center" />
      </DataTable>

    </div>

    <Group>
      <div class="text-xl font-normal leading-snug flex items-center gap-2 h-14">
      Job-Level Breakdown (By Individual Speeds)
      </div>

      <span style="font-size: 0.875rem; color: #666;">
        Breaks down each job by all the speeds it ran at, showing efficiency and total output for each speed.
      </span>

      <DataTable data={helper8} wrapTitles={true} rowNumbers={false}>
          <Column id="job_name" align="center" />
          <Column id="speed_bpm" contentType="colorscale" colorScale="#ce5050" align="center" />
          <Column id="calculated_efficiency" contentType="colorscale" colorScale="#a85ab8" align="center" title="Avg Efficiency" />
          <Column id="total_pack_qty" contentType="colorscale" colorScale="#e3af05" align="center" />
      </DataTable>
    </Group>
  </Grid>
</div>

<!-- <ScatterPlot
  data={job_speed_summary}
  x="speed_bpm"
  y="calculated_efficiency"
  series="job_name"
  subtitle="Efficiency vs Speed per Job"
/>
<LineChart
  data={job_speed_summary}
  x="speed_bpm"
  y="calculated_efficiency"
  series="job_name"
  markers={true}
  subtitle="Efficiency at Different Speeds"
/>
<BarChart
  data={job_speed_summary}
  x="speed_bpm"
  y="total_pack_qty"
  series="job_name"
  subtitle="Total Output by Speed per Job"
/> -->
