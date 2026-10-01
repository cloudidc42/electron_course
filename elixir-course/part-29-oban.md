# Part 29: Background Jobs ด้วย Oban (Steps 311-330)

## Step 311: Oban Overview

```
Oban = Production-grade background job processing
- Backed by PostgreSQL (ACID guarantees)
- No Redis required
- Scheduled jobs (cron)
- Priority queues
- Unique jobs
- Rate limiting per worker
- Dead letter queue
- Web UI (Oban Web)
```

---

## Step 312: Setup

```elixir
# mix.exs
{:oban, "~> 2.17"}

# config/config.exs
config :my_app, Oban,
  repo: MyApp.Repo,
  plugins: [
    {Oban.Plugins.Pruner, max_age: 60 * 60 * 24 * 7},  # 7 days
    {Oban.Plugins.Cron,
     crontab: [
       {"0 * * * *",  MyApp.Workers.HourlyReport},
       {"0 0 * * *",  MyApp.Workers.DailyDigest},
       {"*/5 * * * *", MyApp.Workers.SyncInventory}
     ]}
  ],
  queues: [
    default:  10,
    email:    5,
    media:    3,
    critical: 20
  ]

# application.ex
children = [
  MyApp.Repo,
  {Oban, Application.fetch_env!(:my_app, Oban)}
]

# migration
# mix ecto.gen.migration add_oban_jobs_table
defmodule MyApp.Repo.Migrations.AddObanJobsTable do
  use Ecto.Migration
  def change, do: Oban.Migrations.change()
end
```

---

## Step 313: Basic Worker

```elixir
defmodule MyApp.Workers.EmailWorker do
  use Oban.Worker,
    queue: :email,
    max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"user_id" => user_id, "type" => type}}) do
    user = MyApp.Accounts.get_user!(user_id)
    
    case type do
      "welcome"        -> MyApp.Email.send_welcome(user)
      "password_reset" -> MyApp.Email.send_password_reset(user)
      "digest"         -> MyApp.Email.send_daily_digest(user)
    end
    
    :ok
  end
end

# Enqueue a job
%{user_id: user.id, type: "welcome"}
|> MyApp.Workers.EmailWorker.new()
|> Oban.insert!()

# With options
MyApp.Workers.EmailWorker.new(%{user_id: user.id, type: "digest"},
  schedule_in: 60 * 60,  # 1 hour from now
  priority: 1,
  queue: :critical
)
|> Oban.insert!()
```

---

## Step 314: Scheduled Jobs

```elixir
defmodule MyApp.Workers.DailyDigest do
  use Oban.Worker, queue: :email

  @impl Oban.Worker
  def perform(_job) do
    users = MyApp.Accounts.active_users_with_digest_enabled()
    
    Enum.each(users, fn user ->
      %{user_id: user.id}
      |> MyApp.Workers.SendDigestEmail.new()
      |> Oban.insert!()
    end)
    
    :ok
  end
end

defmodule MyApp.Workers.SyncInventory do
  use Oban.Worker,
    queue: :default,
    max_attempts: 5

  @impl Oban.Worker
  def perform(_job) do
    case MyApp.Inventory.sync_from_supplier() do
      {:ok, count}  ->
        require Logger
        Logger.info("Synced #{count} inventory items")
        :ok
      {:error, reason} ->
        {:error, reason}  # will retry
    end
  end
end
```

---

## Step 315: Unique Jobs

```elixir
defmodule MyApp.Workers.ProcessPayment do
  use Oban.Worker,
    queue: :critical,
    unique: [
      period: 60,
      fields: [:args],
      keys: [:payment_id]
    ]

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"payment_id" => payment_id}}) do
    MyApp.Payments.process(payment_id)
  end
end

# ถ้า insert job เดิมซ้ำภายใน 60 วินาที → จะ skip
%{payment_id: "pay_123"}
|> MyApp.Workers.ProcessPayment.new()
|> Oban.insert()
# => {:ok, %Oban.Job{conflict?: true}} ถ้า unique conflict

# Unique across period:
defmodule MyApp.Workers.SendSMS do
  use Oban.Worker,
    unique: [
      period: :infinity,  # never duplicate
      fields: [:args],
      keys: [:phone, :message_id]
    ]
end
```

---

## Step 316: Error Handling และ Retry

```elixir
defmodule MyApp.Workers.ProcessOrder do
  use Oban.Worker,
    queue: :default,
    max_attempts: 5

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"order_id" => order_id}, attempt: attempt}) do
    require Logger
    
    case MyApp.Orders.process(order_id) do
      {:ok, order} ->
        :ok
      
      {:error, :not_found} ->
        # ไม่ retry ถ้า order ไม่มีอยู่
        {:cancel, "Order #{order_id} not found"}
      
      {:error, :payment_gateway_down} when attempt < 3 ->
        # Retry with backoff
        {:snooze, :math.pow(2, attempt) |> round()}
      
      {:error, reason} ->
        Logger.error("Order processing failed", order_id: order_id, error: reason)
        {:error, reason}  # use default retry
    end
  end
end

# Retry strategies:
# :ok                   → success, mark complete
# {:error, reason}      → failure, retry (up to max_attempts)
# {:cancel, reason}     → failure, no retry, mark discarded
# {:snooze, seconds}    → reschedule N seconds later, same attempt count
# :discard              → silently mark discarded
```

---

## Step 317: Batch Processing

```elixir
defmodule MyApp.Workers.BatchEmailWorker do
  use Oban.Worker, queue: :email

  # Enqueue many jobs at once
  def enqueue_all(user_ids) do
    jobs = Enum.map(user_ids, fn id ->
      MyApp.Workers.EmailWorker.new(%{user_id: id, type: "notification"})
    end)
    
    Oban.insert_all(jobs)
  end
  
  # Or with Ecto.Multi for atomic enqueueing
  def enqueue_with_record(attrs) do
    Ecto.Multi.new()
    |> Ecto.Multi.insert(:order, Order.changeset(%Order{}, attrs))
    |> Oban.insert(:job, fn %{order: order} ->
      MyApp.Workers.ProcessOrder.new(%{order_id: order.id})
    end)
    |> MyApp.Repo.transaction()
  end
end

defmodule MyApp.Workers.BulkNotify do
  use Oban.Worker, queue: :default

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"user_ids" => user_ids, "message" => message}}) do
    user_ids
    |> Enum.chunk_every(100)
    |> Enum.each(fn chunk ->
      Enum.each(chunk, fn user_id ->
        MyApp.Notifications.send(user_id, message)
      end)
    end)
    
    :ok
  end
end
```

---

## Step 318: Rate Limiting

```elixir
defmodule MyApp.Workers.SendSMS do
  use Oban.Worker,
    queue: :sms,
    max_attempts: 3

  # Rate limit: max 10 per second per phone number
  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"phone" => phone, "message" => message}}) do
    case check_rate_limit(phone) do
      :ok ->
        SMSProvider.send(phone, message)
        :ok
      
      {:rate_limited, retry_in} ->
        {:snooze, retry_in}
    end
  end
  
  defp check_rate_limit(phone) do
    key = "sms_rate:#{phone}"
    
    case MyApp.RateLimiter.check(key, limit: 10, window: 1) do
      :ok             -> :ok
      {:deny, reset}  -> {:rate_limited, max(1, reset)}
    end
  end
end

# Oban Pro: built-in rate limiting
defmodule MyApp.Workers.APICall do
  use Oban.Worker

  @impl Oban.Worker
  def perform(%Oban.Job{} = job) do
    # Pro feature: limit globally
    Oban.Pro.Workers.Chunk.handle_chunk(job, &process_batch/1)
  end
end
```

---

## Step 319: Monitoring Jobs

```elixir
defmodule MyApp.ObanReporter do
  use GenServer
  require Logger
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def init(_opts) do
    :ok = :telemetry.attach_many(
      "oban-reporter",
      [
        [:oban, :job, :start],
        [:oban, :job, :stop],
        [:oban, :job, :exception]
      ],
      &handle_event/4,
      nil
    )
    {:ok, %{}}
  end
  
  def handle_event([:oban, :job, :start], _measurements, meta, _) do
    Logger.info("Job started",
      worker: meta.worker,
      id: meta.id,
      queue: meta.queue
    )
  end
  
  def handle_event([:oban, :job, :stop], measurements, meta, _) do
    Logger.info("Job completed",
      worker: meta.worker,
      id: meta.id,
      duration_ms: System.convert_time_unit(measurements.duration, :native, :millisecond)
    )
  end
  
  def handle_event([:oban, :job, :exception], _measurements, meta, _) do
    Logger.error("Job failed",
      worker: meta.worker,
      id: meta.id,
      error: meta.reason,
      attempt: meta.attempt
    )
  end
end

# Query job stats
def get_queue_stats do
  Oban.check_queue(queue: :default)
end

def get_job_counts do
  %{
    available:  Oban.Job |> where(state: "available") |> Repo.aggregate(:count, :id),
    executing:  Oban.Job |> where(state: "executing") |> Repo.aggregate(:count, :id),
    retryable:  Oban.Job |> where(state: "retryable") |> Repo.aggregate(:count, :id),
    discarded:  Oban.Job |> where(state: "discarded") |> Repo.aggregate(:count, :id),
    completed:  Oban.Job |> where(state: "completed") |> Repo.aggregate(:count, :id)
  }
end
```

---

## Step 320: Testing Oban Jobs

```elixir
defmodule MyApp.Workers.EmailWorkerTest do
  use MyApp.DataCase
  use Oban.Testing, repo: MyApp.Repo
  
  alias MyApp.Workers.EmailWorker
  
  test "enqueues welcome email" do
    user = user_fixture()
    
    %{user_id: user.id, type: "welcome"}
    |> EmailWorker.new()
    |> Oban.insert!()
    
    assert_enqueued(worker: EmailWorker, args: %{user_id: user.id, type: "welcome"})
  end
  
  test "performs successfully" do
    user = user_fixture()
    
    assert :ok = perform_job(EmailWorker, %{user_id: user.id, type: "welcome"})
  end
  
  test "returns error for unknown user" do
    assert {:cancel, _reason} = perform_job(EmailWorker, %{user_id: 99999, type: "welcome"})
  end
  
  test "no unexpected jobs enqueued" do
    # ... do stuff ...
    refute_enqueued(worker: EmailWorker)
  end
  
  test "all enqueued" do
    # Process inline in tests
    assert :ok = Oban.drain_queue(queue: :email)
  end
end
```

---

## สรุป Part 29

✅ **Step 311** - Oban overview  
✅ **Step 312** - Setup + configuration  
✅ **Step 313** - Basic workers  
✅ **Step 314** - Scheduled (cron) jobs  
✅ **Step 315** - Unique jobs  
✅ **Step 316** - Error handling + retry  
✅ **Step 317** - Batch processing  
✅ **Step 318** - Rate limiting  
✅ **Step 319** - Monitoring  
✅ **Step 320** - Testing  

➡️ [Part 30: REST API Best Practices](./part-30-rest-api.md)
