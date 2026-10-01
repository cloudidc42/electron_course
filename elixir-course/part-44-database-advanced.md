# Part 44: Database Advanced (Steps 491-510)

## Step 491: Multi-Tenant Architecture

```elixir
# Row-level tenancy using schema prefix
defmodule MyApp.Repo do
  use Ecto.Repo, otp_app: :my_app, adapter: Ecto.Adapters.Postgres
  
  def with_tenant(tenant_id, fun) do
    put_dynamic_repo({:tenant, tenant_id})
    result = fun.()
    put_dynamic_repo(:default)
    result
  end
end

# Prefix-based tenancy (each tenant = separate schema)
defmodule MyApp.TenantMigrations do
  def create_tenant_schema(tenant_id) do
    schema = "tenant_#{tenant_id}"
    
    Ecto.Adapters.SQL.query!(MyApp.Repo, "CREATE SCHEMA IF NOT EXISTS #{schema}")
    
    # Run migrations for this tenant
    Ecto.Migrator.run(MyApp.Repo, migrations_path(), :up,
      all: true,
      prefix: schema
    )
  end
  
  defp migrations_path, do: Application.app_dir(:my_app, "priv/repo/migrations")
end

# Use in queries
def list_products(tenant_id) do
  Product
  |> Repo.all(prefix: "tenant_#{tenant_id}")
end

def create_product(tenant_id, attrs) do
  %Product{}
  |> Product.changeset(attrs)
  |> Repo.insert(prefix: "tenant_#{tenant_id}")
end
```

---

## Step 492: Soft Deletes

```elixir
defmodule MyApp.SoftDelete do
  import Ecto.Query
  
  defmacro __using__(_opts) do
    quote do
      import MyApp.SoftDelete
      
      def delete(record) do
        MyApp.Repo.update(Ecto.Changeset.change(record, deleted_at: DateTime.utc_now()))
      end
      
      def restore(record) do
        MyApp.Repo.update(Ecto.Changeset.change(record, deleted_at: nil))
      end
      
      def with_deleted(query) do
        exclude_deleted_scope(query, false)
      end
      
      def only_deleted(query) do
        from q in query, where: not is_nil(q.deleted_at)
      end
    end
  end
  
  def exclude_deleted(query) do
    from q in query, where: is_nil(q.deleted_at)
  end
end

defmodule MyApp.Catalog.Product do
  use Ecto.Schema
  use MyApp.SoftDelete
  
  schema "products" do
    field :name, :string
    field :deleted_at, :utc_datetime
    timestamps()
  end
end

# Ecto default scope (filter deleted automatically)
defmodule MyApp.SoftDeleteRepo do
  def all(queryable, opts \\ []) do
    queryable
    |> MyApp.SoftDelete.exclude_deleted()
    |> MyApp.Repo.all(opts)
  end
  
  def get(queryable, id, opts \\ []) do
    queryable
    |> MyApp.SoftDelete.exclude_deleted()
    |> MyApp.Repo.get(id, opts)
  end
end
```

---

## Step 493: Database Sharding

```elixir
defmodule MyApp.ShardedRepo do
  @shard_count 4
  
  def shard_for(user_id) do
    shard_id = rem(user_id, @shard_count)
    :"repo_shard_#{shard_id}"
  end
  
  def get_user(user_id) do
    repo = shard_for(user_id)
    Ecto.Repo.get(repo, MyApp.User, user_id)
  end
  
  def list_users_for_shard(shard_id) do
    repo = :"repo_shard_#{shard_id}"
    Ecto.Repo.all(repo, MyApp.User)
  end
  
  def fan_out_query(query) do
    tasks = for shard <- 0..(@shard_count - 1) do
      repo = :"repo_shard_#{shard}"
      Task.async(fn -> Ecto.Repo.all(repo, query) end)
    end
    
    tasks
    |> Task.await_many(10_000)
    |> List.flatten()
  end
end

# config/config.exs
for i <- 0..3 do
  config :"my_app_repo_shard_#{i}",
    url: "postgresql://postgres@shard#{i}:5432/my_app",
    pool_size: 5
end
```

---

## Step 494: Read Replicas

```elixir
defmodule MyApp.Repo do
  use Ecto.Repo, otp_app: :my_app, adapter: Ecto.Adapters.Postgres
end

defmodule MyApp.ReadRepo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres,
    read_only: true
end

# config/prod.exs
config :my_app, MyApp.Repo,
  url: System.get_env("PRIMARY_DB_URL"),
  pool_size: 20

config :my_app, MyApp.ReadRepo,
  url: System.get_env("REPLICA_DB_URL"),
  pool_size: 30  # more read connections

# Usage
defmodule MyApp.Products do
  # Reads from replica
  def list do
    MyApp.ReadRepo.all(Product)
  end
  
  def get(id) do
    MyApp.ReadRepo.get(Product, id)
  end
  
  # Writes to primary
  def create(attrs) do
    %Product{}
    |> Product.changeset(attrs)
    |> MyApp.Repo.insert()
  end
  
  # Read-your-writes: after a write, route to primary for a bit
  def create_and_return(attrs) do
    with {:ok, product} <- create(attrs) do
      # Fetch from primary to avoid replication lag
      {:ok, MyApp.Repo.get!(Product, product.id)}
    end
  end
end
```

---

## Step 495: Full-Text Search

```elixir
defmodule MyApp.Search do
  import Ecto.Query
  
  # PostgreSQL full-text search
  def search_products(query) do
    search_term = query |> String.trim() |> tsquery_escape()
    
    from(p in Product,
      where: fragment(
        "to_tsvector('english', ? || ' ' || ?) @@ to_tsquery('english', ?)",
        p.name,
        p.description,
        ^search_term
      ),
      order_by: fragment(
        "ts_rank(to_tsvector('english', ? || ' ' || ?), to_tsquery('english', ?)) DESC",
        p.name,
        p.description,
        ^search_term
      )
    )
    |> Repo.all()
  end
  
  # GIN index for performance
  # migration: execute "CREATE INDEX products_search_idx ON products USING gin(to_tsvector('english', name || ' ' || description))"
  
  # Using tsvector column (pre-computed)
  def search_with_column(query) do
    from(p in Product,
      where: fragment("? @@ plainto_tsquery('english', ?)", p.search_vector, ^query),
      order_by: fragment("ts_rank(?, plainto_tsquery('english', ?)) DESC", p.search_vector, ^query)
    )
    |> Repo.all()
  end
  
  # Highlight matching text
  def search_with_highlight(query) do
    from(p in Product,
      where:  fragment("to_tsvector('english', ?) @@ plainto_tsquery(?)", p.name, ^query),
      select: %{
        id:        p.id,
        name:      p.name,
        highlight: fragment(
          "ts_headline('english', ?, plainto_tsquery(?))",
          p.description,
          ^query
        )
      }
    )
    |> Repo.all()
  end
  
  defp tsquery_escape(term) do
    term
    |> String.split()
    |> Enum.map(&"#{&1}:*")
    |> Enum.join(" & ")
  end
end
```

---

## Step 496: Database Migrations Best Practices

```elixir
defmodule MyApp.Repo.Migrations.AddEmailIndex do
  use Ecto.Migration
  
  @disable_ddl_transaction true  # Allow CONCURRENTLY
  @disable_migration_lock true
  
  def up do
    # Safe for large tables (doesn't lock)
    execute "CREATE INDEX CONCURRENTLY IF NOT EXISTS users_email_idx ON users(email)"
  end
  
  def down do
    execute "DROP INDEX CONCURRENTLY IF EXISTS users_email_idx"
  end
end

defmodule MyApp.Repo.Migrations.AddNotNullColumn do
  use Ecto.Migration
  
  # Safe multi-step NOT NULL column addition:
  # Step 1: Add nullable column
  def change do
    alter table(:users) do
      add :phone, :string
    end
  end
end

# Step 2: Backfill (in a separate migration or deploy step)
# Step 3: Add NOT NULL constraint + check constraint (another migration)
defmodule MyApp.Repo.Migrations.PhoneNotNull do
  use Ecto.Migration
  
  def change do
    # Add check constraint first (validates new rows)
    execute "ALTER TABLE users ADD CONSTRAINT users_phone_not_null CHECK (phone IS NOT NULL) NOT VALID"
    # Validate existing rows (table scan, but doesn't lock)
    execute "ALTER TABLE users VALIDATE CONSTRAINT users_phone_not_null"
  end
end
```

---

## Step 497: Ecto Multi (Complex Transactions)

```elixir
defmodule MyApp.Orders do
  alias Ecto.Multi
  
  def create_order(user_id, items) do
    Multi.new()
    |> Multi.run(:validate_stock, fn _repo, _changes ->
      validate_all_in_stock(items)
    end)
    |> Multi.insert(:order, fn _changes ->
      Order.changeset(%Order{}, %{user_id: user_id, status: :pending})
    end)
    |> Multi.insert_all(:order_items, OrderItem, fn %{order: order} ->
      Enum.map(items, fn item ->
        %{
          order_id:   order.id,
          product_id: item.product_id,
          quantity:   item.quantity,
          price:      item.price
        }
      end)
    end)
    |> Multi.run(:reserve_inventory, fn _repo, %{order_items: {_, items}} ->
      reserve_inventory(items)
    end)
    |> Multi.run(:charge_payment, fn _repo, %{order: order} ->
      charge(order.total, user_id)
    end)
    |> Multi.update(:confirm_order, fn %{order: order, charge_payment: txn_id} ->
      Order.changeset(order, %{status: :confirmed, payment_id: txn_id})
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{confirm_order: order}} ->
        broadcast_order_created(order)
        {:ok, order}
      
      {:error, :validate_stock, reason, _changes} ->
        {:error, {:out_of_stock, reason}}
      
      {:error, :charge_payment, reason, _changes} ->
        {:error, {:payment_failed, reason}}
      
      {:error, step, reason, _changes} ->
        {:error, {step, reason}}
    end
  end
end
```

---

## Step 498: Database Observability

```elixir
defmodule MyApp.DBObserver do
  use GenServer
  
  @check_interval 60_000
  @slow_query_ms 1000
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def init(_) do
    setup_telemetry()
    schedule_check()
    {:ok, %{slow_queries: [], long_transactions: []}}
  end
  
  defp setup_telemetry do
    :telemetry.attach(
      "slow_query_handler",
      [:my_app, :repo, :query, :stop],
      fn _event, %{total_time: time}, meta, _config ->
        ms = System.convert_time_unit(time, :native, :millisecond)
        if ms >= @slow_query_ms do
          require Logger
          Logger.warning("Slow query (#{ms}ms): #{meta.source} - #{inspect(meta.query)}")
        end
      end,
      nil
    )
  end
  
  def handle_info(:check, state) do
    # Find slow queries in pg_stat_statements
    slow = Repo.query!("""
      SELECT query, mean_exec_time::int, calls
      FROM pg_stat_statements
      WHERE mean_exec_time > #{@slow_query_ms}
      ORDER BY mean_exec_time DESC
      LIMIT 10
    """)
    
    # Find long running transactions
    long_txns = Repo.query!("""
      SELECT pid, now() - xact_start AS duration, state, query
      FROM pg_stat_activity
      WHERE xact_start IS NOT NULL AND now() - xact_start > interval '1 minute'
    """)
    
    schedule_check()
    {:noreply, %{state | slow_queries: slow.rows, long_transactions: long_txns.rows}}
  end
  
  defp schedule_check do
    Process.send_after(self(), :check, @check_interval)
  end
end
```

---

## Step 499: TimescaleDB (Time-Series)

```elixir
# TimescaleDB = PostgreSQL extension for time-series

defmodule MyApp.Repo.Migrations.CreateMetrics do
  use Ecto.Migration
  
  def up do
    create table(:metrics, primary_key: false) do
      add :time,       :timestamptz, null: false
      add :device_id,  :string,      null: false
      add :metric,     :string,      null: false
      add :value,      :float,       null: false
    end
    
    # Convert to hypertable (TimescaleDB magic)
    execute "SELECT create_hypertable('metrics', 'time')"
    
    # Create indexes
    create index(:metrics, [:device_id, :time])
    create index(:metrics, [:metric, :time])
  end
  
  def down do
    drop table(:metrics)
  end
end

defmodule MyApp.Metrics do
  import Ecto.Query
  
  def record(device_id, metric, value) do
    Repo.insert_all(:metrics, [%{
      time:      DateTime.utc_now(),
      device_id: device_id,
      metric:    metric,
      value:     value
    }])
  end
  
  def get_hourly_average(device_id, metric, from, to) do
    from(m in "metrics",
      where: m.device_id == ^device_id and
             m.metric == ^metric and
             m.time >= ^from and
             m.time <= ^to,
      group_by: fragment("time_bucket('1 hour', ?)", m.time),
      select: %{
        bucket: fragment("time_bucket('1 hour', ?)", m.time),
        avg:    avg(m.value),
        min:    min(m.value),
        max:    max(m.value)
      },
      order_by: [fragment("time_bucket('1 hour', ?)", m.time)]
    )
    |> Repo.all()
  end
end
```

---

## Step 500: GraphQL + Database Performance

```elixir
defmodule MyApp.Dataloaders do
  def data, do: Dataloader.Ecto.new(MyApp.Repo)
  
  def context(ctx) do
    loader = Dataloader.new()
      |> Dataloader.add_source(:db, data())
    
    Map.put(ctx, :loader, loader)
  end
  
  def plugins do
    [Absinthe.Middleware.Dataloader | Absinthe.Plugin.defaults()]
  end
end

defmodule MyApp.Schema.ProductType do
  use Absinthe.Schema.Notation
  import Absinthe.Resolution.Helpers, only: [dataloader: 1]
  
  object :product do
    field :id,          :id
    field :name,        :string
    field :price,       :decimal
    
    # Use DataLoader to batch category loads
    field :category, :category do
      resolve dataloader(:db)
    end
    
    # Batch reviews count
    field :reviews_count, :integer do
      resolve fn product, _args, %{context: %{loader: loader}} ->
        loader
        |> Dataloader.load(:db, {:count, MyApp.Review}, product_id: product.id)
        |> on_load(fn loader ->
          count = Dataloader.get(loader, :db, {:count, MyApp.Review}, product_id: product.id)
          {:ok, count}
        end)
      end
    end
  end
end
```

---

## สรุป Part 44

✅ **Step 491** - Multi-tenant  
✅ **Step 492** - Soft deletes  
✅ **Step 493** - Database sharding  
✅ **Step 494** - Read replicas  
✅ **Step 495** - Full-text search  
✅ **Step 496** - Safe migrations  
✅ **Step 497** - Ecto.Multi  
✅ **Step 498** - DB observability  
✅ **Step 499** - TimescaleDB  
✅ **Step 500** - GraphQL + DataLoader  

➡️ [Part 45: Capstone Project](./part-45-capstone.md)
