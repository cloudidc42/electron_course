# Part 95: LiveBook & Data Science (Steps 1011-1020)

## Step 1011: LiveBook Introduction

```elixir
# LiveBook is Elixir's interactive notebook (like Jupyter)
# Install: mix escript.install hex livebook
# Run: livebook server

# In a LiveBook cell, Elixir runs live:
defmodule DataAnalysis do
  def summarize(data) do
    n    = length(data)
    sum  = Enum.sum(data)
    mean = sum / n
    
    sorted  = Enum.sort(data)
    median  = Enum.at(sorted, div(n, 2))
    
    variance = Enum.sum(Enum.map(data, fn x -> :math.pow(x - mean, 2) end)) / n
    std_dev  = :math.sqrt(variance)
    
    %{n: n, sum: sum, mean: mean, median: median, std_dev: Float.round(std_dev, 4)}
  end
end

prices = [10.5, 20.0, 15.3, 22.1, 18.7, 12.0, 25.0]
IO.inspect(DataAnalysis.summarize(prices))
```

---

## Step 1012: Data Visualization with VegaLite

```elixir
# In LiveBook with {:vega_lite, "~> 0.1"} and {:kino_vega_lite, "~> 0.1"}

alias VegaLite, as: Vl

# Line chart
Vl.new(width: 600, height: 400)
|> Vl.data_from_values(
  Enum.map(1..12, fn m ->
    %{
      month:   Date.new!(2024, m, 1) |> Date.to_iso8601(),
      revenue: :rand.uniform(50_000) + 20_000
    }
  end)
)
|> Vl.mark(:line, point: true)
|> Vl.encode_field(:x, "month", type: :temporal, title: "Month")
|> Vl.encode_field(:y, "revenue", type: :quantitative, title: "Revenue ($)")

# Bar chart with color
Vl.new(width: 500)
|> Vl.data_from_values([
  %{category: "A", value: 100, group: "Q1"},
  %{category: "B", value: 150, group: "Q1"},
  %{category: "A", value: 120, group: "Q2"},
  %{category: "B", value: 180, group: "Q2"}
])
|> Vl.mark(:bar)
|> Vl.encode_field(:x,     "category", type: :nominal)
|> Vl.encode_field(:y,     "value",    type: :quantitative)
|> Vl.encode_field(:color, "group",    type: :nominal)
|> Vl.encode_field(:xOffset, "group")
```

---

## Step 1013: Exploratory Data Analysis

```elixir
# mix.exs: {:explorer, "~> 0.9"}, {:kino_explorer, "~> 0.1"}

alias Explorer.DataFrame
alias Explorer.Series

# Load and explore data
df = DataFrame.from_csv!("sales.csv")

# Basic exploration
IO.puts "Shape: #{DataFrame.n_rows(df)} rows × #{DataFrame.n_columns(df)} cols"
IO.inspect DataFrame.dtypes(df)
IO.inspect DataFrame.describe(df)

# Filter and transform
filtered = df
  |> DataFrame.filter(col("revenue") > 1000)
  |> DataFrame.select(["date", "product", "revenue", "quantity"])
  |> DataFrame.sort_by([desc: col("revenue")])

# Group by and aggregate
summary = df
  |> DataFrame.group_by("product")
  |> DataFrame.summarise(
    total_revenue: sum(col("revenue")),
    avg_revenue:   mean(col("revenue")),
    count:         count(col("revenue"))
  )
  |> DataFrame.sort_by([desc: col("total_revenue")])

# Correlation analysis
numeric_cols = ["revenue", "quantity", "price"]
correlations = for c1 <- numeric_cols, c2 <- numeric_cols, into: %{} do
  r = Explorer.Series.pearson(DataFrame.pull(df, c1), DataFrame.pull(df, c2))
  {"#{c1}_vs_#{c2}", Float.round(r, 3)}
end
```

---

## Step 1014: Statistical Analysis

```elixir
defmodule MyApp.Statistics do
  def t_test(group_a, group_b) do
    n_a = length(group_a)
    n_b = length(group_b)
    
    mean_a = Enum.sum(group_a) / n_a
    mean_b = Enum.sum(group_b) / n_b
    
    var_a = variance(group_a, mean_a)
    var_b = variance(group_b, mean_b)
    
    se = :math.sqrt(var_a / n_a + var_b / n_b)
    t  = (mean_a - mean_b) / se
    df = n_a + n_b - 2
    
    %{t_statistic: Float.round(t, 4), degrees_of_freedom: df,
      mean_a: Float.round(mean_a, 4), mean_b: Float.round(mean_b, 4)}
  end

  def chi_square(observed, expected) do
    chi2 = Enum.zip(observed, expected)
      |> Enum.sum_by(fn {o, e} -> :math.pow(o - e, 2) / e end)
    
    df = length(observed) - 1
    %{chi2: Float.round(chi2, 4), df: df}
  end

  def confidence_interval(data, confidence \\ 0.95) do
    n    = length(data)
    mean = Enum.sum(data) / n
    std  = :math.sqrt(variance(data, mean))
    se   = std / :math.sqrt(n)
    
    # Z-score for confidence level
    z = case confidence do
      0.90 -> 1.645
      0.95 -> 1.960
      0.99 -> 2.576
    end
    
    margin = z * se
    {mean - margin, mean + margin}
  end

  defp variance(data, mean) do
    n = length(data)
    data |> Enum.sum_by(fn x -> :math.pow(x - mean, 2) end) |> Kernel./(n - 1)
  end
end
```

---

## Step 1015: Data Pipelines with Explorer

```elixir
defmodule MyApp.DataPipeline do
  alias Explorer.DataFrame
  alias Explorer.Series

  def process_sales_data(csv_path) do
    csv_path
    |> DataFrame.from_csv!(
         dtypes: %{
           "date"     => :date,
           "revenue"  => :float,
           "quantity" => :integer
         }
       )
    |> clean_data()
    |> add_derived_columns()
    |> aggregate_by_period()
  end

  defp clean_data(df) do
    df
    |> DataFrame.filter(not(is_nil(col("revenue"))))
    |> DataFrame.filter(col("revenue") > 0)
    |> DataFrame.filter(col("quantity") > 0)
    |> DataFrame.mutate(
         revenue:  cast(col("revenue"), :float),
         quantity: cast(col("quantity"), :integer)
       )
  end

  defp add_derived_columns(df) do
    DataFrame.mutate(df,
      avg_order_value: col("revenue") / col("quantity"),
      month:           strftime(col("date"), "%Y-%m"),
      day_of_week:     day_of_week(col("date"))
    )
  end

  defp aggregate_by_period(df) do
    df
    |> DataFrame.group_by("month")
    |> DataFrame.summarise(
         total_revenue: sum(col("revenue")),
         total_orders:  count(col("revenue")),
         avg_value:     mean(col("avg_order_value"))
       )
    |> DataFrame.sort_by(col("month"))
  end

  def export_to_csv(df, path) do
    DataFrame.to_csv!(df, path)
  end
end
```

---

## Step 1016: Machine Learning in LiveBook

```elixir
# In LiveBook with {:nx, "~> 0.7"}, {:axon, "~> 0.6"}, {:scholar, "~> 0.3"}

# Data preparation
alias Explorer.DataFrame
import Scholar.Preprocessing

raw_data = DataFrame.from_csv!("housing.csv")

features = ~w(sqft bedrooms bathrooms age)
target   = "price"

x = raw_data |> DataFrame.select(features) |> Explorer.Backends.LazySeries.to_tensor()
y = raw_data |> DataFrame.pull(target) |> Explorer.Series.to_tensor()

# Normalize
{x_train, x_mean, x_std} = normalize(x)

# Split 80/20
n       = Nx.axis_size(x, 0)
n_train = trunc(n * 0.8)

x_train = x[0..n_train - 1]
x_test  = x[n_train..n - 1]
y_train = y[0..n_train - 1]
y_test  = y[n_train..n - 1]

# Train linear regression
model = Scholar.Linear.LinearRegression.fit(x_train, y_train)

# Evaluate
y_pred = Scholar.Linear.LinearRegression.predict(model, x_test)

mae  = Scholar.Metrics.Regression.mean_absolute_error(y_test, y_pred)
rmse = Scholar.Metrics.Regression.mean_squared_error(y_test, y_pred) |> Nx.sqrt()

IO.puts "MAE: #{Nx.to_number(mae)}"
IO.puts "RMSE: #{Nx.to_number(rmse)}"
```

---

## Step 1017: LiveBook Kino Widgets

```elixir
# Interactive widgets in LiveBook

# Input widget
name_input = Kino.Input.text("Your name:")
Kino.render(name_input)

# Read value
name = Kino.Input.read(name_input)
IO.puts "Hello, #{name}!"

# DataTable display
data = for _ <- 1..20 do
  %{
    id:    :rand.uniform(1000),
    name:  Enum.random(~w(Alice Bob Charlie Diana Eve)),
    score: :rand.uniform(100)
  }
end

Kino.DataTable.new(data, keys: [:id, :name, :score])

# Markdown output
Kino.Markdown.new("""
## Analysis Results

| Metric | Value |
|--------|-------|
| Mean   | #{Enum.sum(Enum.map(data, & &1.score)) / length(data) |> Float.round(1)} |
| Max    | #{Enum.max(Enum.map(data, & &1.score))} |
| Min    | #{Enum.min(Enum.map(data, & &1.score))} |
""")
```

---

## Step 1018: Streaming Data in LiveBook

```elixir
# Live updating charts in LiveBook

widget = Kino.VegaLite.new()
  |> Vl.new(width: 600, height: 300)
  |> Vl.mark(:line)
  |> Vl.encode_field(:x, "x", type: :quantitative)
  |> Vl.encode_field(:y, "y", type: :quantitative)
  |> Kino.VegaLite.new()

# Stream data updates
Task.start(fn ->
  for x <- 1..100 do
    point = %{x: x, y: :math.sin(x / 10.0) + :rand.uniform() * 0.1}
    Kino.VegaLite.push(widget, point)
    Process.sleep(100)
  end
end)

widget
```

---

## Step 1019: Report Generation

```elixir
defmodule MyApp.Reports.Generator do
  # Generate HTML reports from LiveBook notebooks

  def generate_html(data) do
    charts     = build_charts(data)
    stats      = build_statistics(data)
    narrative  = generate_narrative(stats)

    """
    <!DOCTYPE html>
    <html>
    <head>
      <title>Analytics Report - #{Date.utc_today()}</title>
      <style>
        body { font-family: sans-serif; max-width: 1200px; margin: 0 auto; padding: 20px; }
        .metric { display: inline-block; background: #f0f4ff; padding: 20px; margin: 10px; border-radius: 8px; }
        .metric .value { font-size: 2em; font-weight: bold; }
      </style>
    </head>
    <body>
      <h1>Analytics Report</h1>
      <p>Generated: #{DateTime.utc_now() |> DateTime.to_string()}</p>
      
      <section class="metrics">
        #{render_metrics(stats)}
      </section>
      
      <section class="charts">
        #{charts}
      </section>
      
      <section class="narrative">
        #{narrative}
      </section>
    </body>
    </html>
    """
  end

  defp render_metrics(stats) do
    Enum.map_join(stats, "\n", fn {name, value} ->
      """
      <div class="metric">
        <div class="label">#{humanize(name)}</div>
        <div class="value">#{format_value(value)}</div>
      </div>
      """
    end)
  end

  defp build_charts(_data), do: ""
  defp build_statistics(_data), do: %{revenue: 100_000, orders: 500}
  defp generate_narrative(_stats), do: ""
  defp humanize(atom), do: atom |> to_string() |> String.replace("_", " ") |> String.capitalize()
  defp format_value(n) when is_integer(n), do: to_string(n)
  defp format_value(n) when is_float(n),   do: Float.round(n, 2) |> to_string()
  defp format_value(v), do: to_string(v)
end
```

---

## Step 1020: Notebooks as Documentation

```elixir
# Using LiveBook for interactive documentation

# Smart cells for database queries
# Use Kino.SmartCell for self-describing cells

defmodule MyApp.Docs.QueryCell do
  use Kino.JS
  use Kino.JS.Live

  def new(query, opts \\ []) do
    Kino.JS.Live.new(__MODULE__, {query, opts})
  end

  def init({query, opts}, ctx) do
    {:ok, assign(ctx, query: query, result: nil, opts: opts)}
  end

  def handle_event("execute", _data, ctx) do
    result = MyApp.Repo.query!(ctx.assigns.query) |> format_result()
    {:noreply, assign(ctx, result: result)}
  end

  defp format_result(%{columns: cols, rows: rows}) do
    header = Enum.join(cols, " | ")
    divider = String.duplicate("-", String.length(header))
    data_rows = Enum.map(rows, fn row ->
      row |> Enum.map(&to_string/1) |> Enum.join(" | ")
    end) |> Enum.join("\n")
    "#{header}\n#{divider}\n#{data_rows}"
  end
end

# In LiveBook notebook:
# This documents the API usage with runnable examples

# 1. Connection setup
{:ok, pid} = MyApp.Repo.start_link(url: "postgres://localhost/myapp_dev")

# 2. Query example
MyApp.Repo.all(from u in MyApp.User, limit: 5)
|> Enum.each(&IO.inspect/1)
```

---

## สรุป Part 95

✅ **Step 1011** - LiveBook introduction  
✅ **Step 1012** - Data visualization with VegaLite  
✅ **Step 1013** - Exploratory data analysis  
✅ **Step 1014** - Statistical analysis  
✅ **Step 1015** - Data pipelines with Explorer  
✅ **Step 1016** - Machine learning in LiveBook  
✅ **Step 1017** - LiveBook Kino widgets  
✅ **Step 1018** - Streaming data  
✅ **Step 1019** - Report generation  
✅ **Step 1020** - Notebooks as documentation  

➡️ [Part 96: Security Hardening](./part-96-security.md)
