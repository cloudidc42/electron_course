# Part 56: SaaS Architecture (Steps 611-630)

## Step 611: Multi-Tenant SaaS Setup

```elixir
# SaaS with plan-based feature gating and per-tenant isolation
defmodule MyApp.Tenants do
  use Ecto.Schema
  import Ecto.Changeset

  schema "tenants" do
    field :name,       :string
    field :slug,       :string
    field :plan,       Ecto.Enum, values: [:free, :starter, :pro, :enterprise]
    field :trial_ends, :utc_datetime
    field :active,     :boolean, default: true
    timestamps()
  end

  def changeset(tenant, attrs) do
    tenant
    |> cast(attrs, [:name, :slug, :plan])
    |> validate_required([:name, :slug])
    |> validate_format(:slug, ~r/^[a-z0-9\-]+$/)
    |> unique_constraint(:slug)
  end
end

# Plug: resolve tenant from subdomain
defmodule MyAppWeb.Plugs.ResolveTenant do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    subdomain = conn.host
      |> String.split(".")
      |> List.first()

    case MyApp.Tenants.get_by_slug(subdomain) do
      nil ->
        conn
        |> send_resp(404, "Tenant not found")
        |> halt()
      tenant ->
        assign(conn, :current_tenant, tenant)
    end
  end
end
```

---

## Step 612: Plan-Based Feature Flags

```elixir
defmodule MyApp.Plans do
  @features %{
    free:       [:basic_reports, :up_to_5_users, :1gb_storage],
    starter:    [:advanced_reports, :up_to_25_users, :10gb_storage, :api_access],
    pro:        [:custom_reports, :unlimited_users, :100gb_storage, :api_access, :webhooks],
    enterprise: [:all_features, :sso, :audit_log, :custom_domain, :priority_support]
  }

  def can?(tenant, feature) do
    plan_features = Map.get(@features, tenant.plan, [])
    feature in plan_features or :all_features in plan_features
  end

  def require_plan!(tenant, feature) do
    unless can?(tenant, feature) do
      raise MyApp.PlanUpgradeRequired, feature: feature, current_plan: tenant.plan
    end
  end

  def plan_limit(tenant, :users) do
    case tenant.plan do
      :free       -> 5
      :starter    -> 25
      :pro        -> :unlimited
      :enterprise -> :unlimited
    end
  end
end

# Usage in controller
defmodule MyAppWeb.WebhooksController do
  use MyAppWeb, :controller

  def create(conn, params) do
    tenant = conn.assigns.current_tenant
    
    if MyApp.Plans.can?(tenant, :webhooks) do
      # create webhook
    else
      conn
      |> put_status(402)
      |> json(%{error: "Webhooks require Pro plan", upgrade_url: "/billing"})
    end
  end
end
```

---

## Step 613: Billing with Stripe

```elixir
defmodule MyApp.Billing do
  # mix.exs: {:stripity_stripe, "~> 2.17"}

  def create_subscription(tenant, plan) do
    price_id = get_stripe_price_id(plan)

    case Stripe.Subscription.create(%{
      customer: tenant.stripe_customer_id,
      items: [%{price: price_id}],
      payment_behavior: "default_incomplete",
      expand: ["latest_invoice.payment_intent"]
    }) do
      {:ok, subscription} ->
        update_tenant_plan(tenant, plan, subscription.id)
        {:ok, subscription}
      {:error, e} ->
        {:error, e}
    end
  end

  def cancel_subscription(tenant) do
    case Stripe.Subscription.cancel(tenant.stripe_subscription_id) do
      {:ok, _sub} ->
        update_tenant_plan(tenant, :free, nil)
        :ok
      {:error, e} ->
        {:error, e}
    end
  end

  # Webhook handler
  def handle_webhook("customer.subscription.updated", event) do
    sub = event.data.object
    tenant = get_tenant_by_stripe_id(sub.customer)
    plan = stripe_price_to_plan(sub.items.data |> List.first() |> Map.get(:price) |> Map.get(:id))
    update_tenant_plan(tenant, plan, sub.id)
  end

  def handle_webhook("invoice.payment_failed", event) do
    customer_id = event.data.object.customer
    tenant = get_tenant_by_stripe_id(customer_id)
    notify_payment_failure(tenant)
  end

  def handle_webhook(_, _event), do: :ok

  defp get_stripe_price_id(:starter),    do: System.get_env("STRIPE_STARTER_PRICE_ID")
  defp get_stripe_price_id(:pro),        do: System.get_env("STRIPE_PRO_PRICE_ID")
  defp get_stripe_price_id(:enterprise), do: System.get_env("STRIPE_ENTERPRISE_PRICE_ID")
end
```

---

## Step 614: Usage Metering

```elixir
defmodule MyApp.UsageMetering do
  import Ecto.Query

  def record_usage(tenant_id, metric, amount \\ 1) do
    now = Date.utc_today()
    
    MyApp.Repo.insert_all(
      "usage_records",
      [%{tenant_id: tenant_id, metric: metric, amount: amount, date: now}],
      on_conflict: {:unsafe_fragment, "amount = usage_records.amount + EXCLUDED.amount"},
      conflict_target: [:tenant_id, :metric, :date]
    )
  end

  def get_usage(tenant_id, metric, period \\ :current_month) do
    {start_date, end_date} = date_range(period)
    
    MyApp.Repo.one(
      from u in "usage_records",
      where: u.tenant_id == ^tenant_id and
             u.metric    == ^metric and
             u.date >= ^start_date and
             u.date <= ^end_date,
      select: sum(u.amount)
    ) || 0
  end

  def within_limit?(tenant, metric) do
    current_usage = get_usage(tenant.id, metric)
    limit = MyApp.Plans.plan_limit(tenant, metric)
    
    case limit do
      :unlimited -> true
      n          -> current_usage < n
    end
  end

  defp date_range(:current_month) do
    today = Date.utc_today()
    start = Date.beginning_of_month(today)
    finish = Date.end_of_month(today)
    {start, finish}
  end
end
```

---

## Step 615: Tenant Data Isolation

```elixir
defmodule MyApp.TenantScope do
  import Ecto.Query

  defmacro scoped(schema, tenant_id) do
    quote do
      from q in unquote(schema), where: q.tenant_id == ^unquote(tenant_id)
    end
  end

  # Ensure all queries are tenant-scoped
  def scope(queryable, tenant_id) do
    queryable
    |> where(tenant_id: ^tenant_id)
  end
end

# All schemas include tenant_id
defmodule MyApp.Projects.Project do
  use Ecto.Schema

  schema "projects" do
    field :name,      :string
    field :tenant_id, :integer
    timestamps()
  end

  def for_tenant(tenant_id), do: where(__MODULE__, tenant_id: ^tenant_id)
end

# Controller automatically scopes queries to current tenant
defmodule MyAppWeb.ProjectsController do
  use MyAppWeb, :controller

  def index(conn, _params) do
    tenant = conn.assigns.current_tenant
    projects = MyApp.Projects.list_for_tenant(tenant.id)
    json(conn, %{projects: projects})
  end

  def show(conn, %{"id" => id}) do
    tenant = conn.assigns.current_tenant

    # Always scope by tenant to prevent cross-tenant access
    case MyApp.Projects.get_for_tenant(id, tenant.id) do
      nil     -> send_resp(conn, 404, "Not found")
      project -> json(conn, project)
    end
  end
end
```

---

## Step 616: SaaS Onboarding Flow

```elixir
defmodule MyApp.Onboarding do
  alias MyApp.{Repo, Tenants, Accounts, Billing}
  alias Ecto.Multi

  def register_tenant(attrs) do
    Multi.new()
    |> Multi.insert(:tenant, Tenants.changeset(%Tenants.Tenant{}, attrs))
    |> Multi.run(:owner, fn _repo, %{tenant: tenant} ->
      Accounts.create_user(%{
        name:       attrs[:owner_name],
        email:      attrs[:owner_email],
        password:   attrs[:password],
        tenant_id:  tenant.id,
        role:       :owner
      })
    end)
    |> Multi.run(:stripe_customer, fn _repo, %{owner: owner, tenant: tenant} ->
      Billing.create_stripe_customer(owner.email, tenant.name)
    end)
    |> Multi.run(:trial, fn _repo, %{tenant: tenant, stripe_customer: customer} ->
      trial_ends = DateTime.add(DateTime.utc_now(), 14 * 86400)
      Tenants.update(tenant, %{
        plan:               :pro,  # Start on pro trial
        stripe_customer_id: customer.id,
        trial_ends:         trial_ends
      })
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{tenant: tenant, owner: owner}} ->
        send_welcome_email(owner)
        schedule_trial_ending_reminder(tenant)
        {:ok, tenant, owner}
      {:error, step, reason, _} ->
        {:error, {step, reason}}
    end
  end

  defp send_welcome_email(user) do
    MyApp.Workers.WelcomeEmailWorker.new(%{user_id: user.id})
    |> Oban.insert()
  end

  defp schedule_trial_ending_reminder(tenant) do
    reminder_at = DateTime.add(DateTime.utc_now(), 11 * 86400)  # day 11 of 14
    MyApp.Workers.TrialReminderWorker.new(
      %{tenant_id: tenant.id},
      scheduled_at: reminder_at
    )
    |> Oban.insert()
  end
end
```

---

## Step 617: Admin Dashboard

```elixir
defmodule MyAppWeb.AdminLive.Dashboard do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket), do: :timer.send_interval(30_000, :refresh)

    {:ok, assign(socket, stats: get_stats())}
  end

  def handle_info(:refresh, socket) do
    {:noreply, assign(socket, :stats, get_stats())}
  end

  defp get_stats do
    %{
      total_tenants:    MyApp.Tenants.count(),
      active_tenants:   MyApp.Tenants.count_active(),
      mrr:              MyApp.Billing.calculate_mrr(),
      new_this_week:    MyApp.Tenants.count_new(days: 7),
      churn_this_month: MyApp.Billing.churn_count(days: 30),
      plan_distribution: MyApp.Tenants.count_by_plan()
    }
  end

  def render(assigns) do
    ~H"""
    <div class="admin-dashboard">
      <div class="stats-grid">
        <.stat_card title="Total Tenants" value={@stats.total_tenants} />
        <.stat_card title="MRR" value={"$#{@stats.mrr}"} />
        <.stat_card title="New This Week" value={@stats.new_this_week} />
        <.stat_card title="Monthly Churn" value={@stats.churn_this_month} />
      </div>
      
      <h2>Plan Distribution</h2>
      <div :for={{plan, count} <- @stats.plan_distribution}>
        <span><%= plan %>:</span> <span><%= count %></span>
      </div>
    </div>
    """
  end
end
```

---

## Step 618: Tenant-Aware Background Jobs

```elixir
defmodule MyApp.Workers.TenantWorker do
  use Oban.Worker

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"tenant_id" => tenant_id} = args}) do
    tenant = MyApp.Tenants.get!(tenant_id)

    # Skip if tenant is not active
    unless tenant.active do
      {:discard, :tenant_inactive}
    else
      do_work(tenant, args)
    end
  end

  defp do_work(tenant, args) do
    # All operations scoped to tenant
    products = MyApp.Products.list_for_tenant(tenant.id)
    # ... process
    :ok
  end
end

# Enqueue jobs for all tenants
defmodule MyApp.Workers.GlobalScheduler do
  use Oban.Worker

  @impl Oban.Worker
  def perform(_job) do
    MyApp.Tenants.list_active()
    |> Enum.each(fn tenant ->
      MyApp.Workers.MonthlyReportWorker.new(%{tenant_id: tenant.id})
      |> Oban.insert()
    end)
    :ok
  end
end
```

---

## Step 619: SSO with SAML/OAuth

```elixir
defmodule MyApp.SSO do
  # Using Ueberauth for OAuth2 + SAML
  # mix.exs: {:ueberauth, "~> 0.10"}, {:ueberauth_google, "~> 0.10"}

  def handle_callback(%Ueberauth.Auth{} = auth, tenant) do
    unless MyApp.Plans.can?(tenant, :sso) do
      {:error, :sso_not_available}
    else
      email = auth.info.email
      
      case MyApp.Accounts.get_or_create_sso_user(email, tenant.id, auth) do
        {:ok, user} -> {:ok, user}
        {:error, e} -> {:error, e}
      end
    end
  end

  def verify_saml_assertion(xml, tenant) do
    # Verify XML signature with tenant's IdP certificate
    case ExSaml.verify(xml, tenant.sso_certificate) do
      {:ok, attributes} ->
        email = Map.get(attributes, "email")
        {:ok, email}
      {:error, e} ->
        {:error, e}
    end
  end
end

# router.ex
scope "/auth", MyAppWeb do
  pipe_through [:browser, :resolve_tenant]
  
  get  "/:provider",          AuthController, :request
  get  "/:provider/callback", AuthController, :callback
  post "/saml/callback",      SamlController, :callback
end
```

---

## Step 620: SaaS Metrics

```elixir
defmodule MyApp.SaaSMetrics do
  import Ecto.Query

  def monthly_recurring_revenue do
    plan_prices = %{starter: 29, pro: 99, enterprise: 299}

    MyApp.Repo.all(
      from t in MyApp.Tenants.Tenant,
      where: t.active == true and t.plan != :free,
      group_by: t.plan,
      select: {t.plan, count(t.id)}
    )
    |> Enum.reduce(0, fn {plan, count}, acc ->
      acc + (Map.get(plan_prices, plan, 0) * count)
    end)
  end

  def customer_lifetime_value(tenant_id) do
    subscriptions = MyApp.Repo.all(
      from s in "subscription_history",
      where: s.tenant_id == ^tenant_id,
      select: %{amount: s.amount, months: s.months}
    )

    Enum.reduce(subscriptions, 0, fn s, acc -> acc + s.amount * s.months end)
  end

  def churn_rate(days \\ 30) do
    period_start = DateTime.add(DateTime.utc_now(), -days * 86400)

    churned = MyApp.Repo.aggregate(
      from(t in MyApp.Tenants.Tenant, where: t.cancelled_at >= ^period_start),
      :count
    )

    total_start_of_period = MyApp.Repo.aggregate(
      from(t in MyApp.Tenants.Tenant, where: t.inserted_at < ^period_start),
      :count
    )

    if total_start_of_period > 0 do
      churned / total_start_of_period * 100
    else
      0.0
    end
  end

  def net_promoter_score do
    scores = MyApp.Repo.all(from s in "nps_surveys", select: s.score)
    
    promoters  = Enum.count(scores, & &1 >= 9)
    detractors = Enum.count(scores, & &1 <= 6)
    total      = length(scores)
    
    if total > 0, do: (promoters - detractors) / total * 100, else: 0
  end
end
```

---

## สรุป Part 56

✅ **Step 611** - Multi-tenant setup  
✅ **Step 612** - Plan feature flags  
✅ **Step 613** - Stripe billing  
✅ **Step 614** - Usage metering  
✅ **Step 615** - Data isolation  
✅ **Step 616** - Onboarding flow  
✅ **Step 617** - Admin dashboard  
✅ **Step 618** - Tenant background jobs  
✅ **Step 619** - SSO  
✅ **Step 620** - SaaS metrics  

➡️ [Part 57: NIF & Native Integration](./part-57-nif.md)
