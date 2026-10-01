# Part 26: Performance และ Optimization (Steps 281-300)

## Step 281: BEAM Performance Characteristics

```
BEAM Strengths:
- Concurrency: millions of lightweight processes
- Soft real-time: predictable latencies
- Hot code reloading: no downtime deploys
- Garbage collection: per-process GC (no stop-the-world)
- Fault tolerance: supervisor trees

BEAM Weaknesses:
- CPU-intensive work (use NIF or Task.async)
- Memory: each process has overhead (~2KB min)
- Binary data > 64 bytes goes to shared heap

Profile before optimizing!
```

---

## Step 282: Profiling กับ :eprof และ :fprof

```elixir
# :eprof - time profiling (fast)
:eprof.start_profiling([self()])
expensive_function()
:eprof.stop_profiling()
:eprof.analyze(:total)

# :fprof - detailed call graph (slow but thorough)
:fprof.apply(&MyModule.function/1, [arg])
:fprof.profile()
:fprof.analyse([sort: :own, totals: true])

# Mix task
# mix profile.eprof -e "MyModule.work()"
# mix profile.fprof -e "MyModule.work()"

# :observer - GUI profiler
:observer.start()

# Benchee สำหรับ benchmarking
# mix.exs: {:benchee, "~> 1.3", only: :dev}

defmodule BenchmarkExample do
  def run do
    data = Enum.to_list(1..10_000)
    
    Benchee.run(
      %{
        "Enum.map"    => fn -> Enum.map(data, &(&1 * 2)) end,
        "for comprehension" => fn -> for n <- data, do: n * 2 end,
        ":lists.map"  => fn -> :lists.map(&(&1 * 2), data) end
      },
      time: 5,
      memory_time: 2,
      formatters: [Benchee.Formatters.Console, Benchee.Formatters.HTML]
    )
  end
end
```

---

## Step 283: ETS สำหรับ Performance

```elixir
defmodule FastCache do
  @table :fast_cache
  
  def setup do
    :ets.new(@table, [
      :named_table,
      :set,
      :public,
      read_concurrency: true,   # parallel reads
      write_concurrency: true   # parallel writes (for bags)
    ])
  end
  
  # O(1) read - ไม่ต้องผ่าน GenServer
  def get(key) do
    case :ets.lookup(@table, key) do
      [{^key, value, _expires}] -> value
      []                        -> nil
    end
  end
  
  # Atomic counter - thread-safe, O(1)
  def increment_counter(key) do
    :ets.update_counter(@table, key, {2, 1}, {key, 0})
  end
  
  # Bulk insert
  def put_many(pairs) when is_list(pairs) do
    :ets.insert(@table, pairs)
  end
end

# เปรียบเทียบ ETS vs GenServer
# ETS get: ~0.3μs (ไม่มี process message passing)
# GenServer.call get: ~3μs (message passing overhead)
# ความต่าง ~10x สำหรับ read-heavy workloads
```

---

## Step 284: Database Performance

```elixir
defmodule MyApp.QueryOptimizer do
  import Ecto.Query
  alias MyApp.{Repo, Blog.Post}
  
  # N+1 Problem
  
  # ❌ BAD: N+1 queries
  def bad_posts_with_users do
    posts = Repo.all(Post)  # 1 query
    Enum.map(posts, fn post ->
      user = Repo.get!(MyApp.Accounts.User, post.user_id)  # N queries!
      %{post: post, user: user}
    end)
  end
  
  # ✅ GOOD: Preload in single query
  def good_posts_with_users do
    Post
    |> preload(:user)  # 2 queries (or 1 join)
    |> Repo.all()
  end
  
  # ✅ EVEN BETTER: Join in single query
  def best_posts_with_users do
    from(p in Post,
      join: u in assoc(p, :user),
      preload: [user: u]
    )
    |> Repo.all()
  end
  
  # Indexing
  
  # migration: create_index(:posts, [:user_id, :status])
  # เพิ่ม index สำหรับ columns ที่ filter/sort บ่อย
  
  def find_user_published_posts(user_id) do
    from(p in Post,
      where: p.user_id == ^user_id and p.status == :published,
      # ← ใช้ index (user_id, status)
      order_by: [desc: p.inserted_at]
    )
    |> Repo.all()
  end
  
  # Pagination (Cursor-based)
  
  def cursor_paginate(cursor, limit \\ 20) do
    query = from(p in Post,
      order_by: [desc: p.id],
      limit: ^(limit + 1)
    )
    
    query = if cursor do
      where(query, [p], p.id < ^cursor)
    else
      query
    end
    
    results = Repo.all(query)
    
    has_more = length(results) > limit
    items = Enum.take(results, limit)
    next_cursor = if has_more, do: List.last(items).id, else: nil
    
    %{items: items, next_cursor: next_cursor, has_more: has_more}
  end
  
  # Batch operations
  
  def batch_update_status(post_ids, status) do
    from(p in Post, where: p.id in ^post_ids)
    |> Repo.update_all(set: [status: status, updated_at: NaiveDateTime.utc_now()])
  end
  
  def batch_insert(posts) do
    now = NaiveDateTime.utc_now() |> NaiveDateTime.truncate(:second)
    
    entries = Enum.map(posts, fn post ->
      Map.merge(post, %{inserted_at: now, updated_at: now})
    end)
    
    Repo.insert_all(Post, entries, on_conflict: :nothing)
  end
end
```

---

## Step 285: Connection Pooling

```elixir
# config/config.exs
config :my_app, MyApp.Repo,
  pool: Ecto.Adapters.SQL.Sandbox,  # test
  pool_size: 10                      # production

# config/prod.exs
config :my_app, MyApp.Repo,
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "20"),
  queue_target: 50,    # milliseconds
  queue_interval: 1000 # milliseconds

# pool_size ควรเป็น = (CPU cores * 2) + HDD spindles
# สำหรับ web app: 10-20 is typical
# Monitor: Repo.checkout/1, :telemetry events

defmodule MyApp.DBMetrics do
  def setup do
    events = [
      [:my_app, :repo, :query]
    ]
    
    :telemetry.attach_many("db-metrics", events, &handle_event/4, nil)
  end
  
  def handle_event([:my_app, :repo, :query], measurements, metadata, _) do
    duration_ms = measurements.query_time / 1_000_000
    
    if duration_ms > 100 do
      require Logger
      Logger.warning("Slow query (#{Float.round(duration_ms, 2)}ms): #{metadata.query}")
    end
  end
end
```

---

## Step 286: Process Pooling ด้วย Poolboy/Nimble

```elixir
# mix.exs
{:nimble_pool, "~> 1.0"}

defmodule MyApp.HTTPPool do
  @pool_name :http_pool
  
  def child_spec(_opts) do
    NimblePool.child_spec(
      worker: {MyApp.HTTPWorker, []},
      pool_size: 10,
      name: @pool_name
    )
  end
  
  def request(url, opts \\ []) do
    NimblePool.checkout!(@pool_name, :checkout, fn _from, worker ->
      try do
        result = MyApp.HTTPWorker.get(worker, url, opts)
        {result, worker}
      rescue
        e -> {{:error, e}, worker}
      end
    end)
  end
end

defmodule MyApp.HTTPWorker do
  @behaviour NimblePool
  
  @impl NimblePool
  def init_worker(_pool_state) do
    {:ok, conn} = Mint.HTTP.connect(:https, "api.example.com", 443)
    {:ok, conn, _}
  end
  
  @impl NimblePool
  def handle_checkout(:checkout, _from, conn, _pool_state) do
    {:ok, conn, conn, _}
  end
  
  @impl NimblePool
  def handle_checkin(conn, _from, _old_conn, pool_state) do
    {:ok, conn, pool_state}
  end
  
  def get(conn, path, _opts) do
    {:ok, conn, response} = Mint.HTTP.request(conn, "GET", path, [], nil)
    {response, conn}
  end
end
```

---

## Step 287: Caching Strategies

```elixir
defmodule MyApp.Cache do
  @moduledoc "Multi-level caching strategy"
  
  # L1: Process dictionary (per-process, request-scoped)
  def get_from_process(key) do
    Process.get({:cache, key})
  end
  
  def put_to_process(key, value) do
    Process.put({:cache, key}, value)
    value
  end
  
  # L2: ETS (shared in-memory, fast)
  def get_from_ets(key) do
    case :ets.lookup(:app_cache, key) do
      [{^key, val, exp}] when exp > now() -> {:hit, val}
      _ -> :miss
    end
  end
  
  # L3: Redis (distributed, persistent)
  def get_from_redis(key) do
    case Redix.command(:redis, ["GET", key]) do
      {:ok, nil}   -> :miss
      {:ok, value} -> {:hit, :erlang.binary_to_term(value)}
      _            -> :miss
    end
  end
  
  # Multi-level get with fallback
  def get(key, loader_fn) do
    with :miss <- get_from_process(key),
         :miss <- get_from_ets(key) do
      # Cache miss - load and cache
      value = loader_fn.()
      put_to_process(key, value)
      put_to_ets(key, value)
      value
    else
      {:hit, value} -> value
    end
  end
  
  defp now, do: System.monotonic_time(:millisecond)
  
  defp put_to_ets(key, value, ttl \\ 60_000) do
    :ets.insert(:app_cache, {key, value, now() + ttl})
  end
end

# Cache-aside pattern
defmodule MyApp.Blog do
  def get_post(id) do
    MyApp.Cache.get("post:#{id}", fn ->
      Repo.get!(Post, id)
    end)
  end
  
  def update_post(id, attrs) do
    {:ok, post} = do_update_post(id, attrs)
    # Invalidate cache
    :ets.delete(:app_cache, "post:#{id}")
    {:ok, post}
  end
end
```

---

## Step 288: Memory Optimization

```elixir
defmodule MemoryOptimized do
  # Binaries > 64 bytes ไปอยู่ใน shared heap
  # ดี: หลาย processes share binary โดยไม่ copy
  # เสีย: GC ไม่ free จนกว่า process ที่ reference จะตาย
  
  # ✅ Binary ขนาดใหญ่ → เก็บแยก reference
  def process_large_file(path) do
    {:ok, content} = File.read(path)
    result = process_content(content)
    # content ไม่ถูก copy เข้า process heap
    result
  end
  
  # Process Hibernation
  defmodule IdleServer do
    use GenServer
    
    @idle_timeout 5_000
    
    @impl true
    def handle_call(:request, _from, state) do
      {:reply, :ok, state, @idle_timeout}
    end
    
    @impl true
    def handle_info(:timeout, state) do
      # Hibernate จนกว่าจะมี message ใหม่
      # ลด memory จาก ~1MB เป็น ~200 bytes
      {:noreply, state, :hibernate}
    end
  end
  
  # Use atoms sparingly - they're never garbage collected!
  # ❌ BAD: String.to_atom("user_#{id}")  # memory leak!
  # ✅ GOOD: String.to_existing_atom("known_atom")
  # ✅ GOOD: keep as string or use integer keys
  
  # Binary optimization
  def efficient_join(list) do
    # ใช้ IO data แทน string concat
    # ไม่สร้าง intermediate binaries
    IO.iodata_to_binary(Enum.intersperse(list, ", "))
  end
end
```

---

## Step 289: Concurrent Processing Patterns

```elixir
defmodule ParallelProcessor do
  # Fan-out / Fan-in pattern
  
  def process_all(items) do
    items
    |> Task.async_stream(&process_item/1,
         max_concurrency: System.schedulers_online() * 2,
         timeout: 30_000,
         on_timeout: :kill_task)
    |> Stream.filter(&match?({:ok, _}, &1))
    |> Stream.map(fn {:ok, result} -> result end)
    |> Enum.to_list()
  end
  
  # Batch parallel processing
  def batch_process(items, batch_size \\ 100) do
    items
    |> Enum.chunk_every(batch_size)
    |> Task.async_stream(&process_batch/1, max_concurrency: 4, timeout: 60_000)
    |> Enum.flat_map(fn {:ok, results} -> results end)
  end
  
  defp process_item(item) do
    # Heavy computation
    item
  end
  
  defp process_batch(items) do
    Enum.map(items, &process_item/1)
  end
  
  # Pipeline with back-pressure using GenStage
  def setup_pipeline(source_data) do
    {:ok, producer}  = GenStage.start_link(DataProducer, source_data)
    {:ok, processor} = GenStage.start_link(DataProcessor, [])
    {:ok, consumer}  = GenStage.start_link(DataConsumer, [])
    
    GenStage.sync_subscribe(processor, to: producer, max_demand: 100)
    GenStage.sync_subscribe(consumer, to: processor, max_demand: 100)
    
    {producer, processor, consumer}
  end
end

defmodule DataProducer do
  use GenStage
  
  def init(data) do
    {:producer, {data, 0}}
  end
  
  def handle_demand(demand, {data, offset}) do
    chunk = Enum.slice(data, offset, demand)
    {:noreply, chunk, {data, offset + length(chunk)}}
  end
end
```

---

## Step 290: Production Monitoring

```elixir
defmodule MyApp.Metrics do
  @moduledoc "Application metrics with Telemetry"
  
  def child_spec(_) do
    children = [
      {TelemetryMetricsPrometheus, metrics: metrics()}
    ]
    
    Supervisor.child_spec(
      %{id: __MODULE__, start: {Supervisor, :start_link, [children, [strategy: :one_for_one]]}},
      []
    )
  end
  
  def metrics do
    [
      # HTTP request metrics
      Telemetry.Metrics.summary("phoenix.endpoint.stop.duration",
        unit: {:native, :millisecond},
        tags: [:status, :route]
      ),
      
      # Database metrics
      Telemetry.Metrics.summary("my_app.repo.query.total_time",
        unit: {:native, :millisecond},
        tags: [:source]
      ),
      
      # Custom business metrics
      Telemetry.Metrics.counter("my_app.orders.created.count"),
      Telemetry.Metrics.sum("my_app.orders.revenue.total"),
      
      # VM metrics
      Telemetry.Metrics.last_value("vm.memory.total",
        unit: {:byte, :megabyte}
      ),
      Telemetry.Metrics.last_value("vm.total_run_queue_lengths.total")
    ]
  end
  
  # Emit custom metrics
  def track_order(order) do
    :telemetry.execute([:my_app, :orders, :created], %{count: 1}, %{})
    :telemetry.execute([:my_app, :orders, :revenue], %{total: order.total}, %{})
  end
end
```

---

## สรุป Part 26

✅ **Step 281** - BEAM performance  
✅ **Step 282** - Profiling tools  
✅ **Step 283** - ETS performance  
✅ **Step 284** - Database optimization  
✅ **Step 285** - Connection pooling  
✅ **Step 286** - Process pooling  
✅ **Step 287** - Multi-level caching  
✅ **Step 288** - Memory optimization  
✅ **Step 289** - Concurrent processing  
✅ **Step 290** - Production monitoring  

➡️ [Part 27: Deployment และ DevOps](./part-27-deployment.md)
