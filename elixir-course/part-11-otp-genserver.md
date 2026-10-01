# Part 11: OTP และ GenServer (Steps 101-120)

> OTP (Open Telecom Platform) คือ framework สำหรับสร้าง fault-tolerant distributed systems
> GenServer คือ behaviour ที่ abstract การสร้าง stateful server process

## Step 101: OTP คืออะไร?

### 101.1 OTP Components

```
OTP = Open Telecom Platform
┌─────────────────────────────────────┐
│              OTP                     │
│  ┌──────────┐  ┌───────────────┐   │
│  │GenServer │  │  Supervisor   │   │
│  │(Stateful │  │  (Fault       │   │
│  │ Server)  │  │  Tolerance)   │   │
│  └──────────┘  └───────────────┘   │
│  ┌──────────┐  ┌───────────────┐   │
│  │  Agent   │  │     Task      │   │
│  │(Simple   │  │  (Async       │   │
│  │ State)   │  │   Work)       │   │
│  └──────────┘  └───────────────┘   │
│  ┌──────────┐  ┌───────────────┐   │
│  │GenEvent  │  │  GenStateMachine│  │
│  │(Events)  │  │  (FSM)        │   │
│  └──────────┘  └───────────────┘   │
└─────────────────────────────────────┘
```

### 101.2 ทำไมต้องใช้ OTP?

```elixir
# โดยไม่ใช้ OTP - ต้อง implement เอง:
defmodule ManualServer do
  def loop(state) do
    receive do
      {:get, from, ref} ->
        send(from, {ref, state})
        loop(state)
      {:set, new_state} ->
        loop(new_state)
      # ต้อง handle errors เอง
      # ต้อง implement logging เอง
      # ต้อง implement timeouts เอง
      # ต้อง handle init/terminate เอง
      # ...
    end
  end
end

# ด้วย OTP GenServer - OTP ดูแลทั้งหมดให้:
defmodule OTPServer do
  use GenServer
  
  def init(initial_state), do: {:ok, initial_state}
  
  def handle_call(:get, _from, state), do: {:reply, state, state}
  def handle_cast({:set, new_state}, _state), do: {:noreply, new_state}
  
  # OTP ให้:
  # ✅ Automatic error handling
  # ✅ Built-in logging  
  # ✅ Timeout support
  # ✅ Proper init/terminate lifecycle
  # ✅ OTP-compliant interface
  # ✅ Tracing/debugging support
  # ✅ Hot code reloading
end
```

---

## Step 102: GenServer Basics

### 102.1 GenServer Structure

```elixir
defmodule MyServer do
  use GenServer
  
  # ───────── Client API ─────────
  
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, :ok, opts)
  end
  
  def get_state(server) do
    GenServer.call(server, :get_state)
  end
  
  def update_state(server, new_state) do
    GenServer.cast(server, {:update, new_state})
  end
  
  # ───────── Server Callbacks ─────────
  
  @impl true
  def init(:ok) do
    {:ok, %{data: [], count: 0}}  # initial state
  end
  
  @impl true
  def handle_call(:get_state, _from, state) do
    {:reply, state, state}  # {result_to_client, new_state}
  end
  
  @impl true
  def handle_cast({:update, new_data}, state) do
    new_state = %{state | data: [new_data | state.data], count: state.count + 1}
    {:noreply, new_state}  # no reply to client
  end
  
  @impl true
  def handle_info(:cleanup, state) do
    IO.puts("Running cleanup...")
    {:noreply, %{state | data: []}}
  end
  
  @impl true
  def terminate(reason, state) do
    IO.puts("Server terminating: #{inspect(reason)}")
    :ok
  end
end
```

### 102.2 start_link กับ Options

```elixir
# เริ่ม server หลายวิธี
{:ok, pid} = GenServer.start_link(MyServer, :ok)
{:ok, pid} = GenServer.start_link(MyServer, :ok, name: :my_server)
{:ok, pid} = GenServer.start_link(MyServer, :ok, name: {:global, :my_server})

# Timeout
{:ok, pid} = GenServer.start_link(MyServer, :ok, timeout: 10_000)

# เรียกใช้
GenServer.call(:my_server, :get_state)     # timeout 5000ms default
GenServer.call(:my_server, :get_state, 10_000)  # custom timeout
GenServer.cast(:my_server, {:update, "data"})   # async, no reply
```

---

## Step 103: GenServer Callbacks ลึกขึ้น

### 103.1 init/1

```elixir
@impl true
def init(args) do
  # Return options:
  {:ok, initial_state}
  {:ok, initial_state, timeout_ms}      # send :timeout after ms
  {:ok, initial_state, :hibernate}      # hibernate to save memory
  {:stop, reason}                       # fail to start
  :ignore                               # ignore start (no process created)
end

# init กับ side effects
@impl true
def init(config) do
  Process.flag(:trap_exit, true)  # จับ EXIT signals
  
  # ตั้ง timer
  :timer.send_interval(5000, self(), :heartbeat)
  
  # Subscribe to events
  EventBus.subscribe(:user_events)
  
  state = %{
    config: config,
    started_at: DateTime.utc_now(),
    request_count: 0
  }
  
  {:ok, state}
end
```

### 103.2 handle_call/3

```elixir
# handle_call ใช้สำหรับ synchronous requests ที่ต้องการ response
@impl true
def handle_call(message, from, state) do
  # from = {pid, ref} ของ caller
  
  # Return options:
  {:reply, reply, new_state}
  {:reply, reply, new_state, timeout_ms}
  {:reply, reply, new_state, :hibernate}
  {:noreply, new_state}                 # ตอบทีหลังด้วย GenServer.reply/2
  {:noreply, new_state, timeout_ms}
  {:stop, reason, reply, new_state}     # stop server แล้วตอบ
  {:stop, reason, new_state}            # stop server ไม่ตอบ
end

# ตัวอย่าง
@impl true
def handle_call({:get_user, id}, _from, state) do
  case Map.get(state.users, id) do
    nil  -> {:reply, {:error, :not_found}, state}
    user -> {:reply, {:ok, user}, state}
  end
end

# Async reply (ส่ง reply ทีหลัง)
@impl true
def handle_call({:slow_operation, data}, from, state) do
  Task.start(fn ->
    result = do_slow_work(data)
    GenServer.reply(from, result)
  end)
  {:noreply, state}  # ยังไม่ตอบตอนนี้
end
```

### 103.3 handle_cast/2

```elixir
# handle_cast ใช้สำหรับ asynchronous requests (fire and forget)
@impl true
def handle_cast(message, state) do
  # Return options:
  {:noreply, new_state}
  {:noreply, new_state, timeout_ms}
  {:noreply, new_state, :hibernate}
  {:stop, reason, new_state}
end

@impl true
def handle_cast({:log, level, message}, state) do
  entry = %{level: level, message: message, at: DateTime.utc_now()}
  new_state = %{state | logs: [entry | state.logs]}
  {:noreply, new_state}
end
```

### 103.4 handle_info/2

```elixir
# handle_info รับ messages ที่ส่งด้วย send/2 โดยตรง
@impl true
def handle_info(:heartbeat, state) do
  IO.puts("Server alive, uptime: #{uptime(state)}")
  {:noreply, %{state | heartbeat_count: state.heartbeat_count + 1}}
end

@impl true
def handle_info({:DOWN, ref, :process, pid, reason}, state) do
  # Handle monitored process going down
  IO.puts("Monitored process #{inspect(pid)} down: #{inspect(reason)}")
  {:noreply, state}
end

@impl true
def handle_info(:timeout, state) do
  # handle inactivity timeout
  IO.puts("Server timed out, cleaning up...")
  {:stop, :normal, state}
end

@impl true
def handle_info(msg, state) do
  IO.puts("Unexpected message: #{inspect(msg)}")
  {:noreply, state}
end
```

---

## Step 104: สร้าง Real GenServer - Cache System

```elixir
defmodule Cache do
  use GenServer
  require Logger
  
  @default_ttl 60_000  # 1 minute in milliseconds
  
  # ───── Client API ─────
  
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def put(key, value, ttl \\ @default_ttl) do
    GenServer.cast(__MODULE__, {:put, key, value, ttl})
  end
  
  def get(key) do
    GenServer.call(__MODULE__, {:get, key})
  end
  
  def delete(key) do
    GenServer.cast(__MODULE__, {:delete, key})
  end
  
  def clear do
    GenServer.cast(__MODULE__, :clear)
  end
  
  def stats do
    GenServer.call(__MODULE__, :stats)
  end
  
  # ───── Callbacks ─────
  
  @impl true
  def init(_opts) do
    # ตั้ง timer เพื่อ cleanup expired entries ทุก 30 วินาที
    :timer.send_interval(30_000, self(), :cleanup)
    
    state = %{
      entries: %{},
      hits: 0,
      misses: 0,
      evictions: 0
    }
    
    {:ok, state}
  end
  
  @impl true
  def handle_cast({:put, key, value, ttl}, state) do
    expires_at = System.monotonic_time(:millisecond) + ttl
    
    entry = %{
      value: value,
      expires_at: expires_at,
      created_at: DateTime.utc_now()
    }
    
    new_entries = Map.put(state.entries, key, entry)
    {:noreply, %{state | entries: new_entries}}
  end
  
  @impl true
  def handle_cast({:delete, key}, state) do
    {:noreply, %{state | entries: Map.delete(state.entries, key)}}
  end
  
  @impl true
  def handle_cast(:clear, state) do
    {:noreply, %{state | entries: %{}}}
  end
  
  @impl true
  def handle_call({:get, key}, _from, state) do
    now = System.monotonic_time(:millisecond)
    
    case Map.get(state.entries, key) do
      nil ->
        {:reply, nil, %{state | misses: state.misses + 1}}
      
      %{expires_at: exp} when exp < now ->
        # Expired
        new_entries = Map.delete(state.entries, key)
        new_state = %{state | 
          entries: new_entries,
          misses: state.misses + 1,
          evictions: state.evictions + 1
        }
        {:reply, nil, new_state}
      
      %{value: value} ->
        {:reply, value, %{state | hits: state.hits + 1}}
    end
  end
  
  @impl true
  def handle_call(:stats, _from, state) do
    total = state.hits + state.misses
    hit_rate = if total > 0, do: state.hits / total * 100, else: 0.0
    
    stats = %{
      entries: map_size(state.entries),
      hits: state.hits,
      misses: state.misses,
      evictions: state.evictions,
      hit_rate: Float.round(hit_rate, 2)
    }
    
    {:reply, stats, state}
  end
  
  @impl true
  def handle_info(:cleanup, state) do
    now = System.monotonic_time(:millisecond)
    
    {valid, expired} = Enum.split_with(state.entries, fn {_, entry} ->
      entry.expires_at > now
    end)
    
    evicted = length(expired)
    
    if evicted > 0 do
      Logger.debug("Cache cleanup: removed #{evicted} expired entries")
    end
    
    {:noreply, %{state | 
      entries: Map.new(valid),
      evictions: state.evictions + evicted
    }}
  end
end

# การใช้งาน
{:ok, _pid} = Cache.start_link()

Cache.put(:user_1, %{name: "Alice", age: 30})
Cache.put(:config, %{theme: "dark"}, 300_000)  # 5 minute TTL

Cache.get(:user_1)   # %{name: "Alice", age: 30}
Cache.get(:missing)  # nil

Cache.stats()
# %{entries: 2, hits: 1, misses: 1, evictions: 0, hit_rate: 50.0}
```

---

## Step 105: GenServer กับ Supervisor Integration

### 105.1 Application Structure

```elixir
defmodule MyApp do
  use Application
  
  def start(_type, _args) do
    children = [
      # {Module, initial_args}
      {Cache, []},
      {UserRegistry, name: UserRegistry},
      {TaskQueue, max_workers: 5}
    ]
    
    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

### 105.2 child_spec

```elixir
defmodule MyWorker do
  use GenServer
  
  # GenServer.start_link ถ้าไม่มี name จะใช้ pid
  # แต่ถ้าต้องการให้ Supervisor restart ได้ ต้องมี unique id
  
  def child_spec(opts) do
    %{
      id: __MODULE__,
      start: {__MODULE__, :start_link, [opts]},
      restart: :permanent,  # :permanent | :transient | :temporary
      shutdown: 5000,
      type: :worker
    }
  end
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @impl true
  def init(opts) do
    {:ok, %{opts: opts}}
  end
end
```

---

## Step 106: call vs cast vs info

### 106.1 เมื่อไหร่ใช้อะไร

```elixir
# GenServer.call - ใช้เมื่อ:
# 1. ต้องการ response จาก server
# 2. ต้องการ synchronize กับ server state
# 3. Operation ที่อาจ fail และต้องการ error handling

result = GenServer.call(server, {:get_balance, account_id})
case result do
  {:ok, balance}   -> IO.puts("Balance: #{balance}")
  {:error, reason} -> IO.puts("Error: #{reason}")
end

# GenServer.cast - ใช้เมื่อ:
# 1. Fire-and-forget operations
# 2. Logging, events
# 3. State updates ที่ไม่ต้องการรู้ผลทันที

GenServer.cast(logger, {:log, :info, "User logged in"})
GenServer.cast(cache, {:invalidate, key})

# send/handle_info - ใช้เมื่อ:
# 1. Scheduled work (timers)
# 2. External events (network, files)
# 3. Process monitoring events

Process.send_after(self(), :check_connections, 30_000)
send(server, {:external_event, data})
```

### 106.2 Back-pressure กับ call

```elixir
defmodule SlowServer do
  use GenServer
  
  # call จะ block caller ถ้า server ยุ่ง
  # ใช้เป็น natural back-pressure mechanism
  
  @impl true
  def handle_call({:process, data}, _from, state) do
    # ถ้าใช้เวลานาน caller จะ block
    result = expensive_computation(data)
    {:reply, result, state}
  end
  
  # Timeout guard
  def process(server, data) do
    try do
      GenServer.call(server, {:process, data}, 10_000)
    catch
      :exit, {:timeout, _} -> {:error, :timeout}
    end
  end
end
```

---

## Step 107: Advanced GenServer Patterns

### 107.1 Delayed Initialization

```elixir
defmodule LazyLoader do
  use GenServer
  
  @impl true
  def init(_) do
    # ส่ง message ให้ตัวเองทำงานหลัง init เสร็จ
    send(self(), :load_data)
    {:ok, %{loaded: false, data: nil}}
  end
  
  @impl true
  def handle_info(:load_data, state) do
    # ทำ heavy initialization ใน handle_info แทน init
    # เพื่อไม่ให้ block supervisor
    data = load_from_database()
    {:noreply, %{state | loaded: true, data: data}}
  end
  
  @impl true
  def handle_call(:get_data, _from, %{loaded: false} = state) do
    {:reply, {:error, :not_ready}, state}
  end
  
  @impl true
  def handle_call(:get_data, _from, %{data: data} = state) do
    {:reply, {:ok, data}, state}
  end
  
  defp load_from_database do
    Process.sleep(100)  # simulate slow load
    %{users: ["Alice", "Bob"], products: ["Widget"]}
  end
end
```

### 107.2 Request Coalescing

```elixir
defmodule RequestCoalescer do
  use GenServer
  
  @batch_wait 50  # milliseconds
  
  @impl true
  def init(_) do
    {:ok, %{pending: [], timer_ref: nil}}
  end
  
  def request(server, data) do
    GenServer.call(server, {:request, data})
  end
  
  @impl true
  def handle_call({:request, data}, from, state) do
    new_pending = [{from, data} | state.pending]
    
    # Cancel existing timer
    if state.timer_ref, do: Process.cancel_timer(state.timer_ref)
    
    # Set new timer to batch requests
    timer_ref = Process.send_after(self(), :process_batch, @batch_wait)
    
    {:noreply, %{state | pending: new_pending, timer_ref: timer_ref}}
  end
  
  @impl true
  def handle_info(:process_batch, state) do
    batch = Enum.reverse(state.pending)
    
    # Process all pending requests at once
    results = process_batch(Enum.map(batch, fn {_, data} -> data end))
    
    # Reply to all callers
    Enum.zip(batch, results)
    |> Enum.each(fn {{from, _}, result} ->
         GenServer.reply(from, result)
       end)
    
    {:noreply, %{state | pending: [], timer_ref: nil}}
  end
  
  defp process_batch(items) do
    IO.puts("Processing batch of #{length(items)} items")
    Enum.map(items, fn item -> {:processed, item} end)
  end
end
```

### 107.3 Hibernate สำหรับ Memory

```elixir
defmodule HibernatingServer do
  use GenServer
  
  @idle_timeout 60_000  # hibernate after 1 minute of inactivity
  
  @impl true
  def init(_) do
    {:ok, %{data: nil}, @idle_timeout}
  end
  
  @impl true
  def handle_call(:get, _from, state) do
    {:reply, state.data, state, @idle_timeout}  # reset timeout
  end
  
  @impl true
  def handle_cast({:put, data}, _state) do
    {:noreply, %{data: data}, @idle_timeout}
  end
  
  @impl true
  def handle_info(:timeout, state) do
    # Hibernate - ลด memory footprint ระหว่างรอ
    IO.puts("Going to hibernate...")
    {:noreply, state, :hibernate}
  end
end
```

---

## Step 108: Testing GenServer

### 108.1 Unit Testing

```elixir
defmodule CacheTest do
  use ExUnit.Case
  
  setup do
    # เริ่ม fresh Cache สำหรับแต่ละ test
    {:ok, cache} = start_supervised(Cache)
    {:ok, cache: cache}
  end
  
  test "put and get value", %{cache: _} do
    Cache.put(:key, "value")
    assert Cache.get(:key) == "value"
  end
  
  test "returns nil for missing key", %{cache: _} do
    assert Cache.get(:nonexistent) == nil
  end
  
  test "expires entries after TTL", %{cache: _} do
    Cache.put(:key, "value", 100)  # 100ms TTL
    assert Cache.get(:key) == "value"
    
    Process.sleep(200)
    
    assert Cache.get(:key) == nil
  end
  
  test "tracks stats correctly", %{cache: _} do
    Cache.put(:key1, "v1")
    Cache.put(:key2, "v2")
    
    Cache.get(:key1)
    Cache.get(:key1)
    Cache.get(:missing)
    
    stats = Cache.stats()
    assert stats.hits == 2
    assert stats.misses == 1
    assert stats.hit_rate == 66.67
  end
end
```

### 108.2 Testing Async Operations

```elixir
defmodule AsyncServerTest do
  use ExUnit.Case
  
  test "cast update is reflected in call" do
    {:ok, server} = start_supervised(MyServer)
    
    # cast หรือ async
    GenServer.cast(server, {:update, :state_value})
    
    # ต้องรอ cast ทำงานเสร็จก่อน
    # วิธีที่ 1: sleep (ไม่แนะนำ)
    # Process.sleep(10)
    
    # วิธีที่ 2: ใช้ call synchronize (แนะนำ)
    # call จะ queue ตาม cast ในมลbox
    state = GenServer.call(server, :get_state)
    assert state == :state_value
  end
end
```

---

## Step 109: GenServer กับ External Services

### 109.1 Database Connection Pool

```elixir
defmodule DBConnectionPool do
  use GenServer
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def with_connection(func) do
    conn = GenServer.call(__MODULE__, :checkout, 10_000)
    try do
      result = func.(conn)
      GenServer.cast(__MODULE__, {:checkin, conn})
      result
    rescue
      error ->
        GenServer.cast(__MODULE__, {:checkin, conn})
        reraise error, __STACKTRACE__
    end
  end
  
  @impl true
  def init(opts) do
    pool_size = Keyword.get(opts, :pool_size, 10)
    
    # สร้าง connections
    connections = Enum.map(1..pool_size, fn id ->
      {:ok, conn} = Database.connect(opts)
      {id, conn}
    end)
    
    state = %{
      available: connections,
      in_use: [],
      waiting: :queue.new()
    }
    
    {:ok, state}
  end
  
  @impl true
  def handle_call(:checkout, from, state) do
    case state.available do
      [] ->
        # ไม่มี connection ว่าง - รอ
        new_waiting = :queue.in(from, state.waiting)
        {:noreply, %{state | waiting: new_waiting}}
      
      [{id, conn} | rest] ->
        new_in_use = [{id, conn, from} | state.in_use]
        {:reply, conn, %{state | available: rest, in_use: new_in_use}}
    end
  end
  
  @impl true
  def handle_cast({:checkin, conn}, state) do
    case :queue.out(state.waiting) do
      {{:value, from}, new_waiting} ->
        # มีคนรอ - ส่ง connection ให้ทันที
        GenServer.reply(from, conn)
        {:noreply, %{state | waiting: new_waiting}}
      
      {:empty, _} ->
        # ไม่มีคนรอ - return to available
        entry = List.keyfind(state.in_use, conn, 1)
        if entry do
          {id, _, _} = entry
          new_in_use = List.keydelete(state.in_use, conn, 1)
          {:noreply, %{state | 
            available: [{id, conn} | state.available],
            in_use: new_in_use
          }}
        else
          {:noreply, state}
        end
    end
  end
end
```

---

## Step 110: Complete GenServer Example - Rate Limiter

```elixir
defmodule RateLimiter do
  @moduledoc """
  Token bucket rate limiter using GenServer.
  
  Allows N requests per minute per key.
  """
  
  use GenServer
  require Logger
  
  @max_tokens 100      # max requests
  @refill_rate 100     # tokens per minute
  @refill_interval 60_000  # milliseconds
  
  defstruct [:max_tokens, :refill_rate, :buckets]
  
  # ───── Client API ─────
  
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @doc """
  Check and consume a token for the given key.
  Returns :allow or {:deny, retry_after_ms}
  """
  def check(key, tokens_required \\ 1) do
    GenServer.call(__MODULE__, {:check, key, tokens_required})
  end
  
  @doc """
  Get current state of a bucket
  """
  def bucket_info(key) do
    GenServer.call(__MODULE__, {:info, key})
  end
  
  # ───── Callbacks ─────
  
  @impl true
  def init(opts) do
    max_tokens    = Keyword.get(opts, :max_tokens, @max_tokens)
    refill_rate   = Keyword.get(opts, :refill_rate, @refill_rate)
    
    # ตั้ง refill timer
    :timer.send_interval(@refill_interval, self(), :refill)
    
    state = %{
      max_tokens: max_tokens,
      refill_rate: refill_rate,
      buckets: %{}    # key => {tokens, last_request_at}
    }
    
    {:ok, state}
  end
  
  @impl true
  def handle_call({:check, key, tokens_required}, _from, state) do
    now = System.monotonic_time(:millisecond)
    bucket = get_or_create_bucket(state, key, now)
    
    if bucket.tokens >= tokens_required do
      # Allow request
      updated_bucket = %{bucket | 
        tokens: bucket.tokens - tokens_required,
        last_request: now
      }
      new_state = put_in(state, [:buckets, key], updated_bucket)
      {:reply, :allow, new_state}
    else
      # Deny request
      tokens_needed = tokens_required - bucket.tokens
      refill_per_ms = state.refill_rate / @refill_interval
      retry_after = ceil(tokens_needed / refill_per_ms)
      
      {:reply, {:deny, retry_after}, state}
    end
  end
  
  @impl true
  def handle_call({:info, key}, _from, state) do
    bucket = Map.get(state.buckets, key)
    {:reply, bucket, state}
  end
  
  @impl true
  def handle_info(:refill, state) do
    now = System.monotonic_time(:millisecond)
    
    new_buckets = Map.new(state.buckets, fn {key, bucket} ->
      elapsed_ms = now - (bucket.last_refill || now)
      tokens_to_add = state.refill_rate * elapsed_ms / @refill_interval
      
      new_tokens = min(
        state.max_tokens,
        bucket.tokens + tokens_to_add
      )
      
      {key, %{bucket | tokens: new_tokens, last_refill: now}}
    end)
    
    {:noreply, %{state | buckets: new_buckets}}
  end
  
  # ───── Private ─────
  
  defp get_or_create_bucket(state, key, now) do
    Map.get_lazy(state.buckets, key, fn ->
      %{
        tokens: state.max_tokens,
        last_request: now,
        last_refill: now
      }
    end)
  end
end

# การใช้งาน
{:ok, _} = RateLimiter.start_link(max_tokens: 10, refill_rate: 10)

# Test rate limiting
for i <- 1..15 do
  case RateLimiter.check("user:alice") do
    :allow            -> IO.puts("Request #{i}: Allowed")
    {:deny, retry_ms} -> IO.puts("Request #{i}: Denied, retry after #{retry_ms}ms")
  end
end

# Output:
# Request 1-10: Allowed
# Request 11-15: Denied, retry after Xms
```

---

## สรุป Part 11: OTP และ GenServer

✅ **Step 101** - OTP คืออะไรและทำไมต้องใช้  
✅ **Step 102** - GenServer structure และ start_link  
✅ **Step 103** - Callbacks: init, handle_call, handle_cast, handle_info  
✅ **Step 104** - Cache System ตัวอย่างจริง  
✅ **Step 105** - Integration กับ Supervisor  
✅ **Step 106** - call vs cast vs info  
✅ **Step 107** - Advanced patterns  
✅ **Step 108** - Testing GenServer  
✅ **Step 109** - External service integration  
✅ **Step 110** - Rate Limiter complete example  

---

## แบบฝึกหัด Part 11

1. สร้าง `SessionStore` GenServer ที่:
   - สร้าง/ดึง/ลบ sessions
   - Sessions expire อัตโนมัติ
   - Track active user count

2. สร้าง `JobQueue` GenServer ที่:
   - เพิ่ม jobs ด้วย priority
   - Workers pull jobs จาก queue
   - Track job status (pending/running/done/failed)

3. สร้าง `MetricsCollector` ที่:
   - รับ metrics events (count, timing, gauge)
   - Aggregate ทุก 10 วินาที
   - Report summary

---

➡️ [Part 12: Supervisor Trees](./part-12-supervisor.md)
