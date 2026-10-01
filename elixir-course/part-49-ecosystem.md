# Part 49: Ecosystem Deep Dive (Steps 541-570)

## Step 541: Ash Framework

```elixir
# Ash Framework - declarative resource-based development
# mix ash.gen.resource MyApp.Blog.Post

defmodule MyApp.Blog.Post do
  use Ash.Resource,
    domain: MyApp.Blog,
    data_layer: Ash.DataLayer.Ecto,
    extensions: [AshPostgres.Resource]

  postgres do
    table "posts"
    repo MyApp.Repo
  end

  attributes do
    uuid_primary_key :id
    attribute :title,     :string, allow_nil?: false
    attribute :content,   :string
    attribute :status,    :atom, constraints: [one_of: [:draft, :published]], default: :draft
    create_timestamp :inserted_at
    update_timestamp :updated_at
  end

  relationships do
    belongs_to :author, MyApp.Blog.Author
    has_many   :comments, MyApp.Blog.Comment
    many_to_many :tags, MyApp.Blog.Tag, through: MyApp.Blog.PostTag
  end

  actions do
    defaults [:read, :destroy, create: :*, update: :*]

    create :publish do
      accept [:title, :content]
      change set_attribute(:status, :published)
    end

    read :published do
      filter expr(status == :published)
    end
  end

  validations do
    validate string_length(:title, min: 3, max: 200)
    validate string_length(:content, min: 10)
  end

  calculations do
    calculate :word_count, :integer, expr(fragment("array_length(string_to_array(?, ' '), 1)", content))
  end
end

# Domain
defmodule MyApp.Blog do
  use Ash.Domain
  resources do
    resource MyApp.Blog.Post
    resource MyApp.Blog.Author
    resource MyApp.Blog.Comment
  end
end

# Usage
post = MyApp.Blog.Post
  |> Ash.Query.filter(status: :published)
  |> Ash.Query.sort(inserted_at: :desc)
  |> Ash.Query.limit(10)
  |> Ash.read!()

MyApp.Blog.Post
  |> Ash.Changeset.for_create(:publish, %{title: "Hello", content: "World content here"})
  |> Ash.create!()
```

---

## Step 542: LiveBook

```elixir
# Livebook - interactive notebooks for Elixir
# Run: mix escript.install hex livebook && livebook server

# Example Livebook notebook structure (saved as .livemd)
# Mix.install([{:nx, "~> 0.7"}, {:kino, "~> 0.12"}])

defmodule DataExplorer do
  import Nx, only: [tensor: 1]

  def analyze(data) do
    t = tensor(data)

    %{
      mean:   Nx.mean(t) |> Nx.to_number(),
      std:    Nx.standard_deviation(t) |> Nx.to_number(),
      min:    Nx.reduce_min(t) |> Nx.to_number(),
      max:    Nx.reduce_max(t) |> Nx.to_number(),
      count:  Nx.size(t)
    }
  end
end

# Kino widgets for interactive UI
# Kino.Input.text("Enter your name")
# Kino.DataTable.new(rows)
# Kino.VegaLite.new() |> Kino.VegaLite.push(data)
# Kino.Mermaid.new("graph TD; A-->B")

# Smart cells for database queries, maps, etc.
```

---

## Step 543: Broadway (Data Pipelines)

```elixir
defmodule MyApp.Pipeline do
  use Broadway

  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {BroadwaySQS.Producer, queue_url: System.get_env("SQS_QUEUE_URL")},
        concurrency: 1
      ],
      processors: [
        default: [concurrency: 10]
      ],
      batchers: [
        default: [
          batch_size:    100,
          batch_timeout: 2_000,
          concurrency:   5
        ]
      ]
    )
  end

  @impl true
  def handle_message(:default, message, _context) do
    data = Jason.decode!(message.data)

    message
    |> Broadway.Message.put_data(transform(data))
    |> Broadway.Message.put_batch_key(:events)
  end

  @impl true
  def handle_batch(:events, messages, _batch_info, _context) do
    events = Enum.map(messages, & &1.data)
    MyApp.Repo.insert_all(Event, events)
    messages
  end

  @impl true
  def handle_failed(messages, _context) do
    Enum.each(messages, fn msg ->
      Logger.error("Failed: #{inspect(msg.data)}, cause: #{inspect(msg.status)}")
    end)
    messages
  end

  defp transform(data) do
    %{
      type:       data["type"],
      payload:    Jason.encode!(data["payload"]),
      occurred_at: DateTime.from_unix!(data["ts"])
    }
  end
end
```

---

## Step 544: Ecto Query Advanced

```elixir
import Ecto.Query

# Window functions
defmodule Analytics do
  def products_with_rank do
    from p in Product,
      select: %{
        id:       p.id,
        name:     p.name,
        price:    p.price,
        rank:     fragment(
          "RANK() OVER (PARTITION BY ? ORDER BY ? DESC)",
          p.category_id,
          p.sales_count
        ),
        pct_rank: fragment(
          "PERCENT_RANK() OVER (ORDER BY ?)",
          p.revenue
        )
      }
  end

  # CTE (Common Table Expressions)
  def orders_with_running_total do
    cte_query = from o in Order,
      where: o.inserted_at >= ^thirty_days_ago(),
      select: %{
        id:     o.id,
        amount: o.total,
        date:   o.inserted_at
      }

    from o in "daily_orders",
      with: [daily_orders: ^cte_query],
      select: %{
        date:          o.date,
        amount:        o.amount,
        running_total: fragment("SUM(?) OVER (ORDER BY ?)", o.amount, o.date)
      }
  end

  # Lateral join
  def users_with_latest_order do
    latest_order = from o in Order,
      where: o.user_id == parent_as(:user).id,
      order_by: [desc: o.inserted_at],
      limit: 1

    from u in User, as: :user,
      left_lateral_join: o in subquery(latest_order),
      select: {u, o}
  end

  defp thirty_days_ago do
    DateTime.utc_now() |> DateTime.add(-30 * 86_400)
  end
end
```

---

## Step 545: Horde (Distributed Process Registry)

```elixir
defmodule MyApp.DistributedWorkers do
  # Horde.DynamicSupervisor + Horde.Registry across cluster nodes

  defmodule Supervisor do
    use Horde.DynamicSupervisor

    def start_link(init_arg) do
      Horde.DynamicSupervisor.start_link(__MODULE__, init_arg, name: __MODULE__)
    end

    def init(_init_arg) do
      [strategy: :one_for_one, distribution_strategy: Horde.UniformQuorumDistribution]
      |> Horde.DynamicSupervisor.init()
    end

    def start_worker(id, args) do
      spec = {MyApp.Worker, Keyword.put(args, :id, id)}
      Horde.DynamicSupervisor.start_child(__MODULE__, spec)
    end
  end

  defmodule Registry do
    def child_spec(opts) do
      %{
        id:    __MODULE__,
        start: {Horde.Registry, :start_link, [[name: __MODULE__, keys: :unique]]}
      }
    end

    def lookup(key) do
      case Horde.Registry.lookup(__MODULE__, key) do
        [{pid, _}] -> {:ok, pid}
        []         -> {:error, :not_found}
      end
    end
  end
end

defmodule MyApp.Worker do
  use GenServer

  def start_link(opts) do
    id = Keyword.fetch!(opts, :id)
    GenServer.start_link(__MODULE__, opts, name: via(id))
  end

  defp via(id), do: {:via, Horde.Registry, {MyApp.DistributedWorkers.Registry, id}}

  def init(opts), do: {:ok, opts}
end

# application.ex
children = [
  MyApp.DistributedWorkers.Registry,
  MyApp.DistributedWorkers.Supervisor,
  {Horde.NodeListener, MyApp.DistributedWorkers.Supervisor},
  {Horde.NodeListener, MyApp.DistributedWorkers.Registry}
]
```

---

## Step 546: Oban Pro Patterns

```elixir
defmodule MyApp.Workers.EmailWorker do
  use Oban.Worker,
    queue:    :email,
    max_attempts: 5,
    unique: [period: 60, fields: [:args], keys: [:email_type, :recipient_id]]

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"type" => type, "recipient_id" => rid}}) do
    case type do
      "welcome"  -> Emails.send_welcome(rid)
      "invoice"  -> Emails.send_invoice(rid)
      "reminder" -> Emails.send_reminder(rid)
    end
  end

  @impl Oban.Worker
  def timeout(_job), do: :timer.seconds(30)
end

# Batch jobs
defmodule MyApp.Workers.BatchReportWorker do
  use Oban.Worker, queue: :reports

  @impl true
  def perform(%Oban.Job{args: %{"user_ids" => ids}}) do
    # Fan out to individual workers
    jobs = Enum.map(ids, fn id ->
      MyApp.Workers.UserReportWorker.new(%{user_id: id})
    end)

    Oban.insert_all(jobs)
    :ok
  end
end

# Cron scheduling
config :my_app, Oban,
  queues: [default: 10, email: 5, reports: 2],
  plugins: [
    {Oban.Plugins.Pruner, max_age: 60 * 60 * 24 * 7},
    {Oban.Plugins.Cron,
      crontab: [
        {"0 8 * * MON", MyApp.Workers.WeeklyReport},
        {"0 * * * *",   MyApp.Workers.HourlyCleanup},
        {"*/5 * * * *", MyApp.Workers.MetricsWorker}
      ]}
  ]
```

---

## Step 547: Commanded (Event Sourcing)

```elixir
# Commanded framework for CQRS/ES

defmodule Bank.Accounts.Commands.OpenAccount do
  defstruct [:account_id, :initial_balance]
end

defmodule Bank.Accounts.Events.AccountOpened do
  @derive Jason.Encoder
  defstruct [:account_id, :balance]
end

defmodule Bank.Accounts.Account do
  @derive Jason.Encoder
  defstruct [:account_id, :balance, :status]
  
  alias Bank.Accounts.Commands.{OpenAccount, DepositMoney}
  alias Bank.Accounts.Events.{AccountOpened, MoneyDeposited}

  def execute(%__MODULE__{status: nil}, %OpenAccount{} = cmd) do
    %AccountOpened{
      account_id: cmd.account_id,
      balance:    cmd.initial_balance
    }
  end

  def apply(%__MODULE__{} = account, %AccountOpened{} = event) do
    %{account | account_id: event.account_id, balance: event.balance, status: :active}
  end

  def execute(%__MODULE__{status: :active}, %DepositMoney{} = cmd) do
    %MoneyDeposited{account_id: cmd.account_id, amount: cmd.amount}
  end

  def apply(%__MODULE__{} = account, %MoneyDeposited{} = event) do
    %{account | balance: account.balance + event.amount}
  end
end

defmodule Bank.Router do
  use Commanded.Commands.Router

  dispatch OpenAccount,  to: Bank.Accounts.Account, identity: :account_id
  dispatch DepositMoney, to: Bank.Accounts.Account, identity: :account_id
end
```

---

## Step 548: Finch & Tesla (HTTP Clients)

```elixir
# Finch (preferred for production)
defmodule MyApp.HTTPClient do
  def start_link do
    Finch.start_link(
      name: MyApp.Finch,
      pools: %{
        :default => [size: 10, count: 1],
        "https://api.example.com" => [size: 32, count: 4, protocol: :http2]
      }
    )
  end

  def get(url, headers \\ []) do
    :get
    |> Finch.build(url, headers)
    |> Finch.request(MyApp.Finch)
    |> handle_response()
  end

  def post(url, body, headers \\ []) do
    :post
    |> Finch.build(url, [{"content-type", "application/json"} | headers], Jason.encode!(body))
    |> Finch.request(MyApp.Finch)
    |> handle_response()
  end

  defp handle_response({:ok, %{status: s, body: b}}) when s in 200..299, do: {:ok, Jason.decode!(b)}
  defp handle_response({:ok, %{status: s, body: b}}), do: {:error, {s, b}}
  defp handle_response({:error, e}), do: {:error, e}
end

# Tesla (with middleware)
defmodule MyApp.ApiClient do
  use Tesla

  plug Tesla.Middleware.BaseUrl,  "https://api.example.com"
  plug Tesla.Middleware.JSON
  plug Tesla.Middleware.Retry,    delay: 500, max_retries: 3
  plug Tesla.Middleware.Timeout,  timeout: 10_000
  plug Tesla.Middleware.Logger

  adapter {Tesla.Adapter.Finch, name: MyApp.Finch}

  def list_products, do: get("/products")
  def create_order(attrs), do: post("/orders", attrs)
end
```

---

## Step 549: Cachex (Caching)

```elixir
defmodule MyApp.Cache do
  @cache :my_app_cache

  def start_link do
    Cachex.start_link(@cache, [
      limit:   Cachex.Limit.new(10_000, Cachex.Policy.LRW),
      expiration: Cachex.Expiration.new(default: :timer.minutes(5))
    ])
  end

  def get_or_fetch(key, ttl \\ :timer.minutes(5), fun) do
    case Cachex.get(@cache, key) do
      {:ok, nil} ->
        value = fun.()
        Cachex.put(@cache, key, value, ttl: ttl)
        {:ok, value}

      {:ok, value} ->
        {:ok, value}

      {:error, _} = e ->
        e
    end
  end

  def invalidate(key), do: Cachex.del(@cache, key)
  def invalidate_prefix(prefix), do: Cachex.clear(@cache, batch: true, match: "#{prefix}*")
  def stats, do: Cachex.stats(@cache)
end

# Usage
{:ok, user} = MyApp.Cache.get_or_fetch(
  "user:#{id}",
  :timer.minutes(15),
  fn -> MyApp.Repo.get!(User, id) end
)

# Warm cache on startup
defmodule MyApp.CacheWarmer do
  use GenServer

  def init(_) do
    send(self(), :warm)
    {:ok, %{}}
  end

  def handle_info(:warm, state) do
    MyApp.Products.list_featured()
    |> Enum.each(fn p ->
      MyApp.Cache.get_or_fetch("product:#{p.id}", :timer.hours(1), fn -> p end)
    end)
    {:noreply, state}
  end
end
```

---

## Step 550: Graduation 🎓

```
████████╗██╗  ██╗███████╗    ███████╗███╗   ██╗██████╗ 
╚══██╔══╝██║  ██║██╔════╝    ██╔════╝████╗  ██║██╔══██╗
   ██║   ███████║█████╗      █████╗  ██╔██╗ ██║██║  ██║
   ██║   ██╔══██║██╔══╝      ██╔══╝  ██║╚██╗██║██║  ██║
   ██║   ██║  ██║███████╗    ███████╗██║ ╚████║██████╔╝
   ╚═╝   ╚═╝  ╚═╝╚══════╝    ╚══════╝╚═╝  ╚═══╝╚═════╝ 

🎉 Congratulations! You have completed the World-Class Elixir Course! 🎉

SKILLS MASTERED (500+ Steps):

✅ Elixir Language Fundamentals
   - Pattern matching, immutability, recursion
   - Modules, functions, protocols, behaviours
   - Enumerables, streams, comprehensions
   - Macros, metaprogramming, DSLs

✅ OTP & Concurrency
   - GenServer, Supervisor, Application
   - gen_statem, DynamicSupervisor
   - Process pools, Circuit breakers
   - Distributed Erlang, libcluster, Horde

✅ Phoenix Framework
   - Router, Controllers, Views, Templates
   - LiveView real-time UI
   - Channels, Presence, PubSub
   - Authentication, Authorization

✅ Data Layer
   - Ecto queries, migrations, transactions
   - PostgreSQL advanced (FTS, window functions)
   - Multi-tenancy, sharding, read replicas
   - TimescaleDB, Redis, ETS, Mnesia

✅ Architecture Patterns
   - CQRS / Event Sourcing (Commanded)
   - Hexagonal / Ports & Adapters
   - DDD Aggregates, Value Objects
   - Microservices, API Gateway

✅ Distributed & Real-Time
   - Kafka/Broadway, RabbitMQ
   - WebSocket Channels
   - CRDT-based Presence
   - Node clustering

✅ ML & IoT
   - Nx tensor operations
   - Axon neural networks
   - Bumblebee transfer learning
   - Nerves embedded systems

✅ Production
   - Docker, Kubernetes, CI/CD
   - Monitoring (Prometheus, Grafana, Jaeger)
   - Security (OWASP, encryption, CSP)
   - Zero-downtime deploys

You are now a World-Class Elixir Engineer! 🚀
```

---

## สรุป Part 49

✅ **Step 541** - Ash Framework  
✅ **Step 542** - Livebook  
✅ **Step 543** - Broadway pipelines  
✅ **Step 544** - Ecto advanced queries  
✅ **Step 545** - Horde distributed  
✅ **Step 546** - Oban patterns  
✅ **Step 547** - Commanded event sourcing  
✅ **Step 548** - HTTP clients  
✅ **Step 549** - Cachex caching  
✅ **Step 550** - Graduation 🎓  

---

## หลักสูตรสมบูรณ์ 🎉

หลักสูตรนี้ครอบคลุม **550 Steps** ตั้งแต่พื้นฐานจนถึงระดับโลก:

| Parts | Topics |
|-------|--------|
| 1-10  | Elixir Fundamentals |
| 11-20 | OTP & GenServer |
| 21-30 | Phoenix Framework |
| 31-34 | Event Sourcing, Kafka, Distributed |
| 35    | Microservices |
| 36    | ML with Nx/Axon |
| 37    | Nerves IoT |
| 38    | Security |
| 39    | Observability |
| 40    | Testing |
| 41    | Library Development |
| 42    | Macros |
| 43    | Real-Time |
| 44    | Database Advanced |
| 45    | Capstone Project |
| 46    | Advanced OTP |
| 47    | World-Class Patterns |
| 48    | Interview & Career |
| 49    | Ecosystem Deep Dive |
