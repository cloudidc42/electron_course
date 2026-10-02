# Part 58: BEAM Internals (Steps 631-650)

## Step 631: BEAM Architecture

```
BEAM Virtual Machine Architecture:

┌─────────────────────────────────────────────────────┐
│                    BEAM Process                      │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Heap     │  │ Stack    │  │ PCB              │  │
│  │ (GC'd)   │  │          │  │ (Process Control │  │
│  │          │  │          │  │  Block)          │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │           Mailbox (message queue)            │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                BEAM Schedulers                       │
│                                                      │
│  Scheduler 1   Scheduler 2   Scheduler N             │
│  (OS thread)   (OS thread)   (OS thread)             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │Run Queue│  │Run Queue│  │Run Queue│             │
│  └─────────┘  └─────────┘  └─────────┘             │
│                                                      │
│  Dirty CPU Scheduler(s)  Dirty I/O Scheduler(s)     │
└─────────────────────────────────────────────────────┘

Key Concepts:
- Preemptive scheduling (reductions)
- Per-process heap (isolated GC)
- Message passing via copy (no sharing)
- Lock-free data structures
```

---

## Step 632: Reductions and Scheduling

```elixir
# Each BEAM instruction = 1 reduction
# Process gets ~2000 reductions before being preempted
# This enables fair scheduling across all processes

defmodule MyApp.Reductions do
  def inspect_current do
    {reductions, _} = Process.info(self(), :reductions)
    reductions
  end

  def test_reduction_cost do
    before = inspect_current()
    
    # These operations have different reduction costs
    Enum.reduce(1..10_000, 0, &+/2)  # many reductions
    
    after_val = inspect_current()
    after_val - before
  end

  # Yield control to other processes voluntarily
  def cooperative_task(n) when n > 0 do
    # Process large work in chunks to yield between them
    process_chunk(Enum.take(1..n, 100))
    :erlang.yield()  # yield to scheduler
    cooperative_task(n - 100)
  end
  def cooperative_task(0), do: :done
end

# Check how many reductions a process used
{:reductions, r} = Process.info(pid, :reductions)
IO.puts("Process used #{r} reductions")
```

---

## Step 633: Garbage Collection

```elixir
defmodule MyApp.GCInsights do
  # BEAM uses generational + incremental GC per-process
  # Minor GC: triggered when young generation is full
  # Major GC: triggered when process heap exceeds threshold

  def inspect_heap(pid) do
    Process.info(pid, [
      :heap_size,           # words on heap
      :total_heap_size,     # including old heap
      :garbage_collection,  # GC stats
      :memory               # total bytes
    ])
  end

  def trigger_gc(pid) do
    :erlang.garbage_collect(pid)
  end

  # Tune GC via process flags
  def configure_gc(pid) do
    Process.flag(:min_heap_size, 4096)  # minimum heap words
    Process.flag(:min_bin_vheap_size, 65536)  # binary heap
    Process.flag(pid, :fullsweep_after, 10)   # full GC every 10 GCs
  end

  # Force GC on all processes (dangerous in prod)
  def gc_all_processes do
    Process.list()
    |> Enum.each(&:erlang.garbage_collect/1)
  end

  # Binary reference counting
  def check_binary_memory do
    :erlang.memory(:binary)
  end
end
```

---

## Step 634: Process Internals

```elixir
defmodule MyApp.ProcessInspector do
  def full_info(pid) do
    Process.info(pid, [
      :registered_name,
      :status,            # running, waiting, suspended, etc.
      :current_function,
      :initial_call,
      :reductions,
      :message_queue_len,
      :messages,          # actual messages in queue
      :links,
      :monitors,
      :monitored_by,
      :trap_exit,
      :priority,          # normal, high, max, low
      :dictionary,        # process dictionary
      :ancestors,         # supervision tree
      :label              # OTP 26+
    ])
  end

  def set_priority(pid, priority) when priority in [:low, :normal, :high, :max] do
    Process.flag(pid, :priority, priority)
  end

  def suspend(pid), do: :erlang.suspend_process(pid)
  def resume(pid),  do: :erlang.resume_process(pid)

  # Process dictionary (avoid in most cases, use GenServer state)
  def pdict_set(key, value), do: Process.put(key, value)
  def pdict_get(key),        do: Process.get(key)
  def pdict_all,             do: Process.get()
end
```

---

## Step 635: System Monitoring

```elixir
defmodule MyApp.SystemMonitor do
  def start_monitoring do
    # Monitor process events system-wide
    :erlang.system_monitor(self(), [
      {:long_gc, 100},          # GC taking >100ms
      {:large_heap, 1_000_000}, # heap > 1M words
      :busy_port,               # port busy
      :busy_dist_port           # distribution port busy
    ])
  end

  def handle_info({:monitor, _pid, :long_gc, info}, state) do
    Logger.warning("Long GC: #{inspect(info)}")
    {:noreply, state}
  end

  def handle_info({:monitor, pid, :large_heap, %{heap_size: size}}, state) do
    name = Process.info(pid, :registered_name)
    Logger.warning("Large heap #{size}: #{inspect(name)}")
    {:noreply, state}
  end

  # System statistics
  def stats do
    %{
      process_count:     :erlang.system_info(:process_count),
      process_limit:     :erlang.system_info(:process_limit),
      port_count:        :erlang.system_info(:port_count),
      scheduler_count:   :erlang.system_info(:schedulers),
      memory:            :erlang.memory(),
      atom_count:        :erlang.system_info(:atom_count),
      atom_limit:        :erlang.system_info(:atom_limit)
    }
  end
end
```

---

## Step 636: Distribution Protocol

```elixir
defmodule MyApp.Distribution do
  # BEAM nodes communicate via the Distribution Protocol
  # Uses TCP by default

  def setup_node do
    # Set cookie for authentication
    Node.set_cookie(:secret_cookie)

    # Connect to another node
    Node.connect(:"other@host")

    # List connected nodes
    Node.list()
  end

  # Remote procedure call
  def rpc(node, module, function, args) do
    :rpc.call(node, module, function, args)
  end

  # Async RPC
  def rpc_async(node, module, function, args) do
    key = :rpc.async_call(node, module, function, args)
    :rpc.yield(key)
  end

  # Multicall - call all nodes
  def multicall(module, function, args) do
    :rpc.multicall(module, function, args)
  end

  # Global process registration (across cluster)
  def register_globally(name, pid) do
    :global.register_name(name, pid)
  end

  def lookup_globally(name) do
    :global.whereis_name(name)
  end

  # Monitor remote nodes
  def monitor_nodes do
    :net_kernel.monitor_nodes(true)
    # Receive {:nodeup, node} and {:nodedown, node} messages
  end
end
```

---

## Step 637: ETS Internals

```elixir
defmodule MyApp.ETSInternals do
  # ETS is stored in native memory (not process heap)
  # Copying semantics: data is copied in/out
  # Multiple readers, single writer (by default with write_concurrency)

  def demonstrate_types do
    # :set - hash table, O(1) lookup by key
    :ets.new(:hash_set, [:set, :public, :named_table])

    # :ordered_set - balanced BST, O(log n) lookup, sorted
    :ets.new(:sorted_set, [:ordered_set, :public, :named_table])

    # :bag - multiple values per key
    :ets.new(:multi_bag, [:bag, :public, :named_table])

    # :duplicate_bag - allows duplicate key-value pairs
    :ets.new(:dup_bag, [:duplicate_bag, :public, :named_table])
  end

  def benchmark_access_patterns do
    # Comparison: ETS vs process dict vs persistent_term
    
    :ets.new(:bench, [:set, :public, :named_table, read_concurrency: true])
    :ets.insert(:bench, {:key, "value"})

    # ETS: ~100ns read
    fn -> :ets.lookup(:bench, :key) end

    # persistent_term: ~10ns read (fastest)
    :persistent_term.put(:bench_key, "value")
    fn -> :persistent_term.get(:bench_key) end

    # Process dict: ~5ns (fastest, but only within process)
    Process.put(:bench_key, "value")
    fn -> Process.get(:bench_key) end
  end

  # ETS as LRU cache
  def setup_lru(name, max_size) do
    :ets.new(name, [:ordered_set, :public, :named_table])
    :ets.new(:"#{name}_meta", [:set, :public, :named_table])
    :persistent_term.put({:lru_max, name}, max_size)
  end
end
```

---

## Step 638: Mnesia Deep Dive

```elixir
defmodule MyApp.MnesiaAdvanced do
  # Mnesia = distributed database built into OTP
  # Stored in BEAM memory (RAM and/or disk)
  
  def setup_cluster do
    # Create schema across all nodes
    nodes = [node() | Node.list()]
    :mnesia.create_schema(nodes)
    
    # Start Mnesia on all nodes
    :rpc.multicall(nodes, :mnesia, :start, [])
    
    # Create replicated table
    :mnesia.create_table(:sessions, [
      attributes:    [:token, :user_id, :expires_at, :data],
      ram_copies:    nodes,           # in-memory on all nodes
      type:          :set,
      index:         [:user_id]
    ])
    
    # Create persistent table
    :mnesia.create_table(:audit_log, [
      attributes:  [:id, :tenant_id, :action, :timestamp, :data],
      disc_copies: nodes,             # persisted to disk
      type:        :bag
    ])
  end

  # Complex transactions
  def transfer(from_id, to_id, amount) do
    :mnesia.transaction(fn ->
      [from] = :mnesia.wlock(:accounts, from_id)
      [to]   = :mnesia.wlock(:accounts, to_id)

      if from.balance < amount do
        :mnesia.abort(:insufficient_funds)
      end

      :mnesia.write({:accounts, from_id, from.balance - amount})
      :mnesia.write({:accounts, to_id,   to.balance + amount})
    end)
  end

  # Dirty operations (no transaction, much faster)
  def fast_read(table, key) do
    :mnesia.dirty_read(table, key)
  end

  def fast_write(record) do
    :mnesia.dirty_write(record)
  end
end
```

---

## Step 639: Tracing

```elixir
defmodule MyApp.Tracer do
  # Erlang tracing - powerful but use with care in production
  
  def trace_function(module, function, arity) do
    # Turn on tracing for specific function
    :dbg.tracer()
    :dbg.p(:all, :c)
    :dbg.tpl(module, function, arity, [{[], [], [{:return_trace}]}])
  end

  def stop_trace do
    :dbg.stop_clear()
  end

  # Trace specific process messages
  def trace_messages(pid) do
    :dbg.tracer()
    :dbg.p(pid, [:send, :receive])
  end

  # Using :sys module to trace GenServer
  def trace_gen_server(name_or_pid) do
    :sys.trace(name_or_pid, true)
    # prints each message, state change, etc.
  end

  def stop_gen_server_trace(name_or_pid) do
    :sys.trace(name_or_pid, false)
  end

  # Snapshot GenServer state
  def get_state(name_or_pid) do
    :sys.get_state(name_or_pid)
  end

  # Replace state (for debugging only!)
  def replace_state(name_or_pid, new_state) do
    :sys.replace_state(name_or_pid, fn _old -> new_state end)
  end
end
```

---

## Step 640: BEAM Flags and Tuning

```bash
# BEAM startup flags for production tuning

# Number of schedulers (default = CPU cores)
erl +S 8:8   # 8 online, 8 total

# Scheduler bind type (improve cache locality)
erl +sbt db  # db = default bind type

# Async threads for dirty I/O
erl +A 128   # 128 async threads

# Increase atom table
erl +t 1048576  # 1M atoms

# Max processes
erl +P 2000000  # 2M processes

# Large heap for specific apps
erl +hms 4096   # min process heap size

# Enable time correction
erl +c true

# Mix release: rel/vm.args
# +P 2000000
# +Q 65536
# +S 8:8
# +sbt db
# +K true
# +A 128
```

```elixir
# Runtime BEAM configuration
defmodule MyApp.BEAMConfig do
  def tune_for_throughput do
    # Increase process limit
    # Only at startup via +P flag, can't change at runtime
    
    # Adjust scheduler utilization
    :erlang.system_flag(:scheduler_bind_type, :default_bind)
    
    # Busy wait settings (better latency, higher CPU usage)
    :erlang.system_flag(:busy_wait_threshold, 100)
  end

  def system_info do
    %{
      otp_release:     :erlang.system_info(:otp_release),
      erts_version:    :erlang.system_info(:version),
      schedulers:      :erlang.system_info(:schedulers_online),
      logical_cpus:    :erlang.system_info(:logical_processors),
      process_limit:   :erlang.system_info(:process_limit),
      port_limit:      :erlang.system_info(:port_limit),
      smp_support:     :erlang.system_info(:smp_support),
      hipe_enabled:    false,
      dirty_schedulers: :erlang.system_info(:dirty_cpu_schedulers)
    }
  end
end
```

---

## สรุป Part 58

✅ **Step 631** - BEAM architecture  
✅ **Step 632** - Reductions & scheduling  
✅ **Step 633** - Garbage collection  
✅ **Step 634** - Process internals  
✅ **Step 635** - System monitoring  
✅ **Step 636** - Distribution protocol  
✅ **Step 637** - ETS internals  
✅ **Step 638** - Mnesia deep dive  
✅ **Step 639** - Tracing  
✅ **Step 640** - BEAM flags & tuning  

➡️ [Part 59: Payment & Commerce Systems](./part-59-payments.md)
