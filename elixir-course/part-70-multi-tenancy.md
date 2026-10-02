# Part 70: Multi-Tenancy (Steps 761-780)

## Step 761: Multi-Tenancy Strategies

```
Strategy 1: Row-Level (Shared Schema)
  ┌─────────────────────────────────┐
  │ users                           │
  │ id | tenant_id | email | ...    │ ← tenant_id on every row
  └─────────────────────────────────┘
  ✅ Simple, cost-effective
  ❌ Risk of data leaks, no isolation

Strategy 2: Schema-Per-Tenant
  ┌────────────┐  ┌────────────┐
  │ tenant_a.  │  │ tenant_b.  │
  │   users    │  │   users    │ ← separate PostgreSQL schemas
  └────────────┘  └────────────┘
  ✅ Isolation, easy migration per-tenant
  ❌ Schema management complexity

Strategy 3: Database-Per-Tenant
  ┌────────────┐  ┌────────────┐
  │  db_a      │  │  db_b      │ ← separate databases
  └────────────┘  └────────────┘
  ✅ Complete isolation
  ❌ High operational cost

Elixir recommendation: Strategy 2 (schema-per-tenant)
using triplex or custom prefix logic with Ecto
```

---

## Step 762: Tenant Model

```elixir
defmodule MyApp.Tenants.Tenant do
  use Ecto.Schema
  import Ecto.Changeset

  schema "tenants" do
    field :name,       :string
    field :slug,       :string
    field :plan,       :string, default: "free"
    field :active,     :boolean, default: true
    field :schema,     :string
    field :settings,   :map, default: %{}
    timestamps()
  end

  def changeset(tenant, attrs) do
    tenant
    |> cast(attrs, [:name, :slug, :plan, :settings])
    |> validate_required([:name, :slug])
    |> validate_format(:slug, ~r/^[a-z0-9\-]+$/)
    |> unique_constraint(:slug)
    |> put_schema()
  end

  defp put_schema(changeset) do
    case get_field(changeset, :slug) do
      nil  -> changeset
      slug -> put_change(changeset, :schema, "tenant_#{String.replace(slug, "-", "_")}")
    end
  end
end

defmodule MyApp.Tenants do
  alias MyApp.{Repo, Tenants.Tenant}

  def create_tenant(attrs) do
    Repo.transaction(fn ->
      with {:ok, tenant} <- Repo.insert(Tenant.changeset(%Tenant{}, attrs)),
           :ok           <- create_schema(tenant),
           :ok           <- run_migrations(tenant) do
        tenant
      else
        {:error, reason} -> Repo.rollback(reason)
      end
    end)
  end

  defp create_schema(%{schema: schema}) do
    Repo.query!("CREATE SCHEMA IF NOT EXISTS #{schema}")
    :ok
  end

  defp run_migrations(%{schema: schema}) do
    Ecto.Migrator.run(Repo, :up, all: true, prefix: schema)
    :ok
  end

  def get_by_slug(slug) do
    Repo.get_by(Tenant, slug: slug, active: true)
  end
end
```

---

## Step 763: Tenant-Scoped Ecto Queries

```elixir
defmodule MyApp.TenantRepo do
  # Wrap all Ecto operations with tenant prefix

  def all(queryable, tenant, opts \\ []) do
    MyApp.Repo.all(queryable, prefix: tenant.schema)
  end

  def get(queryable, id, tenant, opts \\ []) do
    MyApp.Repo.get(queryable, id, prefix: tenant.schema)
  end

  def insert(changeset, tenant, opts \\ []) do
    MyApp.Repo.insert(changeset, prefix: tenant.schema)
  end

  def update(changeset, tenant, opts \\ []) do
    MyApp.Repo.update(changeset, prefix: tenant.schema)
  end

  def delete(struct, tenant, opts \\ []) do
    MyApp.Repo.delete(struct, prefix: tenant.schema)
  end

  def transaction(fun, tenant) do
    MyApp.Repo.transaction(fn ->
      fun.(tenant)
    end)
  end
end

# Usage
def list_users(tenant) do
  MyApp.Accounts.User
  |> MyApp.TenantRepo.all(tenant)
end

def get_user(id, tenant) do
  MyApp.Accounts.User
  |> MyApp.TenantRepo.get(id, tenant)
end
```

---

## Step 764: Tenant Detection Plug

```elixir
defmodule MyAppWeb.Plugs.DetectTenant do
  import Plug.Conn
  import Phoenix.Controller, only: [render: 3, put_view: 2]

  def init(opts), do: opts

  def call(conn, _opts) do
    tenant = detect_tenant(conn)

    case tenant do
      nil ->
        conn
        |> put_status(404)
        |> put_view(MyAppWeb.ErrorHTML)
        |> render(:"404")
        |> halt()

      tenant ->
        conn
        |> assign(:current_tenant, tenant)
        |> put_tenant_in_session(tenant)
    end
  end

  defp detect_tenant(conn) do
    # Strategy 1: Subdomain
    subdomain = conn.host |> String.split(".") |> List.first()
    
    case MyApp.Tenants.get_by_slug(subdomain) do
      nil -> detect_from_header(conn)
      t   -> t
    end
  end

  defp detect_from_header(conn) do
    # Strategy 2: X-Tenant-ID header (for API)
    case get_req_header(conn, "x-tenant-id") do
      [slug | _] -> MyApp.Tenants.get_by_slug(slug)
      []         -> nil
    end
  end

  defp put_tenant_in_session(conn, tenant) do
    put_session(conn, :tenant_id, tenant.id)
  end
end

# Router setup
pipeline :tenant_required do
  plug MyAppWeb.Plugs.DetectTenant
end

scope "/" do
  pipe_through [:browser, :tenant_required]
  # tenant-scoped routes
end
```

---

## Step 765: Tenant Context

```elixir
defmodule MyApp.TenantContext do
  # Process dictionary for current tenant
  
  def set(tenant) do
    Process.put(:current_tenant, tenant)
  end

  def get do
    Process.get(:current_tenant)
  end

  def clear do
    Process.delete(:current_tenant)
  end

  def with_tenant(tenant, fun) do
    set(tenant)
    try do
      fun.()
    after
      clear()
    end
  end
end

# Middleware to set tenant context for background jobs
defmodule MyApp.Workers.TenantWorker do
  use Oban.Worker

  def perform(%Oban.Job{args: %{"tenant_id" => tenant_id} = args}) do
    tenant = MyApp.Repo.get!(MyApp.Tenants.Tenant, tenant_id)
    
    MyApp.TenantContext.with_tenant(tenant, fn ->
      do_work(args)
    end)
  end
end
```

---

## Step 766: Per-Tenant Configurations

```elixir
defmodule MyApp.TenantConfig do
  alias MyApp.Repo
  alias MyApp.Tenants.TenantSetting

  def get(tenant, key, default \\ nil) do
    case Repo.get_by(TenantSetting, tenant_id: tenant.id, key: to_string(key)) do
      nil     -> default
      setting -> setting.value
    end
  end

  def set(tenant, key, value) do
    %TenantSetting{}
    |> TenantSetting.changeset(%{tenant_id: tenant.id, key: to_string(key), value: value})
    |> Repo.insert(
      on_conflict: {:replace, [:value, :updated_at]},
      conflict_target: [:tenant_id, :key]
    )
  end

  def all_for(tenant) do
    Repo.all(
      from s in TenantSetting,
      where: s.tenant_id == ^tenant.id
    )
    |> Map.new(fn s -> {s.key, s.value} end)
  end
end

# Usage
tenant = conn.assigns.current_tenant

max_users = MyApp.TenantConfig.get(tenant, :max_users, 5)
features  = MyApp.TenantConfig.get(tenant, :features, [])

if "advanced_reporting" in features do
  # show reports
end
```

---

## Step 767: Tenant Isolation Tests

```elixir
defmodule MyApp.MultiTenancyTest do
  use MyApp.DataCase

  setup do
    {:ok, tenant_a} = MyApp.Tenants.create_tenant(%{name: "Tenant A", slug: "tenant-a"})
    {:ok, tenant_b} = MyApp.Tenants.create_tenant(%{name: "Tenant B", slug: "tenant-b"})
    {:ok, tenant_a: tenant_a, tenant_b: tenant_b}
  end

  test "users are isolated between tenants", %{tenant_a: ta, tenant_b: tb} do
    user_a = insert(:user, prefix: ta.schema)
    user_b = insert(:user, prefix: tb.schema)

    users_a = MyApp.TenantRepo.all(MyApp.User, ta)
    users_b = MyApp.TenantRepo.all(MyApp.User, tb)

    assert length(users_a) == 1
    assert length(users_b) == 1
    assert Enum.map(users_a, & &1.id) != Enum.map(users_b, & &1.id)
  end

  test "cannot access another tenant's data directly" do
    # Without proper scoping, queries must fail or return empty
    user_a = insert(:user, prefix: ta.schema)
    
    # Without tenant scope, should not find user
    assert MyApp.TenantRepo.get(MyApp.User, user_a.id, tb) == nil
  end
end
```

---

## Step 768: Tenant Billing

```elixir
defmodule MyApp.TenantBilling do
  alias MyApp.{Repo, Tenants.Tenant}

  def usage_for(tenant) do
    %{
      users:      count_users(tenant),
      storage_mb: calculate_storage(tenant),
      api_calls:  api_calls_this_month(tenant)
    }
  end

  def check_limits(tenant, resource) do
    limits = plan_limits(tenant.plan)
    usage  = usage_for(tenant)

    current_usage = Map.get(usage, resource, 0)
    limit         = Map.get(limits, resource, :unlimited)

    case limit do
      :unlimited -> :ok
      max when current_usage >= max ->
        {:error, {:limit_reached, resource, max}}
      _ -> :ok
    end
  end

  defp plan_limits("free"),       do: %{users: 5,   storage_mb: 100, api_calls: 1000}
  defp plan_limits("starter"),    do: %{users: 25,  storage_mb: 1000, api_calls: 10000}
  defp plan_limits("pro"),        do: %{users: 100, storage_mb: 5000, api_calls: 100000}
  defp plan_limits("enterprise"), do: %{users: :unlimited, storage_mb: :unlimited, api_calls: :unlimited}

  defp count_users(tenant) do
    Repo.aggregate(MyApp.User, :count, prefix: tenant.schema)
  end

  defp calculate_storage(tenant) do
    # Sum file sizes
    Repo.one(
      from f in MyApp.File,
        select: sum(f.size),
        prefix: ^tenant.schema
    ) |> Kernel.||(0) |> Kernel.div(1024 * 1024)
  end

  defp api_calls_this_month(tenant) do
    start = DateTime.utc_now() |> DateTime.beginning_of_month()
    Repo.aggregate(
      from(l in MyApp.ApiLog, where: l.inserted_at >= ^start),
      :count,
      prefix: tenant.schema
    )
  end
end
```

---

## Step 769: Tenant Migration Strategy

```elixir
defmodule MyApp.TenantMigrator do
  # Run migrations across all tenants

  def migrate_all(direction \\ :up) do
    tenants = MyApp.Repo.all(MyApp.Tenants.Tenant)
    
    results = Enum.map(tenants, fn tenant ->
      result = case direction do
        :up   -> Ecto.Migrator.run(MyApp.Repo, :up, all: true, prefix: tenant.schema)
        :down -> Ecto.Migrator.run(MyApp.Repo, :down, all: true, prefix: tenant.schema)
      end
      {tenant.slug, result}
    end)

    {successes, failures} = Enum.split_with(results, fn {_, r} -> match?({:ok, _, _}, r) end)
    
    %{
      total:     length(results),
      succeeded: length(successes),
      failed:    length(failures),
      errors:    failures
    }
  end

  def migrate_tenant(slug, direction \\ :up) do
    with {:ok, tenant} <- MyApp.Tenants.get_by_slug(slug) do
      Ecto.Migrator.run(MyApp.Repo, direction, all: true, prefix: tenant.schema)
    end
  end
end
```

---

## Step 770: Cross-Tenant Analytics

```elixir
defmodule MyApp.AdminAnalytics do
  alias MyApp.Repo

  def tenant_health_metrics do
    tenants = Repo.all(MyApp.Tenants.Tenant)

    Task.async_stream(tenants, fn tenant ->
      %{
        slug:         tenant.slug,
        plan:         tenant.plan,
        user_count:   Repo.aggregate(MyApp.User, :count, prefix: tenant.schema),
        active_30d:   active_users_last_30d(tenant),
        storage_mb:   storage_mb(tenant)
      }
    end, max_concurrency: 10, timeout: 30_000)
    |> Enum.map(fn {:ok, m} -> m end)
  end

  def revenue_by_plan do
    Repo.all(
      from t in MyApp.Tenants.Tenant,
        where: t.active == true,
        group_by: t.plan,
        select: {t.plan, count(t.id)}
    )
    |> Enum.into(%{})
  end

  defp active_users_last_30d(tenant) do
    since = DateTime.add(DateTime.utc_now(), -30, :day)
    Repo.aggregate(
      from(u in MyApp.User, where: u.last_active_at >= ^since),
      :count,
      prefix: tenant.schema
    )
  end

  defp storage_mb(tenant) do
    Repo.one(
      from(f in MyApp.File, select: sum(f.size), prefix: ^tenant.schema)
    )
    |> Kernel.||(0)
    |> Kernel.div(1_048_576)
  end
end
```

---

## สรุป Part 70

✅ **Step 761** - Multi-tenancy strategies  
✅ **Step 762** - Tenant model  
✅ **Step 763** - Tenant-scoped queries  
✅ **Step 764** - Tenant detection plug  
✅ **Step 765** - Tenant context  
✅ **Step 766** - Per-tenant configurations  
✅ **Step 767** - Isolation tests  
✅ **Step 768** - Tenant billing  
✅ **Step 769** - Migration strategy  
✅ **Step 770** - Cross-tenant analytics  

➡️ [Part 71: Advanced Security](./part-71-security.md)
