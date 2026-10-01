# Part 13: Agent และ Task (Steps 141-160)

## Step 141: Agent - Simple State Abstraction

```elixir
# Agent เป็น abstraction บน GenServer สำหรับ state ง่ายๆ

# เริ่ม Agent
{:ok, agent} = Agent.start_link(fn -> %{count: 0} end)

# อ่าน state
Agent.get(agent, fn state -> state.count end)  # 0

# อัพเดต state
Agent.update(agent, fn state -> %{state | count: state.count + 1} end)

# get_and_update
Agent.get_and_update(agent, fn state ->
  {state.count, %{state | count: state.count + 1}}
end)

# หยุด
Agent.stop(agent)

# Agent กับ name
{:ok, _} = Agent.start_link(fn -> [] end, name: :my_list)
Agent.update(:my_list, fn list -> [:new_item | list] end)
Agent.get(:my_list, & &1)  # [:new_item]
```

---

## Step 142: Agent สำหรับ Global Configuration

```elixir
defmodule AppConfig do
  use Agent
  
  @default_config %{
    max_connections: 100,
    timeout: 5000,
    debug: false,
    features: []
  }
  
  def start_link(opts \\ []) do
    initial = Keyword.get(opts, :config, @default_config)
    Agent.start_link(fn -> initial end, name: __MODULE__)
  end
  
  def get(key) do
    Agent.get(__MODULE__, &Map.get(&1, key))
  end
  
  def get_all do
    Agent.get(__MODULE__, & &1)
  end
  
  def set(key, value) do
    Agent.update(__MODULE__, &Map.put(&1, key, value))
  end
  
  def update(key, func) do
    Agent.update(__MODULE__, fn config ->
      Map.update!(config, key, func)
    end)
  end
  
  def reset do
    Agent.update(__MODULE__, fn _ -> @default_config end)
  end
  
  def enable_feature(feature) do
    update(:features, fn features ->
      if feature in features, do: features, else: [feature | features]
    end)
  end
  
  def feature_enabled?(feature) do
    get(:features) |> Enum.member?(feature)
  end
end

# การใช้งาน
{:ok, _} = AppConfig.start_link()

AppConfig.get(:timeout)       # 5000
AppConfig.set(:debug, true)
AppConfig.enable_feature(:dark_mode)
AppConfig.feature_enabled?(:dark_mode)  # true
```

---

## Step 143: Agent vs GenServer

```elixir
# ใช้ Agent เมื่อ:
# - State management ง่ายๆ (get/update)
# - ไม่ต้องการ custom callbacks
# - Logic อยู่ใน caller (ไม่ใช่ server)

defmodule SimpleCounter do
  use Agent
  
  def start_link(init \\ 0) do
    Agent.start_link(fn -> init end, name: __MODULE__)
  end
  
  def increment(by \\ 1) do
    Agent.update(__MODULE__, &(&1 + by))
  end
  
  def value do
    Agent.get(__MODULE__, & &1)
  end
  
  def reset, do: Agent.update(__MODULE__, fn _ -> 0 end)
end

# ใช้ GenServer เมื่อ:
# - ต้องการ complex business logic ใน server
# - ต้องการ handle multiple message types
# - ต้องการ side effects (timers, subscriptions)
# - ต้องการ proper error handling

defmodule SmartCounter do
  use GenServer
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def increment(by \\ 1) do
    GenServer.call(__MODULE__, {:increment, by})
  end
  
  @impl true
  def init(opts) do
    max = Keyword.get(opts, :max, :infinity)
    {:ok, %{value: 0, max: max, overflow_count: 0}}
  end
  
  @impl true
  def handle_call({:increment, by}, _from, %{value: v, max: max} = state) do
    new_value = v + by
    
    if max != :infinity && new_value > max do
      {:reply, {:error, :overflow}, %{state | overflow_count: state.overflow_count + 1}}
    else
      {:reply, {:ok, new_value}, %{state | value: new_value}}
    end
  end
end
```

---

## Step 144: Task - Concurrent Work

```elixir
# Task สำหรับทำงาน concurrent แบบง่าย

# Task.async + await
task = Task.async(fn ->
  :timer.sleep(1000)
  "result after 1 second"
end)

# ทำงานอื่นระหว่างรอ
IO.puts("Doing other work...")

# รอผล
result = Task.await(task)  # blocks up to 5000ms
IO.puts("Got: #{result}")

# Custom timeout
result = Task.await(task, 10_000)

# Task.await_many - รอหลาย tasks พร้อมกัน
tasks = [
  Task.async(fn -> fetch_user(1) end),
  Task.async(fn -> fetch_user(2) end),
  Task.async(fn -> fetch_user(3) end)
]

results = Task.await_many(tasks, 5000)
# [user1, user2, user3]
```

---

## Step 145: Task.async_stream

```elixir
# Task.async_stream - parallel processing สำหรับ collections

urls = ["http://api1.com", "http://api2.com", "http://api3.com"]

# Process ทุก URL พร้อมกัน (max 5 concurrent)
results = urls
  |> Task.async_stream(&fetch_url/1,
       max_concurrency: 5,
       timeout: 10_000,
       on_timeout: :kill_task)
  |> Enum.to_list()

# Results อยู่ใน {:ok, value} | {:exit, reason}
successes = Enum.filter(results, &match?({:ok, _}, &1))
failures  = Enum.filter(results, &match?({:exit, _}, &1))

# แปลง data พร้อมกัน
defmodule DataProcessor do
  def process_batch(items, opts \\ []) do
    max_concurrency = Keyword.get(opts, :max_concurrency, System.schedulers_online())
    
    items
    |> Task.async_stream(&process_item/1,
         max_concurrency: max_concurrency,
         timeout: 30_000)
    |> Stream.filter(&match?({:ok, _}, &1))
    |> Stream.map(fn {:ok, result} -> result end)
    |> Enum.to_list()
  end
  
  defp process_item(item) do
    # Transform the item
    %{
      id: item.id,
      processed: true,
      result: expensive_transform(item.data)
    }
  end
  
  defp expensive_transform(data) do
    Process.sleep(10)  # simulate work
    String.upcase(data)
  end
end
```

---

## Step 146: Task.Supervisor

```elixir
defmodule JobRunner do
  def start_link(opts) do
    Task.Supervisor.start_link([{:name, __MODULE__} | opts])
  end
  
  # Fire-and-forget สำหรับ background jobs
  def run_async(func) do
    Task.Supervisor.start_child(__MODULE__, func)
  end
  
  # Monitored task
  def run_monitored(func) do
    Task.Supervisor.async(__MODULE__, func)
  end
  
  # Stream กับ supervision
  def process_stream(items, func) do
    Task.Supervisor.async_stream(__MODULE__, items, func,
      max_concurrency: 10,
      timeout: 30_000
    )
  end
end

# Application start
children = [
  {Task.Supervisor, name: JobRunner},
  # ... other children
]

# Usage
JobRunner.run_async(fn ->
  IO.puts("Running in background!")
  :timer.sleep(1000)
  IO.puts("Done!")
end)

task = JobRunner.run_monitored(fn -> do_important_work() end)
result = Task.await(task)
```

---

## Step 147: Task กับ Error Handling

```elixir
defmodule SafeTask do
  # Task ที่ handle errors gracefully
  
  def run(func, opts \\ []) do
    timeout = Keyword.get(opts, :timeout, 5000)
    
    task = Task.async(func)
    
    try do
      {:ok, Task.await(task, timeout)}
    catch
      :exit, {:timeout, _} ->
        Task.shutdown(task, :brutal_kill)
        {:error, :timeout}
      
      :exit, reason ->
        {:error, reason}
    end
  end
  
  # Run หลาย tasks แล้ว collect results
  def run_all(funcs, opts \\ []) do
    timeout = Keyword.get(opts, :timeout, 5000)
    
    tasks = Enum.map(funcs, &Task.async/1)
    
    Enum.map(tasks, fn task ->
      try do
        {:ok, Task.await(task, timeout)}
      catch
        :exit, {:timeout, _} ->
          Task.shutdown(task, :brutal_kill)
          {:error, :timeout}
        :exit, reason ->
          {:error, reason}
      end
    end)
  end
end

# Example: fetch multiple APIs with timeout
results = SafeTask.run_all([
  fn -> HTTP.get("/api/users") end,
  fn -> HTTP.get("/api/products") end,
  fn -> HTTP.get("/api/orders") end
], timeout: 3000)

Enum.each(results, fn
  {:ok, data}        -> IO.inspect(data)
  {:error, :timeout} -> IO.puts("Request timed out")
  {:error, reason}   -> IO.puts("Error: #{inspect(reason)}")
end)
```

---

## Step 148: Combining Agent, GenServer, and Task

```elixir
defmodule DataPipeline do
  @moduledoc """
  Pipeline ที่ใช้ Agent สำหรับ state, Task สำหรับ parallel work
  """
  
  use GenServer
  
  defstruct [:source, :transforms, :sink, :status, :stats]
  
  def start_link(config) do
    GenServer.start_link(__MODULE__, config, name: __MODULE__)
  end
  
  def process(items) do
    GenServer.call(__MODULE__, {:process, items}, 60_000)
  end
  
  def stats do
    GenServer.call(__MODULE__, :stats)
  end
  
  @impl true
  def init(config) do
    {:ok, stats_agent} = Agent.start_link(fn ->
      %{processed: 0, failed: 0, total_time: 0}
    end)
    
    state = %{
      config: config,
      stats_agent: stats_agent,
      supervisor: nil
    }
    
    {:ok, sup} = Task.Supervisor.start_link()
    {:ok, %{state | supervisor: sup}}
  end
  
  @impl true
  def handle_call({:process, items}, _from, state) do
    start_time = System.monotonic_time(:millisecond)
    
    results = items
      |> Task.async_stream(&process_item(&1, state.config),
           supervisor: state.supervisor,
           max_concurrency: 10,
           timeout: 30_000)
      |> Enum.reduce({[], []}, fn
           {:ok, {:ok, result}}, {ok, err}    -> {[result | ok], err}
           {:ok, {:error, _}},  {ok, err}     -> {ok, [items | err]}
           {:exit, _},          {ok, err}     -> {ok, err}
         end)
    
    elapsed = System.monotonic_time(:millisecond) - start_time
    {successes, failures} = results
    
    Agent.update(state.stats_agent, fn stats ->
      %{stats |
        processed: stats.processed + length(successes),
        failed: stats.failed + length(failures),
        total_time: stats.total_time + elapsed
      }
    end)
    
    {:reply, {:ok, Enum.reverse(successes)}, state}
  end
  
  @impl true
  def handle_call(:stats, _from, state) do
    stats = Agent.get(state.stats_agent, & &1)
    {:reply, stats, state}
  end
  
  defp process_item(item, config) do
    Enum.reduce_while(config.transforms, {:ok, item}, fn transform, {:ok, current} ->
      case transform.(current) do
        {:ok, result} -> {:cont, {:ok, result}}
        {:error, _} = err -> {:halt, err}
      end
    end)
  end
end
```

---

## Step 149: Task.yield vs Task.await

```elixir
defmodule ProgressTracker do
  # Task.yield ไม่ raise exception ถ้า timeout
  # ดีสำหรับ polling progress
  
  def run_with_progress(func) do
    task = Task.async(func)
    check_progress(task, 0)
  end
  
  defp check_progress(task, attempt) do
    case Task.yield(task, 500) do  # รอ 500ms
      {:ok, result} ->
        IO.puts("\nDone!")
        result
      
      nil ->
        # ยังทำงานอยู่
        IO.write(".")
        
        if attempt >= 20 do
          # Timeout หลังจาก 20 * 500ms = 10 seconds
          Task.shutdown(task)
          {:error, :timeout}
        else
          check_progress(task, attempt + 1)
        end
      
      {:exit, reason} ->
        {:error, reason}
    end
  end
end

# Usage
result = ProgressTracker.run_with_progress(fn ->
  :timer.sleep(2000)
  "done"
end)
# Output: ....
# Done!
```

---

## Step 150: Complete Example - Parallel Web Scraper

```elixir
defmodule WebScraper do
  use GenServer
  
  defstruct [:visited, :queue, :results, :max_depth]
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def scrape(url, opts \\ []) do
    GenServer.call(__MODULE__, {:scrape, url, opts}, 60_000)
  end
  
  @impl true
  def init(opts) do
    max_concurrent = Keyword.get(opts, :max_concurrent, 10)
    
    {:ok, task_sup} = Task.Supervisor.start_link()
    
    {:ok, %{
      task_supervisor: task_sup,
      max_concurrent: max_concurrent,
      results: %{}
    }}
  end
  
  @impl true
  def handle_call({:scrape, start_url, opts}, _from, state) do
    max_depth = Keyword.get(opts, :max_depth, 2)
    
    results = scrape_recursive([{start_url, 0}], MapSet.new(), %{}, max_depth, state)
    
    {:reply, {:ok, results}, state}
  end
  
  defp scrape_recursive([], _visited, results, _max_depth, _state), do: results
  
  defp scrape_recursive(urls, visited, results, max_depth, state) do
    # Filter already visited
    {to_fetch, _} = Enum.split_with(urls, fn {url, _depth} ->
      not MapSet.member?(visited, url)
    end)
    
    if Enum.empty?(to_fetch), do: results
    
    # Fetch all in parallel
    fetched = to_fetch
      |> Task.async_stream(fn {url, depth} ->
           case fetch_page(url) do
             {:ok, content} ->
               links = extract_links(content, url)
               {:ok, url, depth, content, links}
             {:error, reason} ->
               {:error, url, reason}
           end
         end,
         supervisor: state.task_supervisor,
         max_concurrency: state.max_concurrent,
         timeout: 10_000)
      |> Enum.to_list()
    
    new_visited = Enum.reduce(to_fetch, visited, fn {url, _}, acc ->
      MapSet.put(acc, url)
    end)
    
    new_results = Enum.reduce(fetched, results, fn
      {:ok, {:ok, url, _depth, content, _links}}, acc ->
        Map.put(acc, url, content)
      _, acc -> acc
    end)
    
    # Collect next URLs to visit
    next_urls = fetched
      |> Enum.flat_map(fn
           {:ok, {:ok, _url, depth, _content, links}} when depth < max_depth ->
             Enum.map(links, &{&1, depth + 1})
           _ -> []
         end)
      |> Enum.reject(fn {url, _} -> MapSet.member?(new_visited, url) end)
    
    scrape_recursive(next_urls, new_visited, new_results, max_depth, state)
  end
  
  defp fetch_page(url) do
    # Simulated fetch
    {:ok, "<html>Content of #{url}</html>"}
  end
  
  defp extract_links(html, base_url) do
    # Simulated link extraction
    Regex.scan(~r/href="([^"]+)"/, html, capture: :all_but_first)
    |> List.flatten()
    |> Enum.map(&resolve_url(&1, base_url))
  end
  
  defp resolve_url(url, base) do
    if String.starts_with?(url, "http"), do: url, else: "#{base}#{url}"
  end
end
```

---

## สรุป Part 13

✅ **Step 141** - Agent basics  
✅ **Step 142** - AppConfig ด้วย Agent  
✅ **Step 143** - Agent vs GenServer  
✅ **Step 144** - Task.async/await  
✅ **Step 145** - Task.async_stream  
✅ **Step 146** - Task.Supervisor  
✅ **Step 147** - Error handling ใน Tasks  
✅ **Step 148** - Combining Agent + GenServer + Task  
✅ **Step 149** - Task.yield สำหรับ progress  
✅ **Step 150** - Web Scraper example  

➡️ [Part 14: ETS และ Mnesia](./part-14-ets-mnesia.md)
