# Part 10: Processes พื้นฐาน (Steps 91-100)

> Processes ใน Elixir คือหัวใจของ Concurrency - ไม่ใช่ OS Processes แต่เป็น lightweight BEAM processes

## Step 91: Process คืออะไร?

### 91.1 BEAM Process vs OS Thread

```
OS Thread (Java, Python):           BEAM Process (Elixir):
┌──────────────────────────┐        ┌─────────────────────────────┐
│ Memory: ~1-2MB per thread │        │ Memory: ~0.5-2KB per process │
│ Creation: ~100-1000μs     │        │ Creation: ~1μs               │
│ Context switch: expensive │        │ Context switch: cheap         │
│ Shared memory (locks!)    │        │ No shared memory              │
│ OS managed scheduling     │        │ BEAM managed scheduling       │
└──────────────────────────┘        └─────────────────────────────┘

Java/Python: รองรับ ~1,000-10,000 threads
Elixir/BEAM: รองรับ 1,000,000+ processes พร้อมกัน!
```

### 91.2 Process Identity

```elixir
# แต่ละ process มี PID (Process Identifier)
iex> self()
#PID<0.111.0>

iex> pid = self()
iex> pid == self()
true

# PID ประกอบด้วย node, process id, serial number
# #PID<0.111.0>
#       ↑ node (0 = local)
#         ↑↑↑ process id
#             ↑ serial
```

### 91.3 spawn/1 - สร้าง Process ใหม่

```elixir
# spawn สร้าง process ใหม่และรัน function
pid = spawn(fn ->
  IO.puts("Hello from process #{inspect(self())}!")
end)

# Main process ทำงานต่อทันที ไม่รอ
IO.puts("Main process: #{inspect(self())}")
IO.puts("Spawned process: #{inspect(pid)}")

# Output (ลำดับอาจต่างกัน):
# Main process: #PID<0.111.0>
# Spawned process: #PID<0.112.0>
# Hello from process #PID<0.112.0>!
```

---

## Step 92: Process Communication (Messages)

### 92.1 send/2 และ receive

```elixir
# send ส่ง message ไปยัง process
# receive รอรับ message

# Pattern 1: Simple ping-pong
parent = self()

child = spawn(fn ->
  receive do
    {:ping, from} ->
      IO.puts("Got ping!")
      send(from, :pong)
  end
end)

send(child, {:ping, self()})

receive do
  :pong -> IO.puts("Got pong!")
end

# Output:
# Got ping!
# Got pong!
```

### 92.2 Message Mailbox

```elixir
# ทุก process มี mailbox
# Messages อยู่ใน mailbox จนกว่าจะถูก receive

# ส่ง messages หลายอัน
send(self(), :first)
send(self(), :second)
send(self(), :third)

# receive ทีละอัน
receive do msg -> IO.puts("1: #{msg}") end
receive do msg -> IO.puts("2: #{msg}") end
receive do msg -> IO.puts("3: #{msg}") end

# Output:
# 1: first
# 2: second
# 3: third
```

### 92.3 Pattern Matching ใน receive

```elixir
def listen() do
  receive do
    {:add, a, b} ->
      IO.puts("#{a} + #{b} = #{a + b}")
      listen()
    
    {:multiply, a, b} ->
      IO.puts("#{a} * #{b} = #{a * b}")
      listen()
    
    :stop ->
      IO.puts("Calculator stopped")
    
    msg ->
      IO.puts("Unknown message: #{inspect(msg)}")
      listen()
  end
end

calc = spawn(&listen/0)

send(calc, {:add, 5, 3})       # 5 + 3 = 8
send(calc, {:multiply, 4, 7})  # 4 * 7 = 28
send(calc, :stop)               # Calculator stopped
```

---

## Step 93: Process State

### 93.1 State ผ่าน Recursion

```elixir
# Process ไม่มี mutable state แต่ส่ง state ผ่าน recursive call
def counter(count \\ 0) do
  receive do
    :increment ->
      counter(count + 1)   # ส่ง new state ผ่าน recursion
    
    :decrement ->
      counter(count - 1)
    
    {:get, from} ->
      send(from, count)
      counter(count)
    
    :reset ->
      counter(0)
    
    :stop ->
      IO.puts("Final count: #{count}")
  end
end

# เริ่ม counter process
counter_pid = spawn(&counter/0)

send(counter_pid, :increment)
send(counter_pid, :increment)
send(counter_pid, :increment)
send(counter_pid, :decrement)

send(counter_pid, {:get, self()})
receive do count -> IO.puts("Count: #{count}") end
# Count: 2
```

### 93.2 State กับ Complex Data

```elixir
defmodule Registry do
  def start do
    spawn(fn -> loop(%{}) end)
  end
  
  defp loop(state) do
    receive do
      {:register, key, value, from} ->
        send(from, :ok)
        loop(Map.put(state, key, value))
      
      {:lookup, key, from} ->
        send(from, Map.get(state, key))
        loop(state)
      
      {:unregister, key, from} ->
        send(from, :ok)
        loop(Map.delete(state, key))
      
      {:list, from} ->
        send(from, Map.keys(state))
        loop(state)
      
      :stop ->
        :ok
    end
  end
  
  # Client API
  def register(pid, key, value) do
    send(pid, {:register, key, value, self()})
    receive do :ok -> :ok end
  end
  
  def lookup(pid, key) do
    send(pid, {:lookup, key, self()})
    receive do value -> value end
  end
end

registry = Registry.start()
Registry.register(registry, :user1, %{name: "Alice"})
Registry.register(registry, :user2, %{name: "Bob"})
Registry.lookup(registry, :user1)  # %{name: "Alice"}
```

---

## Step 94: Process Links

### 94.1 link/1 - เชื่อม Processes

```elixir
# linked processes จะพังด้วยกัน
parent = self()

child = spawn(fn ->
  Process.link(parent)  # link กับ parent
  
  receive do
    :work ->
      IO.puts("Working...")
      # จำลองการ crash
      raise "Something went wrong!"
  end
end)

# หรือ spawn_link ทำได้เลย
child = spawn_link(fn ->
  receive do
    :crash -> raise "Crash!"
  end
end)

# ถ้า child crash, parent ก็ crash ด้วย
# (เว้นแต่ parent เป็น trap_exit process)
```

### 94.2 trap_exit สำหรับจัดการ Link Failures

```elixir
# Process.flag(:trap_exit, true) ทำให้รับ exit signals เป็น messages
Process.flag(:trap_exit, true)

child = spawn_link(fn ->
  raise "Oops!"
end)

# รับ exit signal เป็น message แทนที่จะ crash
receive do
  {:EXIT, ^child, reason} ->
    IO.puts("Child crashed: #{inspect(reason)}")
end
# Child crashed: %RuntimeError{message: "Oops!"}
```

### 94.3 Monitor vs Link

```elixir
# Link: two-way, both crash
# Monitor: one-way, only monitor receives notification

# Monitor - ดีกว่าสำหรับ supervisor pattern
ref = Process.monitor(child_pid)

receive do
  {:DOWN, ^ref, :process, ^child_pid, reason} ->
    IO.puts("Process down: #{inspect(reason)}")
end

# spawn_monitor - spawn + monitor พร้อมกัน
{pid, ref} = spawn_monitor(fn ->
  Process.sleep(100)
  raise "Crash!"
end)

receive do
  {:DOWN, ^ref, :process, ^pid, reason} ->
    IO.puts("Down: #{inspect(reason)}")
end
```

---

## Step 95: Process.sleep และ Timing

### 95.1 Scheduling

```elixir
# Process.sleep - หยุด process ชั่วคราว
def worker(id) do
  IO.puts("Worker #{id} starting...")
  Process.sleep(1000)  # รอ 1 วินาที
  IO.puts("Worker #{id} done!")
end

# สร้าง workers หลายตัวพร้อมกัน
pids = Enum.map(1..5, fn id ->
  spawn(fn -> worker(id) end)
end)

# ทุก worker รันพร้อมกัน ใช้เวลา ~1 วินาที ไม่ใช่ 5 วินาที
```

### 95.2 Process Timeout

```elixir
# ส่ง message ไป process พร้อม timeout
def request_with_timeout(pid, message, timeout \\ 5000) do
  ref = make_ref()
  send(pid, {message, self(), ref})
  
  receive do
    {^ref, response} -> {:ok, response}
  after
    timeout -> {:error, :timeout}
  end
end

# ใช้งาน
case request_with_timeout(server_pid, :get_data) do
  {:ok, data}        -> process(data)
  {:error, :timeout} -> handle_timeout()
end
```

---

## Step 96: Process Dictionary

### 96.1 Process.put/get

```elixir
# Process dictionary = per-process key-value store
# ใช้กับระวัง - เหมือน global variables

# เก็บค่า
Process.put(:current_user, %{id: 1, name: "Alice"})
Process.put(:request_id, "req-12345")

# อ่านค่า
Process.get(:current_user)   # %{id: 1, name: "Alice"}
Process.get(:request_id)     # "req-12345"
Process.get(:missing_key)    # nil
Process.get(:missing_key, "default")  # "default"

# ลบค่า
Process.delete(:current_user)

# ดูทั้งหมด
Process.get()
# [current_user: nil, request_id: "req-12345", ...]
```

### 96.2 Request ID Tracking

```elixir
defmodule RequestContext do
  @request_id_key :request_id
  @user_id_key :current_user_id
  
  def set_request_id(id), do: Process.put(@request_id_key, id)
  def get_request_id, do: Process.get(@request_id_key, "unknown")
  
  def set_user(user_id), do: Process.put(@user_id_key, user_id)
  def get_user_id, do: Process.get(@user_id_key)
  
  def clear do
    Process.delete(@request_id_key)
    Process.delete(@user_id_key)
  end
end

# ใน request handler
def handle_request(conn) do
  RequestContext.set_request_id(generate_id())
  RequestContext.set_user(conn.user_id)
  
  # ทุก function ใน process นี้เข้าถึง context ได้
  process_request(conn)
ensure
  RequestContext.clear()
end
```

---

## Step 97: Named Processes

### 97.1 Process.register

```elixir
# ตั้งชื่อ process เพื่อเข้าถึงได้โดยไม่ต้องมี pid
pid = spawn(fn ->
  receive do
    :ping -> IO.puts("Pong!")
  end
end)

# ลงทะเบียนชื่อ
Process.register(pid, :my_server)

# เรียกใช้ด้วยชื่อ
send(:my_server, :ping)  # Pong!

# ดู pid จากชื่อ
Process.whereis(:my_server)  # #PID<0.112.0>

# ยกเลิกการลงทะเบียน
Process.unregister(:my_server)
```

### 97.2 via_name กับ Registry

```elixir
# ใช้ Elixir Registry สำหรับ dynamic process names
{:ok, _} = Registry.start_link(keys: :unique, name: MyRegistry)

# Register process
{:ok, pid} = spawn_and_register("user:123", fn ->
  receive do msg -> msg end
end)

# Lookup ด้วย name
[{pid, _value}] = Registry.lookup(MyRegistry, "user:123")

# หรือใช้ via tuple
{:via, Registry, {MyRegistry, "user:123"}}
```

---

## Step 98: Process Info และ Monitoring

### 98.1 Process.info

```elixir
# ดูข้อมูล process
Process.info(self())
# [
#   current_function: {IEx.Evaluator, :loop, 3},
#   initial_call: {IEx.Evaluator, :init, 4},
#   status: :running,
#   message_queue_len: 0,
#   links: [],
#   dictionary: [...],
#   trap_exit: false,
#   error_handler: :error_handler,
#   priority: :normal,
#   group_leader: #PID<0.65.0>,
#   total_heap_size: 4185,
#   heap_size: 1598,
#   stack_size: 28,
#   reductions: 1234,
#   garbage_collection: [...],
#   suspending: []
# ]

# ดูเฉพาะ field
Process.info(self(), :message_queue_len)  # {:message_queue_len, 0}
Process.info(self(), :memory)             # {:memory, 12345}
```

### 98.2 System Process Monitoring

```elixir
defmodule ProcessMonitor do
  def start_monitoring(pid) do
    spawn(fn -> monitor_loop(pid) end)
  end
  
  defp monitor_loop(pid) do
    ref = Process.monitor(pid)
    
    receive do
      {:DOWN, ^ref, :process, ^pid, :normal} ->
        IO.puts("Process #{inspect(pid)} finished normally")
      
      {:DOWN, ^ref, :process, ^pid, reason} ->
        IO.puts("Process #{inspect(pid)} crashed: #{inspect(reason)}")
        # อาจ restart หรือ alert
      
      :stop ->
        Process.demonitor(ref)
    end
  end
end
```

---

## Step 99: Concurrent Computations

### 99.1 Parallel Tasks

```elixir
# รัน tasks แบบ parallel
defmodule ParallelProcessor do
  def process_all(items, func) do
    # เริ่ม tasks ทั้งหมดพร้อมกัน
    tasks = Enum.map(items, fn item ->
      spawn(fn ->
        result = func.(item)
        send(self(), {:result, item, result})
      end)
    end)
    
    # รอรับผลทั้งหมด
    collect_results(length(items), %{})
  end
  
  defp collect_results(0, results), do: results
  defp collect_results(remaining, results) do
    receive do
      {:result, item, result} ->
        collect_results(remaining - 1, Map.put(results, item, result))
    end
  end
end

# ใช้ Task module แทน (ง่ายกว่า)
tasks = Enum.map([1, 2, 3, 4, 5], fn x ->
  Task.async(fn ->
    Process.sleep(100)
    x * x
  end)
end)

results = Task.await_many(tasks)
# [1, 4, 9, 16, 25]
```

### 99.2 Process Pool Pattern

```elixir
defmodule WorkerPool do
  def start(pool_size, worker_func) do
    # สร้าง pool ของ workers
    workers = Enum.map(1..pool_size, fn id ->
      spawn(fn -> worker_loop(id, worker_func) end)
    end)
    
    # สร้าง pool manager
    spawn(fn -> manager_loop(workers) end)
  end
  
  defp worker_loop(id, func) do
    receive do
      {job, from, ref} ->
        result = func.(job)
        send(from, {ref, result})
        worker_loop(id, func)
    end
  end
  
  defp manager_loop(workers) do
    receive do
      {:submit, job, from} ->
        # Round-robin assignment
        worker = hd(workers)
        ref = make_ref()
        send(worker, {job, from, ref})
        manager_loop(tl(workers) ++ [worker])
    end
  end
  
  def submit(pool, job) do
    send(pool, {:submit, job, self()})
    receive do {_ref, result} -> result end
  end
end
```

---

## Step 100: Building Stateful Server Process

### 100.1 Simple Key-Value Store

```elixir
defmodule KVStore do
  # Client API
  def start do
    spawn(&init/0)
  end
  
  def put(server, key, value) do
    send(server, {:put, key, value})
    :ok
  end
  
  def get(server, key) do
    ref = make_ref()
    send(server, {:get, key, self(), ref})
    receive do
      {^ref, value} -> value
    after
      5000 -> {:error, :timeout}
    end
  end
  
  def delete(server, key) do
    send(server, {:delete, key})
    :ok
  end
  
  def keys(server) do
    ref = make_ref()
    send(server, {:keys, self(), ref})
    receive do
      {^ref, keys} -> keys
    after
      5000 -> []
    end
  end
  
  # Server Implementation
  defp init do
    loop(%{})
  end
  
  defp loop(state) do
    receive do
      {:put, key, value} ->
        loop(Map.put(state, key, value))
      
      {:get, key, from, ref} ->
        send(from, {ref, Map.get(state, key)})
        loop(state)
      
      {:delete, key} ->
        loop(Map.delete(state, key))
      
      {:keys, from, ref} ->
        send(from, {ref, Map.keys(state)})
        loop(state)
      
      :dump ->
        IO.inspect(state, label: "Current state")
        loop(state)
      
      :stop ->
        IO.puts("KVStore stopping. Final state: #{inspect(state)}")
    end
  end
end

# การใช้งาน
store = KVStore.start()

KVStore.put(store, :name, "Alice")
KVStore.put(store, :age, 30)
KVStore.put(store, :city, "Bangkok")

KVStore.get(store, :name)   # "Alice"
KVStore.get(store, :age)    # 30
KVStore.keys(store)          # [:age, :city, :name]

KVStore.delete(store, :city)
KVStore.get(store, :city)    # nil

send(store, :dump)
# Current state: %{age: 30, name: "Alice"}
```

### 100.2 Chat Room Process

```elixir
defmodule ChatRoom do
  def start(name) do
    pid = spawn(fn -> room_loop(name, [], []) end)
    Process.register(pid, String.to_atom(name))
    pid
  end
  
  def join(room, user_pid, username) do
    send(room, {:join, user_pid, username})
  end
  
  def leave(room, user_pid) do
    send(room, {:leave, user_pid})
  end
  
  def send_message(room, user_pid, message) do
    send(room, {:message, user_pid, message})
  end
  
  defp room_loop(name, members, history) do
    receive do
      {:join, pid, username} ->
        IO.puts("[#{name}] #{username} joined")
        broadcast(members, {:joined, username})
        room_loop(name, [{pid, username} | members], history)
      
      {:leave, pid} ->
        case List.keyfind(members, pid, 0) do
          nil ->
            room_loop(name, members, history)
          
          {^pid, username} ->
            IO.puts("[#{name}] #{username} left")
            new_members = List.keydelete(members, pid, 0)
            broadcast(new_members, {:left, username})
            room_loop(name, new_members, history)
        end
      
      {:message, pid, text} ->
        case List.keyfind(members, pid, 0) do
          {^pid, username} ->
            msg = %{from: username, text: text, at: DateTime.utc_now()}
            broadcast(members, {:message, msg})
            room_loop(name, members, [msg | history])
          
          nil ->
            room_loop(name, members, history)
        end
    end
  end
  
  defp broadcast(members, message) do
    Enum.each(members, fn {pid, _} ->
      send(pid, message)
    end)
  end
end

defmodule ChatUser do
  def start(username) do
    spawn(fn -> user_loop(username) end)
  end
  
  defp user_loop(username) do
    receive do
      {:joined, name}            -> IO.puts("[#{username} sees] #{name} joined")
      {:left, name}              -> IO.puts("[#{username} sees] #{name} left")
      {:message, %{from: f, text: t}} -> IO.puts("[#{username} sees] #{f}: #{t}")
    end
    user_loop(username)
  end
end

# การใช้งาน
room = ChatRoom.start("general")

alice = ChatUser.start("Alice")
bob   = ChatUser.start("Bob")

ChatRoom.join(room, alice, "Alice")
ChatRoom.join(room, bob, "Bob")

ChatRoom.send_message(room, alice, "Hello everyone!")
ChatRoom.send_message(room, bob, "Hey Alice!")
```

---

## สรุป Part 10: Processes พื้นฐาน

✅ **Step 91** - Process คืออะไร (spawn, PID)  
✅ **Step 92** - Process communication (send/receive)  
✅ **Step 93** - Process state ผ่าน recursion  
✅ **Step 94** - Process links และ monitoring  
✅ **Step 95** - Process timing  
✅ **Step 96** - Process dictionary  
✅ **Step 97** - Named processes  
✅ **Step 98** - Process info  
✅ **Step 99** - Concurrent computations  
✅ **Step 100** - Stateful server process  

### Key Concepts ที่ต้องจำ

```
Process ใน Elixir/BEAM:
- Lightweight (~2KB memory)
- ไม่แชร์ memory (isolated)
- สื่อสารผ่าน messages เท่านั้น
- BEAM schedule processes ที่ CPU
- Crash ของ 1 process ไม่กระทบ process อื่น
- ล้าน processes ทำงานพร้อมกันได้
```

---

## แบบฝึกหัด Part 10

1. สร้าง `Counter` process ที่มี increment/decrement/reset/get
2. สร้าง `Timer` process ที่ส่ง tick message ทุก N milliseconds
3. สร้าง simple `Pub/Sub` system ด้วย processes
4. สร้าง `Pipeline` ที่ process messages ผ่าน chain ของ processes
5. Implement `rate_limiter` ที่จำกัด requests ต่อวินาที

---

## ยินดีด้วย! คุณจบระดับที่ 1 แล้ว

คุณได้เรียนรู้ foundation ของ Elixir ทั้งหมด:
- Basic types
- Pattern matching
- Functions & Modules
- Control flow
- Recursion & Enumerables
- Strings & Binaries
- **Processes** ← คือหัวใจของ Elixir!

---

➡️ [Part 11: OTP และ GenServer](./part-11-otp-genserver.md)

*ระดับที่ 2 เริ่มต้น: เราจะเรียนรู้ OTP - Erlang/Elixir's secret weapon สำหรับ building fault-tolerant systems*
