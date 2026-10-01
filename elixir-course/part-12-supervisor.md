# Part 12: Supervisor Trees (Steps 121-140)

> Supervisor คือ process ที่ทำหน้าที่ดูแลและ restart child processes
> นี่คือหัวใจของ "Let it Crash" philosophy

## Step 121: Supervisor คืออะไร?

```
Supervisor Tree ตัวอย่าง:

         Application
              │
         Supervisor (root)
        /     │      \
   Cache   Worker   Supervisor (sub)
                   /    │    \
               JobA  JobB   JobC

ถ้า Cache crash → Supervisor restart Cache เท่านั้น
ถ้า JobB crash → Sub-Supervisor restart JobB เท่านั้น
```

### 121.1 Supervisor Strategies

```elixir
# :one_for_one - restart เฉพาะที่ crash (ใช้บ่อยที่สุด)
# :one_for_all - restart ทุก child ถ้าหนึ่งตัว crash
# :rest_for_one - restart ตัวที่ crash และตัวที่ start ทีหลัง

defmodule MyApp.Supervisor do
  use Supervisor
  
  def start_link(opts) do
    Supervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @impl true
  def init(_opts) do
    children = [
      {Cache, []},
      {UserRegistry, []},
      {TaskRunner, concurrency: 5}
    ]
    
    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

---

## Step 122: ประเภทของ Child Spec

### 122.1 child_spec formats

```elixir
children = [
  # 1. Module - ใช้ default child_spec
  MyWorker,
  
  # 2. {Module, args} - ส่ง args ไปที่ start_link
  {MyWorker, [name: :worker1]},
  
  # 3. Map - explicit child spec
  %{
    id: :my_worker,
    start: {MyWorker, :start_link, [[]]},
    restart: :permanent,
    shutdown: 5000,
    type: :worker
  }
]
```

### 122.2 Restart Strategies

```elixir
# :permanent - restart เสมอ (default)
# :transient  - restart เฉพาะถ้า crash abnormally
# :temporary  - ไม่ restart เลย

defmodule BackgroundJob do
  use GenServer
  
  def child_spec(opts) do
    %{
      id: __MODULE__,
      start: {__MODULE__, :start_link, [opts]},
      restart: :transient,  # ไม่ restart ถ้า exit :normal
      shutdown: 10_000,
      type: :worker
    }
  end
end
```

---

## Step 123: DynamicSupervisor

```elixir
defmodule UserSession.Supervisor do
  use DynamicSupervisor
  
  def start_link(opts) do
    DynamicSupervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @impl true
  def init(_opts) do
    DynamicSupervisor.init(strategy: :one_for_one, max_children: 1000)
  end
  
  # เพิ่ม child แบบ dynamic
  def start_session(user_id) do
    spec = {UserSession, user_id: user_id}
    DynamicSupervisor.start_child(__MODULE__, spec)
  end
  
  # หยุด child
  def stop_session(pid) do
    DynamicSupervisor.terminate_child(__MODULE__, pid)
  end
end

defmodule UserSession do
  use GenServer
  
  def start_link(opts) do
    user_id = Keyword.fetch!(opts, :user_id)
    GenServer.start_link(__MODULE__, user_id, name: via_tuple(user_id))
  end
  
  defp via_tuple(user_id) do
    {:via, Registry, {UserSession.Registry, user_id}}
  end
  
  @impl true
  def init(user_id) do
    {:ok, %{user_id: user_id, messages: [], joined_at: DateTime.utc_now()}}
  end
end

# Application setup
defmodule MyApp do
  use Application
  
  def start(_type, _args) do
    children = [
      {Registry, keys: :unique, name: UserSession.Registry},
      UserSession.Supervisor
    ]
    
    Supervisor.start_link(children, strategy: :one_for_one, name: MyApp.Supervisor)
  end
end

# การใช้งาน
{:ok, pid1} = UserSession.Supervisor.start_session("user_1")
{:ok, pid2} = UserSession.Supervisor.start_session("user_2")

# ดู children ทั้งหมด
DynamicSupervisor.which_children(UserSession.Supervisor)

# Count
DynamicSupervisor.count_children(UserSession.Supervisor)
# %{active: 2, specs: 2, supervisors: 0, workers: 2}
```

---

## Step 124: Registry

```elixir
defmodule GameServer do
  use GenServer
  
  def start_link(game_id) do
    name = {:via, Registry, {Game.Registry, game_id}}
    GenServer.start_link(__MODULE__, game_id, name: name)
  end
  
  def get_state(game_id) do
    case Registry.lookup(Game.Registry, game_id) do
      [{pid, _}] -> GenServer.call(pid, :state)
      []         -> {:error, :not_found}
    end
  end
  
  @impl true
  def init(game_id) do
    {:ok, %{id: game_id, players: [], started: false}}
  end
  
  @impl true
  def handle_call(:state, _from, state) do
    {:reply, state, state}
  end
end

# Setup
{:ok, _} = Registry.start_link(keys: :unique, name: Game.Registry)

{:ok, _} = GameServer.start_link("game_1")
{:ok, _} = GameServer.start_link("game_2")

GameServer.get_state("game_1")
# %{id: "game_1", players: [], started: false}

# Registry.dispatch - ส่ง message ไปทุก process ที่ match
Registry.dispatch(Game.Registry, "game_1", fn entries ->
  for {pid, _val} <- entries, do: GenServer.cast(pid, :tick)
end)
```

---

## Step 125: Supervisor กับ Max Restarts

```elixir
defmodule RobustSupervisor do
  use Supervisor
  
  def start_link(opts) do
    Supervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @impl true
  def init(_opts) do
    children = [
      {CriticalService, []},
      {DataProcessor, []},
    ]
    
    # max_restarts: 3 crashes ใน max_seconds: 5 วินาที
    # ถ้าเกิน → supervisor เองก็ crash (บอกให้ parent รู้)
    Supervisor.init(children,
      strategy: :one_for_one,
      max_restarts: 3,
      max_seconds: 5
    )
  end
end
```

---

## Step 126: Supervision Tree ลึก - Chat Application

```elixir
defmodule Chat.Application do
  use Application
  
  def start(_type, _args) do
    children = [
      # Database connections
      Chat.Repo,
      
      # Registry สำหรับ rooms
      {Registry, keys: :unique, name: Chat.RoomRegistry},
      
      # Dynamic supervisor สำหรับ rooms
      Chat.Room.Supervisor,
      
      # Presence tracker
      {Chat.Presence, []},
      
      # Web server
      {Plug.Cowboy, scheme: :http, plug: Chat.Router, port: 4000}
    ]
    
    opts = [strategy: :one_for_one, name: Chat.Supervisor]
    Supervisor.start_link(children, opts)
  end
end

defmodule Chat.Room.Supervisor do
  use DynamicSupervisor
  
  def start_link(opts) do
    DynamicSupervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @impl true
  def init(_opts) do
    DynamicSupervisor.init(strategy: :one_for_one)
  end
  
  def start_room(room_id) do
    DynamicSupervisor.start_child(__MODULE__, {Chat.Room, room_id})
  end
  
  def stop_room(room_id) do
    case Registry.lookup(Chat.RoomRegistry, room_id) do
      [{pid, _}] -> DynamicSupervisor.terminate_child(__MODULE__, pid)
      []         -> {:error, :not_found}
    end
  end
end

defmodule Chat.Room do
  use GenServer
  require Logger
  
  def start_link(room_id) do
    name = {:via, Registry, {Chat.RoomRegistry, room_id}}
    GenServer.start_link(__MODULE__, room_id, name: name)
  end
  
  def join(room_id, user) do
    with_room(room_id, fn pid ->
      GenServer.call(pid, {:join, user})
    end)
  end
  
  def leave(room_id, user_id) do
    with_room(room_id, fn pid ->
      GenServer.cast(pid, {:leave, user_id})
    end)
  end
  
  def send_message(room_id, from_user, message) do
    with_room(room_id, fn pid ->
      GenServer.cast(pid, {:message, from_user, message})
    end)
  end
  
  def get_history(room_id) do
    with_room(room_id, fn pid ->
      GenServer.call(pid, :history)
    end)
  end
  
  defp with_room(room_id, func) do
    case Registry.lookup(Chat.RoomRegistry, room_id) do
      [{pid, _}] -> func.(pid)
      []         -> {:error, :room_not_found}
    end
  end
  
  @impl true
  def init(room_id) do
    Logger.info("Room #{room_id} started")
    {:ok, %{id: room_id, users: %{}, messages: [], created_at: DateTime.utc_now()}}
  end
  
  @impl true
  def handle_call({:join, user}, _from, state) do
    if Map.has_key?(state.users, user.id) do
      {:reply, {:error, :already_joined}, state}
    else
      new_users = Map.put(state.users, user.id, user)
      broadcast(state, {:user_joined, user})
      {:reply, :ok, %{state | users: new_users}}
    end
  end
  
  @impl true
  def handle_call(:history, _from, state) do
    {:reply, Enum.reverse(state.messages), state}
  end
  
  @impl true
  def handle_cast({:leave, user_id}, state) do
    case Map.pop(state.users, user_id) do
      {nil, _} -> {:noreply, state}
      {user, new_users} ->
        broadcast(state, {:user_left, user})
        {:noreply, %{state | users: new_users}}
    end
  end
  
  @impl true
  def handle_cast({:message, from_user, text}, state) do
    msg = %{
      id: generate_id(),
      from: from_user,
      text: text,
      at: DateTime.utc_now()
    }
    broadcast(state, {:new_message, msg})
    new_msgs = [msg | state.messages] |> Enum.take(100)  # keep last 100
    {:noreply, %{state | messages: new_msgs}}
  end
  
  @impl true
  def terminate(reason, state) do
    Logger.info("Room #{state.id} terminating: #{inspect(reason)}")
    :ok
  end
  
  defp broadcast(state, event) do
    Enum.each(state.users, fn {_id, user} ->
      if user.pid && Process.alive?(user.pid) do
        send(user.pid, {:room_event, state.id, event})
      end
    end)
  end
  
  defp generate_id, do: :crypto.strong_rand_bytes(8) |> Base.url_encode64(padding: false)
end
```

---

## Step 127: Process Monitoring ใน Supervisor

```elixir
defmodule ConnectionManager do
  use GenServer
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def register_connection(pid, metadata) do
    GenServer.call(__MODULE__, {:register, pid, metadata})
  end
  
  def list_connections do
    GenServer.call(__MODULE__, :list)
  end
  
  @impl true
  def init(_opts) do
    {:ok, %{connections: %{}}}
  end
  
  @impl true
  def handle_call({:register, pid, meta}, _from, state) do
    # Monitor the process
    ref = Process.monitor(pid)
    conn = %{pid: pid, metadata: meta, ref: ref, connected_at: DateTime.utc_now()}
    new_conns = Map.put(state.connections, ref, conn)
    {:reply, :ok, %{state | connections: new_conns}}
  end
  
  @impl true
  def handle_call(:list, _from, state) do
    conns = Map.values(state.connections)
    {:reply, conns, state}
  end
  
  @impl true
  def handle_info({:DOWN, ref, :process, pid, reason}, state) do
    case Map.pop(state.connections, ref) do
      {nil, _} ->
        {:noreply, state}
      
      {conn, new_conns} ->
        IO.puts("Connection #{inspect(pid)} disconnected: #{inspect(reason)}")
        IO.puts("Was connected for: #{connection_duration(conn)}")
        {:noreply, %{state | connections: new_conns}}
    end
  end
  
  defp connection_duration(conn) do
    diff = DateTime.diff(DateTime.utc_now(), conn.connected_at, :second)
    "#{diff}s"
  end
end
```

---

## Step 128: PartitionSupervisor

```elixir
# PartitionSupervisor สร้าง N copies ของ supervisor/worker
# เพื่อ horizontal scaling

defmodule EventProcessor do
  use GenServer
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: name(opts))
  end
  
  defp name(opts) do
    partition = Keyword.fetch!(opts, :partition)
    {:via, PartitionSupervisor, {__MODULE__, partition}}
  end
  
  def process(event) do
    # Route event to partition based on key
    partition = :erlang.phash2(event.user_id, System.schedulers_online())
    pid = GenServer.whereis({:via, PartitionSupervisor, {__MODULE__, partition}})
    GenServer.cast(pid, {:process, event})
  end
  
  @impl true
  def init(opts) do
    {:ok, %{partition: opts[:partition], processed: 0}}
  end
  
  @impl true
  def handle_cast({:process, event}, state) do
    do_process(event)
    {:noreply, %{state | processed: state.processed + 1}}
  end
  
  defp do_process(event) do
    # actual processing logic
    IO.inspect(event, label: "Processing")
  end
end

# Application start
children = [
  {PartitionSupervisor,
   child_spec: EventProcessor,
   name: EventProcessor,
   partitions: System.schedulers_online()}
]
```

---

## Step 129: Graceful Shutdown

```elixir
defmodule GracefulServer do
  use GenServer
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @impl true
  def init(_opts) do
    Process.flag(:trap_exit, true)
    {:ok, %{active_requests: 0, shutting_down: false}}
  end
  
  @impl true
  def handle_call(:process_request, _from, %{shutting_down: true} = state) do
    {:reply, {:error, :shutting_down}, state}
  end
  
  @impl true
  def handle_call(:process_request, from, state) do
    # เพิ่ม active request count
    new_state = %{state | active_requests: state.active_requests + 1}
    
    # Process async
    Task.start(fn ->
      result = do_work()
      GenServer.reply(from, result)
      GenServer.cast(__MODULE__, :request_done)
    end)
    
    {:noreply, new_state}
  end
  
  @impl true
  def handle_cast(:request_done, state) do
    new_count = state.active_requests - 1
    new_state = %{state | active_requests: new_count}
    
    # ถ้า shutting down และไม่มี active requests → stop
    if state.shutting_down && new_count == 0 do
      {:stop, :normal, new_state}
    else
      {:noreply, new_state}
    end
  end
  
  @impl true
  def terminate(reason, state) do
    IO.puts("Shutting down. Active requests: #{state.active_requests}")
    # Save state, close connections, etc.
    :ok
  end
  
  defp do_work do
    Process.sleep(100)
    :done
  end
end
```

---

## Step 130: Complete Fault-Tolerant System

```elixir
defmodule FaultTolerant.Application do
  use Application
  
  def start(_type, _args) do
    children = [
      # Tier 1: Infrastructure
      {Registry, keys: :unique, name: FaultTolerant.Registry},
      
      # Tier 2: Services (one_for_one - independent)
      {FaultTolerant.Supervisor,
       children: [
         FaultTolerant.DatabasePool,
         FaultTolerant.Cache,
         FaultTolerant.MessageQueue
       ],
       strategy: :one_for_one
      },
      
      # Tier 3: Workers (dynamic)
      FaultTolerant.WorkerSupervisor
    ]
    
    Supervisor.start_link(children, strategy: :one_for_one, name: __MODULE__)
  end
end

defmodule FaultTolerant.DatabasePool do
  use GenServer
  require Logger
  
  @pool_size 5
  @reconnect_interval 5_000
  
  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end
  
  @impl true
  def init([]) do
    send(self(), :connect)
    {:ok, %{connections: [], available: [], status: :connecting}}
  end
  
  @impl true
  def handle_info(:connect, state) do
    case create_connections(@pool_size) do
      {:ok, conns} ->
        Logger.info("Database pool connected (#{length(conns)} connections)")
        {:noreply, %{state | connections: conns, available: conns, status: :ready}}
      
      {:error, reason} ->
        Logger.error("DB connect failed: #{inspect(reason)}, retrying in #{@reconnect_interval}ms")
        Process.send_after(self(), :connect, @reconnect_interval)
        {:noreply, %{state | status: :reconnecting}}
    end
  end
  
  defp create_connections(n) do
    conns = Enum.map(1..n, fn i -> "conn_#{i}" end)
    {:ok, conns}  # simulated
  end
end
```

---

## สรุป Part 12

✅ **Step 121** - Supervisor strategies  
✅ **Step 122** - child_spec formats และ restart types  
✅ **Step 123** - DynamicSupervisor  
✅ **Step 124** - Registry  
✅ **Step 125** - Max restarts  
✅ **Step 126** - Chat application supervision tree  
✅ **Step 127** - Process monitoring  
✅ **Step 128** - PartitionSupervisor  
✅ **Step 129** - Graceful shutdown  
✅ **Step 130** - Complete fault-tolerant system  

➡️ [Part 13: Agent และ Task](./part-13-agent-task.md)
