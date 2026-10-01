# Part 14: ETS และ Mnesia (Steps 151-170)

## Step 151: ETS - Erlang Term Storage

```
ETS (Erlang Term Storage)
- In-memory key-value store ที่สร้างโดย Erlang
- ใช้ร่วมกันระหว่าง processes ได้
- เร็วมาก: O(1) reads/writes
- Concurrent reads (by default)
- ไม่ใช่ persistent (หายเมื่อ process owner crash)
```

### 151.1 สร้างและใช้ ETS

```elixir
# สร้าง ETS table
table = :ets.new(:my_table, [
  :set,          # table type
  :public,       # access control
  :named_table   # ใช้ atom เป็นชื่อ
])

# Insert
:ets.insert(:my_table, {:key1, "value1"})
:ets.insert(:my_table, {:key2, "value2"})
:ets.insert(:my_table, [{:k3, "v3"}, {:k4, "v4"}])  # bulk insert

# Lookup
:ets.lookup(:my_table, :key1)   # [{:key1, "value1"}]
:ets.lookup(:my_table, :missing) # []

# Delete
:ets.delete(:my_table, :key1)

# All entries
:ets.tab2list(:my_table)

# Count
:ets.info(:my_table, :size)
```

---

## Step 152: ETS Table Types

```elixir
# :set - unique keys (default)
:ets.new(:set_table, [:set])

# :bag - multiple values per key
:ets.new(:bag_table, [:bag])
:ets.insert(:bag_table, {:user, "alice"})
:ets.insert(:bag_table, {:user, "bob"})
:ets.lookup(:bag_table, :user)  # [{:user, "alice"}, {:user, "bob"}]

# :duplicate_bag - allows duplicate entries
:ets.new(:dup_table, [:duplicate_bag])

# :ordered_set - sorted by key
:ets.new(:ordered, [:ordered_set])
:ets.insert(:ordered, {3, "c"})
:ets.insert(:ordered, {1, "a"})
:ets.insert(:ordered, {2, "b"})
:ets.tab2list(:ordered)  # [{1,"a"}, {2,"b"}, {3,"c"}]
```

---

## Step 153: ETS Access Control

```elixir
# :public    - ทุก process อ่าน/เขียนได้
# :protected - ทุก process อ่านได้ เฉพาะ owner เขียนได้ (default)
# :private   - เฉพาะ owner เท่านั้น

# ใช้ ETS กับ GenServer (owner)
defmodule CacheWithETS do
  use GenServer
  
  @table :cache_table
  
  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end
  
  # Public API (อ่านตรงจาก ETS ไม่ต้องผ่าน GenServer!)
  def get(key) do
    case :ets.lookup(@table, key) do
      [{^key, value, expires_at}] ->
        if System.monotonic_time(:millisecond) < expires_at do
          {:ok, value}
        else
          :ets.delete(@table, key)
          :miss
        end
      [] -> :miss
    end
  end
  
  # Writes ต้องผ่าน GenServer (เพื่อ serialization)
  def put(key, value, ttl \\ 60_000) do
    GenServer.cast(__MODULE__, {:put, key, value, ttl})
  end
  
  @impl true
  def init([]) do
    # สร้าง table ตอน init
    :ets.new(@table, [:named_table, :set, :public, read_concurrency: true])
    {:ok, %{}}
  end
  
  @impl true
  def handle_cast({:put, key, value, ttl}, state) do
    expires_at = System.monotonic_time(:millisecond) + ttl
    :ets.insert(@table, {key, value, expires_at})
    {:noreply, state}
  end
  
  @impl true
  def terminate(_reason, _state) do
    :ets.delete(@table)
    :ok
  end
end
```

---

## Step 154: ETS Pattern Matching

```elixir
# :ets.match - match กับ pattern
:ets.insert(:users, {:user1, "Alice", :admin, 30})
:ets.insert(:users, {:user2, "Bob",   :user,  25})
:ets.insert(:users, {:user3, "Carol", :admin, 35})

# Match admins (ใช้ :_ สำหรับ wildcard)
:ets.match(:users, {:"$1", :"$2", :admin, :"$3"})
# [[:user1, "Alice", 30], [:user3, "Carol", 35]]

# :ets.match_object - return full records
:ets.match_object(:users, {:_, :_, :admin, :_})
# [{:user1, "Alice", :admin, 30}, {:user3, "Carol", :admin, 35}]

# :ets.select - with match spec
match_spec = [
  {
    {:_, :"$1", :admin, :"$2"},    # pattern
    [{:>, :"$2", 29}],             # guard: age > 29
    [:"$1"]                         # return: name
  }
]
:ets.select(:users, match_spec)  # ["Alice", "Carol"]

# :ets.select_count
:ets.select_count(:users, [{{:_, :_, :admin, :_}, [], [true]}])  # 2
```

---

## Step 155: ETS สำหรับ Counter/Statistics

```elixir
defmodule Stats do
  @table :stats
  
  def setup do
    :ets.new(@table, [:named_table, :set, :public,
                      write_concurrency: true,
                      read_concurrency: true])
  end
  
  # Atomic increment (thread-safe)
  def increment(key, amount \\ 1) do
    :ets.update_counter(@table, key, {2, amount}, {key, 0})
  end
  
  def get(key) do
    case :ets.lookup(@table, key) do
      [{^key, value}] -> value
      []              -> 0
    end
  end
  
  def reset(key) do
    :ets.insert(@table, {key, 0})
  end
  
  def all do
    :ets.tab2list(@table)
    |> Map.new(fn {k, v} -> {k, v} end)
  end
end

# Usage
Stats.setup()
Stats.increment(:page_views)
Stats.increment(:page_views)
Stats.increment(:api_calls, 5)
Stats.get(:page_views)  # 2
Stats.all()             # %{page_views: 2, api_calls: 5}
```

---

## Step 156: :ets.fun2ms - Match Spec Helper

```elixir
require Record

# ใช้ :ets.fun2ms เพื่อสร้าง match spec จาก function
# ต้องใช้ record syntax

defmodule UserTable do
  @table :users_v2
  
  def setup do
    :ets.new(@table, [:named_table, :set, :public])
  end
  
  def insert(user) do
    :ets.insert(@table, {user.id, user.name, user.age, user.role})
  end
  
  def find_adults do
    ms = :ets.fun2ms(fn {id, name, age, _role} when age >= 18 ->
      {id, name}
    end)
    :ets.select(@table, ms)
  end
  
  def find_by_role(role) do
    :ets.match_object(@table, {:_, :_, :_, role})
    |> Enum.map(fn {id, name, age, r} ->
         %{id: id, name: name, age: age, role: r}
       end)
  end
end
```

---

## Step 157: Mnesia - Distributed Database

```elixir
# Mnesia คือ distributed database ที่ built-in ใน Erlang
# - ACID transactions
# - Persistent storage (disk)
# - Replication across nodes
# - ใช้ Erlang records/tuples เป็น schema

# Setup Mnesia
:mnesia.create_schema([node()])
:mnesia.start()

# สร้าง table
:mnesia.create_table(:users, [
  attributes: [:id, :name, :email, :age],
  disc_copies: [node()],     # persistent + in-memory
  type: :set
])
```

---

## Step 158: Mnesia CRUD

```elixir
defmodule UserDB do
  @table :users
  
  def setup do
    :mnesia.create_schema([node()])
    :mnesia.start()
    
    :mnesia.create_table(@table, [
      attributes: [:id, :name, :email, :created_at],
      disc_copies: [node()],
      type: :set
    ])
  end
  
  def insert(user) do
    record = {@table, user.id, user.name, user.email, DateTime.utc_now()}
    
    :mnesia.transaction(fn ->
      :mnesia.write(record)
    end)
    |> handle_result()
  end
  
  def get(id) do
    :mnesia.transaction(fn ->
      :mnesia.read({@table, id})
    end)
    |> case do
         {:atomic, [{@table, id, name, email, created_at}]} ->
           {:ok, %{id: id, name: name, email: email, created_at: created_at}}
         {:atomic, []} ->
           {:error, :not_found}
         {:aborted, reason} ->
           {:error, reason}
       end
  end
  
  def delete(id) do
    :mnesia.transaction(fn ->
      :mnesia.delete({@table, id})
    end)
    |> handle_result()
  end
  
  def all do
    :mnesia.transaction(fn ->
      :mnesia.match_object({@table, :_, :_, :_, :_})
    end)
    |> case do
         {:atomic, records} ->
           users = Enum.map(records, fn {@table, id, name, email, created_at} ->
             %{id: id, name: name, email: email, created_at: created_at}
           end)
           {:ok, users}
         {:aborted, reason} ->
           {:error, reason}
       end
  end
  
  defp handle_result({:atomic, _}), do: :ok
  defp handle_result({:aborted, reason}), do: {:error, reason}
end
```

---

## Step 159: Mnesia Transactions

```elixir
defmodule BankDB do
  def transfer(from_id, to_id, amount) do
    :mnesia.transaction(fn ->
      # อ่าน accounts ใน transaction (เพื่อ consistency)
      [{:accounts, ^from_id, from_name, from_balance}] = 
        :mnesia.read({:accounts, from_id})
      [{:accounts, ^to_id, to_name, to_balance}] = 
        :mnesia.read({:accounts, to_id})
      
      if from_balance < amount do
        :mnesia.abort(:insufficient_funds)
      end
      
      :mnesia.write({:accounts, from_id, from_name, from_balance - amount})
      :mnesia.write({:accounts, to_id, to_name, to_balance + amount})
      
      %{from: from_name, to: to_name, amount: amount, transferred: true}
    end)
    |> case do
         {:atomic, result}              -> {:ok, result}
         {:aborted, :insufficient_funds} -> {:error, :insufficient_funds}
         {:aborted, reason}             -> {:error, reason}
       end
  end
  
  def dirty_read(id) do
    # dirty_read ไม่ใช้ transaction - เร็วกว่าแต่ consistency ต่ำกว่า
    case :mnesia.dirty_read({:accounts, id}) do
      [{:accounts, ^id, name, balance}] -> {:ok, %{name: name, balance: balance}}
      [] -> {:error, :not_found}
    end
  end
end
```

---

## Step 160: ETS + Mnesia ใน Production App

```elixir
defmodule ProductCatalog do
  @moduledoc """
  ใช้ ETS สำหรับ hot cache, Mnesia สำหรับ persistence
  """
  
  use GenServer
  
  @ets_table :product_cache
  @mnesia_table :products
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def get_product(id) do
    # ลอง cache ก่อน
    case ets_lookup(id) do
      {:ok, product} ->
        product
      :miss ->
        # ถ้า miss → ดึงจาก Mnesia แล้ว warm cache
        case mnesia_get(id) do
          {:ok, product} ->
            ets_put(id, product)
            product
          {:error, :not_found} ->
            nil
        end
    end
  end
  
  def create_product(attrs) do
    GenServer.call(__MODULE__, {:create, attrs})
  end
  
  def update_product(id, attrs) do
    GenServer.call(__MODULE__, {:update, id, attrs})
  end
  
  @impl true
  def init(_opts) do
    # สร้าง ETS table
    :ets.new(@ets_table, [
      :named_table, :set, :public,
      read_concurrency: true
    ])
    
    {:ok, %{}}
  end
  
  @impl true
  def handle_call({:create, attrs}, _from, state) do
    id = generate_id()
    product = Map.put(attrs, :id, id)
    
    case mnesia_insert(product) do
      :ok ->
        ets_put(id, product)
        {:reply, {:ok, product}, state}
      {:error, reason} ->
        {:reply, {:error, reason}, state}
    end
  end
  
  @impl true
  def handle_call({:update, id, attrs}, _from, state) do
    case mnesia_get(id) do
      {:ok, product} ->
        updated = Map.merge(product, attrs)
        mnesia_update(updated)
        ets_put(id, updated)
        {:reply, {:ok, updated}, state}
      {:error, :not_found} ->
        {:reply, {:error, :not_found}, state}
    end
  end
  
  # ETS operations
  defp ets_lookup(id) do
    case :ets.lookup(@ets_table, id) do
      [{^id, product}] -> {:ok, product}
      []               -> :miss
    end
  end
  
  defp ets_put(id, product) do
    :ets.insert(@ets_table, {id, product})
  end
  
  # Mnesia operations
  defp mnesia_get(id) do
    case :mnesia.dirty_read({@mnesia_table, id}) do
      [{@mnesia_table, ^id, data}] -> {:ok, data}
      []                           -> {:error, :not_found}
    end
  end
  
  defp mnesia_insert(product) do
    :mnesia.transaction(fn ->
      :mnesia.write({@mnesia_table, product.id, product})
    end)
    |> case do
         {:atomic, _}      -> :ok
         {:aborted, reason} -> {:error, reason}
       end
  end
  
  defp mnesia_update(product) do
    mnesia_insert(product)
  end
  
  defp generate_id do
    :crypto.strong_rand_bytes(8) |> Base.url_encode64(padding: false)
  end
end
```

---

## สรุป Part 14

✅ **Step 151** - ETS basics  
✅ **Step 152** - ETS table types  
✅ **Step 153** - ETS access control + GenServer  
✅ **Step 154** - ETS pattern matching และ match specs  
✅ **Step 155** - ETS สำหรับ atomic counters  
✅ **Step 156** - fun2ms helper  
✅ **Step 157** - Mnesia setup  
✅ **Step 158** - Mnesia CRUD  
✅ **Step 159** - Mnesia transactions  
✅ **Step 160** - ETS + Mnesia production pattern  

➡️ [Part 15: Mix Projects และ Dependencies](./part-15-mix-deps.md)
