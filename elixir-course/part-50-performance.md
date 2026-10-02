# Part 50: Performance Deep Dive (Steps 551-570)

## Step 551: Memory Profiling

```elixir
defmodule MyApp.MemoryProfiler do
  def snapshot do
    :erlang.memory()
    |> Enum.map(fn {type, bytes} ->
      {type, format_bytes(bytes)}
    end)
    |> Map.new()
  end

  def top_processes(n \\ 20) do
    Process.list()
    |> Enum.map(fn pid ->
      info = Process.info(pid, [:memory, :message_queue_len, :registered_name, :current_function])
      {pid, info}
    end)
    |> Enum.sort_by(fn {_, info} -> info[:memory] end, :desc)
    |> Enum.take(n)
  end

  def find_memory_leaks do
    before = :erlang.memory(:total)
    
    # Run your operation here
    :timer.sleep(5_000)
    
    after_mem = :erlang.memory(:total)
    diff = after_mem - before
    
    IO.puts("Memory change: #{format_bytes(diff)}")
    diff
  end

  # Force garbage collection on all processes
  def gc_all do
    Process.list()
    |> Enum.each(&:erlang.garbage_collect/1)
  end

  defp format_bytes(bytes) when bytes >= 1_048_576, do: "#{Float.round(bytes / 1_048_576, 2)} MB"
  defp format_bytes(bytes) when bytes >= 1_024,     do: "#{Float.round(bytes / 1_024, 2)} KB"
  defp format_bytes(bytes),                          do: "#{bytes} B"
end

# Check ETS table sizes
defmodule MyApp.ETSMonitor do
  def all_tables_info do
    :ets.all()
    |> Enum.map(fn t ->
      info = :ets.info(t)
      %{
        name:   info[:name],
        size:   info[:size],
        memory: info[:memory] * :erlang.system_info(:wordsize)
      }
    end)
    |> Enum.sort_by(& &1.memory, :desc)
  end
end
```

---

## Step 552: Scheduler Utilization

```elixir
defmodule MyApp.SchedulerMonitor do
  use GenServer

  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)

  def init(_) do
    :erlang.system_flag(:scheduler_wall_time, true)
    schedule_collect()
    {:ok, %{samples: []}}
  end

  def handle_info(:collect, %{samples: samples} = state) do
    sample = :erlang.statistics(:scheduler_wall_time)
    new_samples = [sample | Enum.take(samples, 9)]  # keep 10 samples
    
    if length(new_samples) >= 2 do
      utilization = calculate_utilization(Enum.at(new_samples, 1), hd(new_samples))
      :telemetry.execute([:scheduler, :utilization], %{value: utilization}, %{})
    end
    
    schedule_collect()
    {:noreply, %{state | samples: new_samples}}
  end

  defp calculate_utilization(before, after_sample) do
    pairs = Enum.zip(before, after_sample)
    
    total = Enum.reduce(pairs, {0, 0}, fn {{_id, active_b, total_b}, {_id, active_a, total_a}}, {a_acc, t_acc} ->
      {a_acc + (active_a - active_b), t_acc + (total_a - total_b)}
    end)
    
    {active_diff, total_diff} = total
    if total_diff > 0, do: active_diff / total_diff * 100, else: 0.0
  end

  defp schedule_collect do
    Process.send_after(self(), :collect, 1_000)
  end
end
```

---

## Step 553: NimblePool

```elixir
defmodule MyApp.DatabasePool do
  @behaviour NimblePool

  def start_link(opts) do
    NimblePool.start_link(worker: {__MODULE__, opts}, pool_size: 10, name: __MODULE__)
  end

  def checkout(fun) do
    NimblePool.checkout!(__MODULE__, :checkout, fn _from, worker ->
      result = fun.(worker)
      {result, worker}
    end)
  end

  @impl NimblePool
  def init_worker(opts) do
    {:ok, conn} = DBConnection.connect(opts)
    {:ok, conn}
  end

  @impl NimblePool
  def handle_checkout(:checkout, _from, conn, _pool_state) do
    {:ok, conn, conn, _pool_state}
  end

  @impl NimblePool
  def handle_checkin(conn, _from, _old_conn, _pool_state) do
    {:ok, conn, _pool_state}
  end

  @impl NimblePool
  def terminate_worker(_reason, conn, _pool_state) do
    DBConnection.disconnect(conn, [])
    :ok
  end
end

# Custom HTTP connection pool
defmodule MyApp.HTTPPool do
  @behaviour NimblePool

  def start_link do
    NimblePool.start_link(worker: {__MODULE__, []}, pool_size: 20, name: __MODULE__)
  end

  def get(url) do
    NimblePool.checkout!(__MODULE__, :checkout, fn _from, conn ->
      result = :hackney.get(url, [], "", [{:connection, conn}])
      {result, conn}
    end)
  end

  @impl NimblePool
  def init_worker(_) do
    {:ok, conn} = :hackney_pool.connect("api.example.com", 443, [:ssl])
    {:ok, conn}
  end

  @impl NimblePool
  def handle_checkout(:checkout, _from, conn, pool_state) do
    {:ok, conn, conn, pool_state}
  end

  @impl NimblePool
  def handle_checkin(conn, _from, _old_conn, pool_state) do
    {:ok, conn, pool_state}
  end
end
```

---

## Step 554: Binary Optimization

```elixir
defmodule MyApp.BinaryOpts do
  # Binary matching is very fast in Elixir/Erlang
  def parse_csv_line(line) do
    parse_fields(line, [])
  end

  defp parse_fields(<<>>, acc), do: Enum.reverse(acc)
  defp parse_fields(<<?,, rest::binary>>, acc), do: parse_fields(rest, acc)
  defp parse_fields(binary, acc) do
    {field, rest} = parse_field(binary, <<>>)
    parse_fields(rest, [field | acc])
  end

  defp parse_field(<<?,, rest::binary>>, acc), do: {acc, rest}
  defp parse_field(<<>>, acc), do: {acc, <<>>}
  defp parse_field(<<c::utf8, rest::binary>>, acc), do: parse_field(rest, <<acc::binary, c::utf8>>)

  # Efficient binary building with iolist (avoid concatenation)
  def build_response(status, headers, body) do
    [
      "HTTP/1.1 ", Integer.to_string(status), "\r\n",
      Enum.map(headers, fn {k, v} -> [k, ": ", v, "\r\n"] end),
      "\r\n",
      body
    ]
    |> IO.iodata_to_binary()
  end

  # Binary pattern matching for protocol parsing
  def parse_frame(<<type::8, flags::8, length::24, payload::binary-size(length), rest::binary>>) do
    {:ok, %{type: type, flags: flags, payload: payload}, rest}
  end
  def parse_frame(data), do: {:incomplete, data}
end
```

---

## Step 555: Process Hibernation

```elixir
defmodule MyApp.HibernatingServer do
  use GenServer

  @hibernate_after 5_000  # ms of inactivity before hibernating

  def start_link(opts), do: GenServer.start_link(__MODULE__, opts, name: __MODULE__)

  def init(opts) do
    {:ok, %{data: nil, opts: opts}, @hibernate_after}
  end

  def handle_call(:get, _from, state) do
    {:reply, state.data, state, @hibernate_after}
  end

  def handle_call({:set, value}, _from, state) do
    {:reply, :ok, %{state | data: value}, @hibernate_after}
  end

  # Timeout fires when idle: hibernate to save memory
  def handle_info(:timeout, state) do
    {:noreply, state, :hibernate}
  end

  # After wakeup, reset the timeout
  def handle_info(msg, state) do
    process_message(msg, state)
    {:noreply, state, @hibernate_after}
  end

  defp process_message(_msg, _state), do: :ok
end

# Check if process is hibernating
defmodule MyApp.ProcessInfo do
  def hibernating?(pid) do
    case Process.info(pid, :current_function) do
      {:current_function, {:erlang, :hibernate, 3}} -> true
      _ -> false
    end
  end

  def memory_savings_from_hibernate(pid) do
    # A hibernated process compacts its heap
    Process.info(pid, :memory)
  end
end
```

---

## Step 556: ETS Advanced Patterns

```elixir
defmodule MyApp.ETSCache do
  @table :fast_cache

  def setup do
    :ets.new(@table, [
      :set,
      :public,
      :named_table,
      read_concurrency:   true,
      write_concurrency:  true,
      decentralized_counters: true
    ])
  end

  # Atomic compare-and-swap
  def cas(key, expected, new_value) do
    :ets.select_replace(@table, [
      {{key, :"$1"}, [{:==, :"$1", expected}], [{{key, new_value}}]}
    ])
  end

  # Batch write
  def insert_many(pairs) do
    :ets.insert(@table, pairs)
  end

  # Efficient pattern matching select
  def find_by_prefix(prefix) do
    match = {{:"$1", :_}, [{:>=, :"$1", prefix}, {:<, :"$1", prefix <> "\xFF"}], [:"$1"]}
    :ets.select(@table, [match])
  end

  # Ordered set for range queries
  def range_query(min_key, max_key) do
    :ets.select(:ordered_cache, [
      {{:"$1", :"$2"},
       [{:>=, :"$1", min_key}, {:"=<", :"$1", max_key}],
       [{{:"$1", :"$2"}}]}
    ])
  end

  # Counter with decentralized_counters for high write throughput
  def increment_counter(key, amount \\ 1) do
    try do
      :ets.update_counter(@table, key, amount)
    rescue
      ArgumentError ->
        :ets.insert_new(@table, {key, amount})
        amount
    end
  end
end
```

---

## Step 557: Flow (Parallel Data Processing)

```elixir
# mix.exs: {:flow, "~> 1.2"}

defmodule MyApp.DataProcessor do
  alias Experimental.Flow

  def process_large_file(path) do
    path
    |> File.stream!(read_ahead: 100_000)
    |> Flow.from_enumerable(max_demand: 1000)
    |> Flow.map(&String.trim/1)
    |> Flow.filter(&String.length(&1) > 0)
    |> Flow.map(&parse_line/1)
    |> Flow.partition(key: fn item -> item.category end)
    |> Flow.group_by(& &1.category)
    |> Flow.map(fn {cat, items} -> {cat, aggregate(items)} end)
    |> Enum.to_list()
  end

  def count_words_parallel(documents) do
    documents
    |> Flow.from_enumerable(stages: System.schedulers_online())
    |> Flow.flat_map(&String.split/1)
    |> Flow.partition()
    |> Flow.reduce(fn -> %{} end, fn word, acc ->
      Map.update(acc, word, 1, &(&1 + 1))
    end)
    |> Flow.emit(:state)
    |> Enum.reduce(%{}, &Map.merge(&1, &2, fn _k, a, b -> a + b end))
  end

  defp parse_line(line) do
    [category | fields] = String.split(line, ",")
    %{category: category, fields: fields}
  end

  defp aggregate(items) do
    %{count: length(items), items: items}
  end
end
```

---

## Step 558: Benchmarking with Benchee

```elixir
# mix.exs: {:benchee, "~> 1.3", only: :dev}

defmodule MyApp.Benchmarks do
  def run_all do
    Benchee.run(
      %{
        "Enum.map"      => fn -> Enum.map(1..10_000, &(&1 * 2)) end,
        "for comprehension" => fn -> for x <- 1..10_000, do: x * 2 end,
        "Stream.map"    => fn -> Stream.map(1..10_000, &(&1 * 2)) |> Enum.to_list() end
      },
      time:   5,
      warmup: 2,
      memory_time: 2,
      formatters: [
        Benchee.Formatters.Console,
        {Benchee.Formatters.HTML, file: "bench/results.html"}
      ]
    )
  end

  def compare_map_implementations do
    data = Map.new(1..1000, fn i -> {i, i * i} end)
    
    Benchee.run(%{
      "Map.get"          => fn -> Map.get(data, 500) end,
      ":maps.get"        => fn -> :maps.get(500, data) end,
      "Access []"        => fn -> data[500] end,
      "pattern match"    => fn -> %{500 => val} = data; val end
    })
  end

  def db_query_benchmark do
    Benchee.run(%{
      "single query"   => fn -> MyApp.Repo.get(User, 1) end,
      "preloaded"      => fn -> MyApp.Repo.get(User, 1) |> MyApp.Repo.preload(:posts) end,
      "cached"         => fn -> MyApp.Cache.get_or_fetch("user:1", fn -> MyApp.Repo.get(User, 1) end) end
    }, time: 10)
  end
end
```

---

## Step 559: :persistent_term

```elixir
defmodule MyApp.Config do
  # :persistent_term is faster than ETS for read-heavy config
  # Reads are zero-copy (shares memory with all processes)
  # Writes are expensive (GC all processes), use rarely

  def setup do
    config = Application.get_all_env(:my_app)
    :persistent_term.put({__MODULE__, :config}, config)
  end

  def get(key) do
    __MODULE__
    |> config()
    |> Keyword.get(key)
  end

  def config do
    :persistent_term.get({__MODULE__, :config})
  end

  # Store compiled regex patterns (immutable, frequently read)
  def compile_patterns do
    patterns = %{
      email:   ~r/^[^\s@]+@[^\s@]+\.[^\s@]+$/,
      phone:   ~r/^\+?[\d\s\-\(\)]{7,20}$/,
      url:     ~r/^https?:\/\/.+/
    }
    :persistent_term.put({__MODULE__, :patterns}, patterns)
  end

  def pattern(name) do
    :persistent_term.get({__MODULE__, :patterns}) |> Map.get(name)
  end

  # List all persistent terms
  def info do
    :persistent_term.info()
  end
end
```

---

## Step 560: Profiling in Production

```elixir
defmodule MyApp.ProdProfiler do
  # Safe profiling in production - time-bounded, minimal overhead

  def sample_profile(duration_ms \\ 5_000) do
    pid = spawn(fn ->
      :eprof.start()
      :eprof.start_profiling(Process.list())
      Process.sleep(duration_ms)
      :eprof.stop_profiling()
      :eprof.analyse(:total)
      :eprof.stop()
    end)
    
    ref = Process.monitor(pid)
    receive do
      {:DOWN, ^ref, :process, ^pid, _} -> :done
    after duration_ms + 1000 -> :timeout
    end
  end

  # Recon library (add to mix.exs in prod)
  def top_processes_by_reductions do
    # :recon.proc_count(:reductions, 10)
    Process.list()
    |> Enum.map(fn p ->
      {:reductions, r} = Process.info(p, :reductions)
      {p, r}
    end)
    |> Enum.sort_by(&elem(&1, 1), :desc)
    |> Enum.take(10)
  end

  def check_message_queues(threshold \\ 100) do
    Process.list()
    |> Enum.filter(fn p ->
      case Process.info(p, :message_queue_len) do
        {:message_queue_len, n} when n > threshold -> true
        _ -> false
      end
    end)
    |> Enum.map(fn p -> {p, Process.info(p, [:message_queue_len, :registered_name])} end)
  end
end
```

---

## สรุป Part 50

✅ **Step 551** - Memory profiling  
✅ **Step 552** - Scheduler utilization  
✅ **Step 553** - NimblePool  
✅ **Step 554** - Binary optimization  
✅ **Step 555** - Process hibernation  
✅ **Step 556** - ETS advanced patterns  
✅ **Step 557** - Flow parallel processing  
✅ **Step 558** - Benchmarking  
✅ **Step 559** - :persistent_term  
✅ **Step 560** - Production profiling  

➡️ [Part 51: LiveView Advanced](./part-51-liveview-advanced.md)
