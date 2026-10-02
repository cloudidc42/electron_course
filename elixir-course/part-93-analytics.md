# Part 93: Analytics & Reporting (Steps 991-1000)

## Step 991: Event Tracking

```elixir
defmodule MyApp.Analytics do
  use Ecto.Schema
  import Ecto.Query

  schema "events" do
    field :user_id,    :integer
    field :event_name, :string
    field :properties, :map, default: %{}
    field :session_id, :string
    field :ip_address, :string
    field :user_agent, :string

    timestamps(updated_at: false)
  end

  def track(event_name, properties \\ %{}, opts \\ []) do
    attrs = %{
      event_name: event_name,
      properties: properties,
      user_id:    Keyword.get(opts, :user_id),
      session_id: Keyword.get(opts, :session_id),
      ip_address: Keyword.get(opts, :ip_address)
    }
    
    # Async insert to avoid blocking request
    Task.start(fn ->
      %__MODULE__{} |> Ecto.Changeset.cast(attrs, Map.keys(attrs)) |> MyApp.Repo.insert()
    end)
    
    # Also send to telemetry
    :telemetry.execute([:analytics, :event], %{count: 1}, %{name: event_name})
  end

  def funnel(steps, period \\ 30) do
    cutoff = DateTime.add(DateTime.utc_now(), -period * 86_400)
    
    Enum.zip(steps, tl(steps ++ [nil]))
    |> Enum.reduce({nil, []}, fn {step, next_step}, {prev_users, acc} ->
      users = from(e in __MODULE__,
        where: e.event_name == ^step and e.inserted_at > ^cutoff,
        select: e.user_id,
        distinct: true
      ) |> MyApp.Repo.all() |> MapSet.new()
      
      users = if prev_users, do: MapSet.intersection(users, prev_users), else: users
      
      entry = %{step: step, count: MapSet.size(users), drop_off: 0}
      {users, [entry | acc]}
    end)
    |> elem(1)
    |> Enum.reverse()
    |> calculate_dropoffs()
  end

  defp calculate_dropoffs([first | rest]) do
    Enum.scan([first | rest], first, fn curr, prev ->
      drop_off = if prev.count > 0, do: (1 - curr.count / prev.count) * 100, else: 0
      Map.put(curr, :drop_off, Float.round(drop_off, 1))
    end)
  end
end
```

---

## Step 992: Dashboard Metrics

```elixir
defmodule MyApp.Dashboard.Metrics do
  import Ecto.Query

  def overview(period \\ :today) do
    {start_at, compare_at} = period_range(period)
    
    current  = compute_metrics(start_at, DateTime.utc_now())
    previous = compute_metrics(compare_at, start_at)
    
    %{
      current:  current,
      previous: previous,
      changes:  compute_changes(current, previous)
    }
  end

  defp compute_metrics(from_dt, to_dt) do
    %{
      revenue:   revenue_in_period(from_dt, to_dt),
      orders:    orders_in_period(from_dt, to_dt),
      users:     new_users_in_period(from_dt, to_dt),
      page_views: page_views_in_period(from_dt, to_dt)
    }
  end

  defp revenue_in_period(from_dt, to_dt) do
    from(o in MyApp.Order,
      where: o.status == "paid" and o.paid_at >= ^from_dt and o.paid_at < ^to_dt,
      select: coalesce(sum(o.total), 0)
    ) |> MyApp.Repo.one()
  end

  defp orders_in_period(from_dt, to_dt) do
    from(o in MyApp.Order,
      where: o.inserted_at >= ^from_dt and o.inserted_at < ^to_dt,
      select: count(o.id)
    ) |> MyApp.Repo.one()
  end

  defp new_users_in_period(from_dt, to_dt) do
    from(u in MyApp.User,
      where: u.inserted_at >= ^from_dt and u.inserted_at < ^to_dt,
      select: count(u.id)
    ) |> MyApp.Repo.one()
  end

  defp page_views_in_period(from_dt, to_dt) do
    from(e in MyApp.Analytics,
      where: e.event_name == "page_view" and e.inserted_at >= ^from_dt and e.inserted_at < ^to_dt,
      select: count(e.id)
    ) |> MyApp.Repo.one()
  end

  defp compute_changes(current, previous) do
    Map.new(current, fn {metric, value} ->
      prev_value = Map.get(previous, metric, 0)
      change = if prev_value > 0, do: (value - prev_value) / prev_value * 100, else: 0
      {metric, Float.round(change, 1)}
    end)
  end

  defp period_range(:today) do
    today = Date.utc_today()
    start = DateTime.new!(today, ~T[00:00:00])
    yesterday = Date.add(today, -1)
    compare = DateTime.new!(yesterday, ~T[00:00:00])
    {start, compare}
  end

  defp period_range(:this_week) do
    now   = DateTime.utc_now()
    start = DateTime.add(now, -7 * 86_400)
    compare = DateTime.add(now, -14 * 86_400)
    {start, compare}
  end
end
```

---

## Step 993: Time Series Data

```elixir
defmodule MyApp.TimeSeries do
  import Ecto.Query

  def revenue_over_time(period, interval) do
    {start_at, trunc_fn} = case {period, interval} do
      {:week, :day}    -> {DateTime.add(DateTime.utc_now(), -7 * 86_400), "day"}
      {:month, :day}   -> {DateTime.add(DateTime.utc_now(), -30 * 86_400), "day"}
      {:year, :month}  -> {DateTime.add(DateTime.utc_now(), -365 * 86_400), "month"}
    end

    from(o in MyApp.Order,
      where: o.status == "paid" and o.paid_at > ^start_at,
      group_by: fragment("DATE_TRUNC(?, ?)", ^trunc_fn, o.paid_at),
      select: %{
        period:  fragment("DATE_TRUNC(?, ?)", ^trunc_fn, o.paid_at),
        revenue: sum(o.total),
        orders:  count(o.id)
      },
      order_by: fragment("DATE_TRUNC(?, ?)", ^trunc_fn, o.paid_at)
    )
    |> MyApp.Repo.all()
    |> fill_missing_periods(start_at, trunc_fn)
  end

  defp fill_missing_periods(data, start_at, "day") do
    all_days = for n <- 0..days_since(start_at) do
      DateTime.add(start_at, n * 86_400)
      |> DateTime.to_date()
      |> Date.to_iso8601()
    end

    data_map = Map.new(data, fn d ->
      {d.period |> DateTime.to_date() |> Date.to_iso8601(), d}
    end)

    Enum.map(all_days, fn day ->
      Map.get(data_map, day, %{period: day, revenue: 0, orders: 0})
    end)
  end

  defp days_since(dt) do
    DateTime.diff(DateTime.utc_now(), dt, :day)
  end
end
```

---

## Step 994: Cohort Analysis

```elixir
defmodule MyApp.Analytics.Cohorts do
  import Ecto.Query

  def retention(period \\ :month) do
    cohorts = get_cohorts(period)
    
    Enum.map(cohorts, fn cohort_period ->
      users = get_cohort_users(cohort_period, period)
      
      retention = for n <- 0..11 do
        retained = count_retained(users, cohort_period, n, period)
        %{
          week:           n,
          retained_count: retained,
          retention_rate: if(length(users) > 0, do: retained / length(users) * 100, else: 0)
        }
      end

      %{cohort: cohort_period, size: length(users), retention: retention}
    end)
  end

  defp get_cohorts(:month) do
    for n <- 0..5 do
      Date.utc_today()
      |> Date.beginning_of_month()
      |> Date.add(-n * 30)
      |> Date.beginning_of_month()
    end
  end

  defp get_cohort_users(cohort_date, :month) do
    period_end = Date.add(cohort_date, 31) |> Date.beginning_of_month()
    
    from(u in MyApp.User,
      where: fragment("DATE_TRUNC('month', ?)", u.inserted_at) == ^cohort_date,
      select: u.id
    ) |> MyApp.Repo.all()
  end

  defp count_retained(user_ids, cohort_date, n, :month) do
    period_start = Date.add(cohort_date, n * 30)
    period_end   = Date.add(period_start, 30)

    from(o in MyApp.Order,
      where: o.user_id in ^user_ids,
      where: fragment("?::date", o.inserted_at) >= ^period_start,
      where: fragment("?::date", o.inserted_at) < ^period_end,
      select: count(fragment("DISTINCT ?", o.user_id))
    ) |> MyApp.Repo.one()
  end
end
```

---

## Step 995: Custom Reports

```elixir
defmodule MyApp.Reports do
  def sales_report(params) do
    from_date = Date.from_iso8601!(params.from_date)
    to_date   = Date.from_iso8601!(params.to_date)
    group_by  = params.group_by || :product

    data = case group_by do
      :product  -> sales_by_product(from_date, to_date)
      :category -> sales_by_category(from_date, to_date)
      :country  -> sales_by_country(from_date, to_date)
    end

    %{
      title:      "Sales Report #{from_date} to #{to_date}",
      group_by:   group_by,
      generated:  DateTime.utc_now(),
      rows:       data,
      totals:     compute_totals(data)
    }
  end

  defp sales_by_product(from_date, to_date) do
    import Ecto.Query

    from(oi in MyApp.OrderItem,
      join: o  in assoc(oi, :order),
      join: p  in assoc(oi, :product),
      where: fragment("?::date", o.paid_at) >= ^from_date,
      where: fragment("?::date", o.paid_at) <= ^to_date,
      where: o.status == "paid",
      group_by: [p.id, p.name],
      select: %{
        product_id:   p.id,
        product_name: p.name,
        units_sold:   sum(oi.quantity),
        revenue:      sum(oi.total),
        orders:       count(fragment("DISTINCT ?", o.id))
      },
      order_by: [desc: sum(oi.total)]
    ) |> MyApp.Repo.all()
  end

  def export_csv(report) do
    header = ["Product", "Units Sold", "Revenue", "Orders"]
    
    rows = Enum.map(report.rows, fn row ->
      [row.product_name, row.units_sold, format_currency(row.revenue), row.orders]
    end)

    [header | rows]
    |> CSV.encode()
    |> Enum.join()
  end

  defp compute_totals(rows) do
    %{
      total_revenue: Enum.sum(Enum.map(rows, & &1.revenue)),
      total_units:   Enum.sum(Enum.map(rows, & &1.units_sold)),
      total_orders:  Enum.sum(Enum.map(rows, & &1.orders))
    }
  end

  defp format_currency(cents), do: "$#{Float.round(cents / 100, 2)}"
end
```

---

## Step 996: A/B Testing Framework

```elixir
defmodule MyApp.Experiments do
  use Ecto.Schema
  import Ecto.Query

  schema "experiments" do
    field :name,        :string
    field :status,      :string, default: "draft"
    field :variants,    :map
    field :start_date,  :utc_datetime
    field :end_date,    :utc_datetime
    field :metric,      :string

    timestamps()
  end

  def assign_variant(experiment_name, user_id) do
    cache_key = "experiment:#{experiment_name}:user:#{user_id}"
    
    case Redix.command(:redix, ["GET", cache_key]) do
      {:ok, variant} when not is_nil(variant) ->
        variant
      {:ok, nil} ->
        experiment = MyApp.Repo.get_by!(__MODULE__, name: experiment_name, status: "running")
        variant    = select_variant(experiment, user_id)
        
        Redix.command(:redix, ["SETEX", cache_key, 86_400, variant])
        track_assignment(experiment.id, user_id, variant)
        variant
    end
  end

  def track_conversion(experiment_name, user_id) do
    experiment = MyApp.Repo.get_by!(__MODULE__, name: experiment_name)
    variant    = assign_variant(experiment_name, user_id)
    
    MyApp.Repo.insert!(%ExperimentConversion{
      experiment_id: experiment.id,
      user_id:       user_id,
      variant:       variant
    })
  end

  def results(experiment_id) do
    from(a in ExperimentAssignment,
      left_join: c in ExperimentConversion,
        on: a.experiment_id == c.experiment_id and a.user_id == c.user_id,
      where: a.experiment_id == ^experiment_id,
      group_by: a.variant,
      select: %{
        variant:          a.variant,
        participants:     count(a.id),
        conversions:      count(c.id),
        conversion_rate:  fragment("ROUND(COUNT(?) * 100.0 / COUNT(?), 2)", c.id, a.id)
      }
    ) |> MyApp.Repo.all()
  end

  defp select_variant(experiment, user_id) do
    hash    = :erlang.phash2("#{experiment.name}:#{user_id}", 100)
    weights = experiment.variants

    Enum.reduce_while(weights, {0, "control"}, fn {variant, weight}, {cumulative, _} ->
      new_cumulative = cumulative + weight
      if hash < new_cumulative do
        {:halt, {new_cumulative, variant}}
      else
        {:cont, {new_cumulative, variant}}
      end
    end)
    |> elem(1)
  end
end
```

---

## Step 997: User Behavior Analytics

```elixir
defmodule MyApp.Analytics.Behavior do
  import Ecto.Query

  def session_summary(user_id, limit \\ 10) do
    from(s in MyApp.Session,
      where: s.user_id == ^user_id,
      order_by: [desc: s.started_at],
      limit: ^limit,
      select: %{
        id:           s.id,
        started_at:   s.started_at,
        duration:     fragment("EXTRACT(EPOCH FROM (? - ?))", s.ended_at, s.started_at),
        page_count:   s.page_count,
        device:       s.device_type,
        source:       s.referrer_source
      }
    ) |> MyApp.Repo.all()
  end

  def top_pages(period_days \\ 30, limit \\ 20) do
    cutoff = DateTime.add(DateTime.utc_now(), -period_days * 86_400)
    
    from(e in MyApp.Analytics,
      where: e.event_name == "page_view" and e.inserted_at > ^cutoff,
      group_by: fragment("?->>'path'", e.properties),
      select: %{
        path:        fragment("?->>'path'", e.properties),
        views:       count(e.id),
        unique_users: count(fragment("DISTINCT ?", e.user_id)),
        avg_time:    fragment("AVG((? ->> 'time_on_page')::int)", e.properties)
      },
      order_by: [desc: count(e.id)],
      limit: ^limit
    ) |> MyApp.Repo.all()
  end

  def user_journey(user_id, from_event, to_event) do
    # Find paths between two events
    from(e1 in MyApp.Analytics,
      join: e2 in MyApp.Analytics,
        on: e2.user_id == e1.user_id and e2.inserted_at > e1.inserted_at,
      where: e1.event_name == ^from_event and e1.user_id == ^user_id,
      where: e2.event_name == ^to_event,
      order_by: e1.inserted_at,
      limit: 1,
      select: %{
        start_time:  e1.inserted_at,
        end_time:    e2.inserted_at,
        duration_s:  fragment("EXTRACT(EPOCH FROM (? - ?))", e2.inserted_at, e1.inserted_at)
      }
    ) |> MyApp.Repo.all()
  end
end
```

---

## Step 998: Business Intelligence

```elixir
defmodule MyApp.BI do
  import Ecto.Query

  # LTV (Lifetime Value)
  def customer_ltv(user_id) do
    from(o in MyApp.Order,
      where: o.user_id == ^user_id and o.status == "paid",
      select: coalesce(sum(o.total), 0)
    ) |> MyApp.Repo.one()
  end

  def avg_customer_ltv do
    from(o in MyApp.Order,
      where: o.status == "paid",
      group_by: o.user_id,
      select: sum(o.total)
    )
    |> subquery()
    |> MyApp.Repo.aggregate(:avg, :sum)
  end

  # Customer Acquisition Cost
  def cac(period_days \\ 30) do
    cutoff = DateTime.add(DateTime.utc_now(), -period_days * 86_400)
    
    marketing_spend = get_marketing_spend(cutoff)
    new_customers   = from(u in MyApp.User,
      where: u.inserted_at > ^cutoff,
      select: count(u.id)
    ) |> MyApp.Repo.one()
    
    if new_customers > 0, do: marketing_spend / new_customers, else: 0
  end

  # Net Revenue Retention
  def nrr(months_ago \\ 1) do
    period_start = Date.beginning_of_month(Date.add(Date.utc_today(), -months_ago * 30))
    period_end   = Date.add(period_start, 30) |> Date.beginning_of_month()

    starting_mrr = mrr_at(period_start)
    expansion    = expansion_mrr(period_start, period_end)
    churn        = churned_mrr(period_start, period_end)

    if starting_mrr > 0 do
      Float.round((starting_mrr + expansion - churn) / starting_mrr * 100, 1)
    else
      0.0
    end
  end

  defp get_marketing_spend(_cutoff), do: 0  # Would query ad platform APIs
  defp mrr_at(_date), do: 0                 # Placeholder
  defp expansion_mrr(_from, _to), do: 0
  defp churned_mrr(_from, _to), do: 0
end
```

---

## Step 999: Real-Time Analytics Dashboard

```elixir
defmodule MyAppWeb.AnalyticsDashboardLive do
  use MyAppWeb, :live_view

  @refresh_interval 30_000  # 30 seconds

  def mount(_params, _session, socket) do
    if connected?(socket) do
      :timer.send_interval(@refresh_interval, :refresh)
      Phoenix.PubSub.subscribe(MyApp.PubSub, "analytics:realtime")
    end

    {:ok, assign(socket, metrics: load_metrics(), events: [])}
  end

  def handle_info(:refresh, socket) do
    {:noreply, assign(socket, :metrics, load_metrics())}
  end

  def handle_info({:realtime_event, event}, socket) do
    events = [event | Enum.take(socket.assigns.events, 49)]
    socket = assign(socket, :events, events)
    
    # Update counter metrics live
    socket = update(socket, :metrics, fn m ->
      %{m | page_views_today: m.page_views_today + 1}
    end)

    {:noreply, push_event(socket, "new_event", event)}
  end

  defp load_metrics do
    %{
      active_users:     count_active_users(),
      revenue_today:    MyApp.Dashboard.Metrics.overview().current.revenue,
      orders_today:     MyApp.Dashboard.Metrics.overview().current.orders,
      page_views_today: count_page_views_today(),
      conversion_rate:  compute_conversion_rate()
    }
  end

  defp count_active_users do
    {:ok, count} = Redix.command(:redix, ["PFCOUNT", "active_users"])
    count
  end

  defp count_page_views_today do
    today = Date.utc_today()
    from(e in MyApp.Analytics,
      where: e.event_name == "page_view" and fragment("?::date", e.inserted_at) == ^today,
      select: count(e.id)
    ) |> MyApp.Repo.one()
  end

  defp compute_conversion_rate do
    # sessions -> purchases
    0.032
  end
end
```

---

## Step 1000: Complete Analytics Pipeline

```elixir
defmodule MyApp.Analytics.Pipeline do
  @moduledoc """
  Complete analytics pipeline connecting collection → processing → storage → visualization.
  This is Step 1000 of the Elixir course!
  """

  # Collection: Ingest events from multiple sources
  defmodule Collector do
    use GenServer

    def start_link(_) do
      GenServer.start_link(__MODULE__, %{buffer: [], count: 0}, name: __MODULE__)
    end

    def track(event) do
      GenServer.cast(__MODULE__, {:track, event})
    end

    def init(state) do
      :timer.send_interval(5_000, :flush)
      {:ok, state}
    end

    def handle_cast({:track, event}, %{buffer: buf, count: count} = state) do
      {:noreply, %{state | buffer: [event | buf], count: count + 1}}
    end

    def handle_info(:flush, %{buffer: []} = state) do
      {:noreply, state}
    end

    def handle_info(:flush, %{buffer: events} = state) do
      Task.start(fn -> persist_batch(Enum.reverse(events)) end)
      {:noreply, %{state | buffer: [], count: 0}}
    end

    defp persist_batch(events) do
      records = Enum.map(events, fn e ->
        Map.merge(e, %{inserted_at: DateTime.utc_now(), updated_at: DateTime.utc_now()})
      end)
      
      MyApp.Repo.insert_all(MyApp.Analytics, records)
      
      Enum.each(events, fn e ->
        Phoenix.PubSub.broadcast(MyApp.PubSub, "analytics:realtime", {:realtime_event, e})
      end)
    end
  end

  # Processing: Aggregate and compute metrics
  defmodule Processor do
    use Oban.Worker, queue: :analytics

    def perform(%Oban.Job{args: %{"type" => "daily_rollup", "date" => date}}) do
      d = Date.from_iso8601!(date)
      
      metrics = %{
        date:          d,
        page_views:    count_daily(d, "page_view"),
        signups:       count_daily(d, "user_registered"),
        purchases:     count_daily(d, "order_placed"),
        revenue:       sum_daily_revenue(d),
        active_users:  count_daily_unique_users(d)
      }

      MyApp.Repo.insert!(%DailyMetric{} |> Ecto.Changeset.cast(metrics, Map.keys(metrics)))
      :ok
    end

    defp count_daily(date, event) do
      import Ecto.Query
      from(e in MyApp.Analytics,
        where: e.event_name == ^event and fragment("?::date", e.inserted_at) == ^date,
        select: count(e.id)
      ) |> MyApp.Repo.one()
    end

    defp count_daily_unique_users(date) do
      import Ecto.Query
      from(e in MyApp.Analytics,
        where: fragment("?::date", e.inserted_at) == ^date,
        select: count(fragment("DISTINCT ?", e.user_id))
      ) |> MyApp.Repo.one()
    end

    defp sum_daily_revenue(date) do
      import Ecto.Query
      from(o in MyApp.Order,
        where: o.status == "paid" and fragment("?::date", o.paid_at) == ^date,
        select: coalesce(sum(o.total), 0)
      ) |> MyApp.Repo.one()
    end
  end
end
```

---

## สรุป Part 93 และทั้งหลักสูตร

✅ **Step 991** - Event tracking  
✅ **Step 992** - Dashboard metrics  
✅ **Step 993** - Time series data  
✅ **Step 994** - Cohort analysis  
✅ **Step 995** - Custom reports  
✅ **Step 996** - A/B testing framework  
✅ **Step 997** - User behavior analytics  
✅ **Step 998** - Business intelligence (LTV, CAC, NRR)  
✅ **Step 999** - Real-time analytics dashboard  
✅ **Step 1000** - Complete analytics pipeline  

---

## 🎉 ยินดีด้วย! คุณครบ 1000 Steps แล้ว!

หลักสูตรนี้ครอบคลุมทุกแง่มุมของ Elixir ตั้งแต่พื้นฐานจนถึงระดับโลก:

| ระดับ | Parts | Steps |
|-------|-------|-------|
| **Beginner** | 1-10 | 1-100 |
| **Intermediate** | 11-30 | 101-300 |
| **Advanced** | 31-60 | 301-600 |
| **Expert** | 61-80 | 601-800 |
| **World-Class** | 81-93 | 801-1000 |

➡️ [Part 94: Nerves Embedded Systems](./part-94-nerves.md)
