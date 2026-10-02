# Part 83: Performance Optimization (Steps 891-910)

## Step 891: Profiling with :fprof

```elixir
defmodule MyApp.Profiler do
  def profile(fun) do
    :fprof.start()
    :fprof.trace(:start)
    result = fun.()
    :fprof.trace(:stop)
    :fprof.profile()
    :fprof.analyse(dest: :console, sort: :own)
    :fprof.stop()
    result
  end

  def eprof(fun, procs \\ [self()]) do
    :eprof.start()
    :eprof.start_profiling(procs)
    result = fun.()
    :eprof.stop_profiling()
    :eprof.analyse(:procs)
    :eprof.stop()
    result
  end

  # Memory usage
  def measure_memory(fun) do
    :erlang.garbage_collect()
    before = :erlang.memory(:total)
    result = fun.()
    :erlang.garbage_collect()
    after_mem = :erlang.memory(:total)
    %{result: result, memory_bytes: after_mem - before}
  end

  # Execution time
  def benchmark(name, fun, iterations \\ 1000) do
    times = Enum.map(1..iterations, fn _ ->
      start = System.monotonic_time(:microsecond)
      fun.()
      System.monotonic_time(:microsecond) - start
    end)

    %{
      name:    name,
      min_us:  Enum.min(times),
      max_us:  Enum.max(times),
      avg_us:  Enum.sum(times) / iterations,
      p95_us:  percentile(times, 95),
      p99_us:  percentile(times, 99)
    }
  end

  defp percentile(data, p) do
    sorted = Enum.sort(data)
    index  = ceil(length(sorted) * p / 100) - 1
    Enum.at(sorted, max(0, index))
  end
end
```

---

## Step 892: Connection Pool Tuning

```elixir
# config/runtime.exs
pool_size = System.get_env("DB_POOL_SIZE") |> String.to_integer()

config :my_app, MyApp.Repo,
  url:                  System.fetch_env!("DATABASE_URL"),
  pool_size:            pool_size,
  queue_target:         50,    # ms to wait before checking queue_interval
  queue_interval:       1_000, # period to measure queue
  timeout:              15_000,
  idle_interval:        100,
  after_connect: fn conn ->
    Postgrex.query!(conn, "SET statement_timeout = '30s'", [])
    Postgrex.query!(conn, "SET lock_timeout = '5s'", [])
  end

# Tune for read-heavy vs write-heavy workloads
# Read replicas
defmodule MyApp.ReadRepo do
  use Ecto.Repo, otp_app: :my_app, adapter: Ecto.Adapters.Postgres
end

config :my_app, MyApp.ReadRepo,
  url:       System.fetch_env!("READ_REPLICA_URL"),
  pool_size: 20,
  readonly:  true

# Route reads to replica
defmodule MyApp.DataLoader do
  def load_products do
    MyApp.ReadRepo.all(Product)
  end

  def create_order(attrs) do
    # Writes still go to primary
    MyApp.Repo.insert(Order.changeset(%Order{}, attrs))
  end
end
```

---

## Step 893: ETS as Cache Layer

```elixir
defmodule MyApp.ProductCache do
  # High-performance in-memory cache using ETS

  @table :products_cache
  @ttl   300  # seconds

  def init do
    :ets.new(@table, [:named_table, :public, read_concurrency: true])
  end

  def get(product_id) do
    case :ets.lookup(@table, product_id) do
      [{_, product, expires}] when expires > System.os_time(:second) ->
        {:ok, product}
      [{_, _, _}] ->
        :ets.delete(@table, product_id)
        miss()
      [] ->
        miss()
    end
  end

  defp miss, do: {:error, :not_found}

  def put(product_id, product) do
    :ets.insert(@table, {product_id, product, System.os_time(:second) + @ttl})
  end

  def invalidate(product_id), do: :ets.delete(@table, product_id)

  def warm(product_ids) do
    products = MyApp.Repo.all(from p in Product, where: p.id in ^product_ids)
    Enum.each(products, fn p -> put(p.id, p) end)
    length(products)
  end

  def get_or_load(product_id) do
    case get(product_id) do
      {:ok, product} -> product
      {:error, :not_found} ->
        product = MyApp.Repo.get!(Product, product_id)
        put(product_id, product)
        product
    end
  end
end
```

---

## Step 894: Query Optimization

```elixir
defmodule MyApp.QueryOptimizer do
  import Ecto.Query

  # Use select to load only needed columns
  def list_products_summary do
    from p in Product,
      select: %{id: p.id, name: p.name, price: p.price, stock: p.stock}
    |> Repo.all()
  end

  # Preload to avoid N+1
  def orders_with_items do
    Order
    |> preload(items: :product)  # single query per association
    |> Repo.all()
  end

  # join vs preload: join when filtering on association
  def orders_for_product(product_id) do
    from o in Order,
      join: i in assoc(o, :items),
      where: i.product_id == ^product_id,
      preload: [items: i],
      distinct: true
    |> Repo.all()
  end

  # Batch loading with Dataloader
  def load_with_dataloader(order_ids) do
    loader = Dataloader.new(MyApp.Repo)
      |> Dataloader.add_source(:orders, Dataloader.Ecto.new(MyApp.Repo))

    orders = Dataloader.load_many(loader, :orders, Order, order_ids)
    Dataloader.run(loader)
    Dataloader.get_many(loader, :orders, Order, order_ids)
  end

  # Use explain to see query plan
  def explain(query) do
    {sql, params} = Ecto.Adapters.SQL.to_sql(:all, Repo, query)
    Repo.query!("EXPLAIN ANALYZE " <> sql, params)
  end
end
```

---

## Step 895: Async Patterns

```elixir
defmodule MyApp.AsyncPatterns do
  # Run multiple independent operations in parallel

  def load_dashboard(user_id) do
    tasks = %{
      orders:    Task.async(fn -> MyApp.Orders.recent(user_id) end),
      activity:  Task.async(fn -> MyApp.Activity.feed(user_id) end),
      stats:     Task.async(fn -> MyApp.Stats.user_stats(user_id) end),
      notifs:    Task.async(fn -> MyApp.Notifications.unread(user_id) end)
    }

    Map.new(tasks, fn {key, task} ->
      result = Task.await(task, 5_000)
      {key, result}
    end)
  rescue
    e ->
      Logger.error("Dashboard load failed: #{inspect(e)}")
      %{orders: [], activity: [], stats: %{}, notifs: []}
  end

  # Process list concurrently with controlled concurrency
  def process_batch(items, fun, concurrency \\ 10) do
    items
    |> Task.async_stream(fun,
         max_concurrency: concurrency,
         timeout:         30_000,
         on_timeout:      :kill_task
       )
    |> Enum.reduce({[], []}, fn
      {:ok, result},   {ok, err} -> {[result | ok], err}
      {:error, reason}, {ok, err} -> {ok, [{reason, :error} | err]}
      {:exit, reason},  {ok, err} -> {ok, [{reason, :timeout} | err]}
    end)
    |> then(fn {ok, err} -> {Enum.reverse(ok), Enum.reverse(err)} end)
  end
end
```

---

## Step 896: Memory Optimization

```elixir
defmodule MyApp.MemoryOptimizer do
  # Binary optimization: reuse binaries

  # Bad: creates many copies
  def concat_bad(list) do
    Enum.reduce(list, "", fn s, acc -> acc <> s end)
  end

  # Good: IO list for efficient binary building
  def concat_good(list) do
    list |> Enum.intersperse(", ") |> IO.iodata_to_binary()
  end

  # Stream large data instead of loading all in memory
  def export_users_csv do
    Stream.resource(
      fn -> {0, 100} end,
      fn {offset, limit} ->
        users = Repo.all(from u in User, limit: ^limit, offset: ^offset)
        if Enum.empty?(users) do
          {:halt, nil}
        else
          rows = Enum.map(users, &to_csv_row/1)
          {rows, {offset + limit, limit}}
        end
      end,
      fn _ -> :ok end
    )
  end

  # Avoid large process dictionaries
  # Bad: storing large state in GenServer
  # Good: use ETS for large data, keep GenServer state small

  # Force GC when processing large data
  def process_and_gc(fun) do
    result = fun.()
    :erlang.garbage_collect()
    result
  end

  defp to_csv_row(user), do: "#{user.id},#{user.name},#{user.email}\n"
end
```

---

## Step 897: Database Indexing

```elixir
defmodule MyApp.Repo.Migrations.AddPerformanceIndexes do
  use Ecto.Migration

  def change do
    # Partial index: index only active users
    create index(:users, [:email], where: "deleted_at IS NULL", unique: true)

    # Composite index for common query patterns
    create index(:orders, [:customer_id, :status, :inserted_at])

    # Expression index for case-insensitive search
    create index(:users, ["lower(email)"],
      name: :users_email_lower_index, unique: true)

    # GIN index for JSONB
    create index(:products, [:attributes],
      using: :gin, name: :products_attributes_gin)

    # BRIN index for time-series data (very small index)
    create index(:events, [:occurred_at],
      using: :brin, name: :events_occurred_at_brin)

    # Covering index: include extra columns to avoid table access
    execute """
    CREATE INDEX orders_customer_covering
    ON orders(customer_id, status)
    INCLUDE (total, inserted_at);
    """, "DROP INDEX IF EXISTS orders_customer_covering;"
  end
end
```

---

## Step 898: Request Pipeline Optimization

```elixir
defmodule MyApp.RequestOptimizer do
  # Compress responses
  # In endpoint.ex
  plug Plug.Static, gzip: true, at: "/", from: :my_app

  # ETag support
  def with_etag(conn, data) do
    etag = :crypto.hash(:md5, :erlang.term_to_binary(data)) |> Base.encode16(case: :lower)

    if get_req_header(conn, "if-none-match") == [etag] do
      conn |> send_resp(304, "") |> halt()
    else
      conn
      |> put_resp_header("etag", etag)
      |> json(data)
    end
  end

  # Cache-control headers
  def cache_for(conn, seconds) do
    conn
    |> put_resp_header("cache-control", "public, max-age=#{seconds}")
    |> put_resp_header("vary", "accept-encoding")
  end

  # Response compression
  plug Plug.Telemetry, event_prefix: [:phoenix, :endpoint]
  plug Plug.Parsers, parsers: [:urlencoded, :multipart, :json],
    pass: ["*/*"],
    json_decoder: Phoenix.json_library()
end
```

---

## Step 899: BEAM Performance Tuning

```elixir
# vm.args / config/vm.args.eex
# +P 1048576          # max processes
# +Q 65536            # max ports
# +A 16               # async threads
# +K true             # kernel polling
# +W w                # warning mode
# -env ERL_MAX_ETS_TABLES 50000
# +sbwt very_short    # scheduler busy wait threshold
# +swt very_low       # scheduler wakeup threshold
# -heart              # watchdog

defmodule MyApp.VMTuner do
  def system_info do
    %{
      processes:        :erlang.system_info(:process_count),
      max_processes:    :erlang.system_info(:process_limit),
      ports:            :erlang.system_info(:port_count),
      ets_tables:       length(:ets.all()),
      schedulers:       :erlang.system_info(:schedulers),
      memory:           :erlang.memory(),
      run_queue:        :erlang.statistics(:run_queue),
      garbage_collections: :erlang.statistics(:garbage_collection) |> elem(0)
    }
  end

  def check_health do
    info = system_info()
    
    %{
      process_usage:  info.processes / info.max_processes * 100,
      memory_mb:      info.memory[:total] / 1_048_576,
      run_queue:      info.run_queue,
      ok:             info.processes / info.max_processes < 0.8 and info.run_queue < 100
    }
  end
end
```

---

## Step 900: Caching Strategies

```elixir
defmodule MyApp.CacheStrategy do
  # Cache-aside pattern
  def cache_aside(key, fun, ttl \\ 3600) do
    case Redix.command(:redix, ["GET", key]) do
      {:ok, nil} ->
        value = fun.()
        Redix.command(:redix, ["SETEX", key, ttl, Jason.encode!(value)])
        value
      {:ok, cached} ->
        Jason.decode!(cached)
    end
  end

  # Write-through: update cache on every write
  def write_through(key, value, persist_fn, ttl \\ 3600) do
    with {:ok, result} <- persist_fn.(value) do
      Redix.command(:redix, ["SETEX", key, ttl, Jason.encode!(value)])
      {:ok, result}
    end
  end

  # Write-behind: write to cache first, async to DB
  def write_behind(key, value, persist_fn, ttl \\ 3600) do
    Redix.command(:redix, ["SETEX", key, ttl, Jason.encode!(value)])
    Task.start(fn ->
      case persist_fn.(value) do
        {:ok, _} -> :ok
        {:error, reason} ->
          Logger.error("Write-behind failed for #{key}: #{inspect(reason)}")
          # Could add to retry queue here
      end
    end)
    :ok
  end

  # Stampede prevention
  def get_with_lock(key, fun, ttl \\ 3600) do
    lock_key = "lock:#{key}"
    
    case Redix.command(:redix, ["GET", key]) do
      {:ok, nil} ->
        # Try to acquire lock
        case Redix.command(:redix, ["SET", lock_key, "1", "NX", "EX", "10"]) do
          {:ok, "OK"} ->
            # We have the lock, compute value
            value = fun.()
            Redix.command(:redix, ["SETEX", key, ttl, Jason.encode!(value)])
            Redix.command(:redix, ["DEL", lock_key])
            value
          {:ok, nil} ->
            # Another process is computing, wait briefly
            Process.sleep(100)
            get_with_lock(key, fun, ttl)
        end
      {:ok, cached} ->
        Jason.decode!(cached)
    end
  end
end
```

---

## สรุป Part 83

✅ **Step 891** - Profiling  
✅ **Step 892** - Connection pool tuning  
✅ **Step 893** - ETS cache layer  
✅ **Step 894** - Query optimization  
✅ **Step 895** - Async patterns  
✅ **Step 896** - Memory optimization  
✅ **Step 897** - Database indexing  
✅ **Step 898** - Request pipeline optimization  
✅ **Step 899** - BEAM performance  
✅ **Step 900** - Caching strategies  

➡️ [Part 84: Real-Time Features](./part-84-realtime.md)
