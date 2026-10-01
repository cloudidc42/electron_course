# Part 34: Kafka และ Message Queues (Steps 361-380)

## Step 361: Kafka Basics

```
Apache Kafka = Distributed event streaming platform
- Topics: คล้าย database table, เก็บ events
- Partitions: parallel processing
- Producers: เขียน events
- Consumers: อ่าน events (ไม่ลบ, track offset เอง)
- Consumer Groups: load balance partitions
- Retention: เก็บ events ตาม policy (ไม่ใช่แค่ queue)

Use Cases:
- Event streaming between microservices
- Activity tracking
- Log aggregation
- Real-time analytics
```

---

## Step 362: Brod (Kafka Client)

```elixir
# mix.exs
{:brod, "~> 3.18"}
# หรือ
{:kafka_ex, "~> 0.13"}

# config/config.exs
config :brod,
  clients: [
    kafka_client: [
      endpoints: [{"localhost", 9092}],
      auto_start_producers: true,
      default_producer_config: [
        required_acks: -1,
        ack_timeout: 10_000
      ]
    ]
  ]

# Producer
defmodule MyApp.KafkaProducer do
  @client :kafka_client
  
  def publish(topic, key, message) when is_map(message) do
    payload = Jason.encode!(message)
    publish(topic, key, payload)
  end
  
  def publish(topic, key, payload) when is_binary(payload) do
    :brod.produce_sync(
      @client,
      topic,
      _partition = :hash,
      to_string(key),
      payload
    )
  end
  
  def publish_batch(topic, messages) do
    msgs = Enum.map(messages, fn {key, value} ->
      {to_string(key), Jason.encode!(value)}
    end)
    
    :brod.produce_sync_offset(
      @client,
      topic,
      :hash,
      "",
      msgs
    )
  end
end
```

---

## Step 363: Consumer

```elixir
defmodule MyApp.KafkaConsumer do
  @behaviour :brod_group_subscriber_v2
  
  def child_spec(opts) do
    %{
      id: __MODULE__,
      start: {__MODULE__, :start_link, [opts]},
      restart: :permanent
    }
  end
  
  def start_link(_opts) do
    group_id = "my-app-consumer-group"
    topics   = ["orders", "payments", "inventory"]
    
    :brod.start_link_group_subscriber_v2(
      _client     = :kafka_client,
      group_id,
      topics,
      _group_config = [offset_commit_policy: :commit_to_kafka_v2],
      _consumer_config = [begin_offset: :earliest],
      __MODULE__,
      _init_data = []
    )
  end
  
  @impl :brod_group_subscriber_v2
  def init(_group_id, _init_data) do
    {:ok, %{}}
  end
  
  @impl :brod_group_subscriber_v2
  def handle_message(topic, partition, message, state) do
    %{offset: offset, value: payload} = message
    
    case Jason.decode(payload) do
      {:ok, event} ->
        process_event(topic, event)
        {:ok, ack, state}
      
      {:error, reason} ->
        require Logger
        Logger.error("Failed to decode message: #{inspect(reason)}")
        {:ok, ack, state}
    end
  end
  
  defp process_event("orders", %{"type" => "OrderPlaced"} = event) do
    MyApp.OrderHandler.handle(event)
  end
  
  defp process_event("payments", %{"type" => "PaymentCompleted"} = event) do
    MyApp.PaymentHandler.handle(event)
  end
  
  defp process_event(topic, event) do
    require Logger
    Logger.debug("Unhandled event on #{topic}: #{inspect(event)}")
  end
end
```

---

## Step 364: Broadway (Stream Processing)

```elixir
# mix.exs
{:broadway, "~> 1.0"},
{:broadway_kafka, "~> 0.4"}

defmodule MyApp.OrderPipeline do
  use Broadway
  
  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {
          BroadwayKafka.Producer,
          hosts: [{"localhost", 9092}],
          group_id: "order-processor",
          topics: ["orders"],
          client_config: [sasl: {:plain, "user", "pass"}]
        },
        concurrency: 1
      ],
      processors: [
        default: [concurrency: 5]  # 5 concurrent processors
      ],
      batchers: [
        database: [
          batch_size: 100,
          batch_timeout: 2_000,
          concurrency: 2
        ]
      ]
    )
  end
  
  @impl Broadway
  def handle_message(:default, message, _context) do
    event = Jason.decode!(message.data)
    
    message
    |> Broadway.Message.put_data(event)
    |> Broadway.Message.put_batcher(:database)
  end
  
  @impl Broadway
  def handle_batch(:database, messages, _batch_info, _context) do
    events = Enum.map(messages, & &1.data)
    
    case MyApp.Events.insert_all(events) do
      {:ok, _}    -> messages
      {:error, _} ->
        Enum.map(messages, &Broadway.Message.failed(&1, "DB insert failed"))
    end
  end
  
  @impl Broadway
  def handle_failed(messages, _context) do
    Enum.each(messages, fn msg ->
      require Logger
      Logger.error("Failed message: #{inspect(msg.data)}")
    end)
    messages
  end
end
```

---

## Step 365: RabbitMQ ด้วย AMQP

```elixir
# mix.exs
{:amqp, "~> 3.3"}

defmodule MyApp.RabbitMQ do
  use GenServer
  
  @exchange "my_app"
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def init(_) do
    {:ok, conn}  = AMQP.Connection.open("amqp://guest:guest@localhost")
    {:ok, chan}  = AMQP.Channel.open(conn)
    
    AMQP.Exchange.declare(chan, @exchange, :topic, durable: true)
    
    {:ok, %{conn: conn, chan: chan}}
  end
  
  def publish(routing_key, payload) do
    GenServer.call(__MODULE__, {:publish, routing_key, payload})
  end
  
  def handle_call({:publish, routing_key, payload}, _from, %{chan: chan} = state) do
    msg = Jason.encode!(payload)
    
    result = AMQP.Basic.publish(
      chan,
      @exchange,
      routing_key,
      msg,
      persistent: true,
      content_type: "application/json"
    )
    
    {:reply, result, state}
  end
end

defmodule MyApp.RabbitMQConsumer do
  use GenServer
  
  @exchange "my_app"
  @queue "order_events"
  @routing_key "orders.*"
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def init(_) do
    {:ok, conn} = AMQP.Connection.open("amqp://guest:guest@localhost")
    {:ok, chan} = AMQP.Channel.open(conn)
    
    AMQP.Queue.declare(chan, @queue, durable: true)
    AMQP.Queue.bind(chan, @queue, @exchange, routing_key: @routing_key)
    AMQP.Basic.qos(chan, prefetch_count: 10)
    {:ok, _tag} = AMQP.Basic.consume(chan, @queue)
    
    {:ok, %{chan: chan}}
  end
  
  def handle_info({:basic_deliver, payload, meta}, %{chan: chan} = state) do
    spawn(fn ->
      {:ok, event} = Jason.decode(payload)
      
      case process_event(event) do
        :ok -> AMQP.Basic.ack(chan, meta.delivery_tag)
        _   -> AMQP.Basic.nack(chan, meta.delivery_tag, requeue: true)
      end
    end)
    
    {:noreply, state}
  end
  
  defp process_event(event) do
    MyApp.EventHandler.handle(event)
    :ok
  rescue
    _ -> :error
  end
end
```

---

## Step 366: Redis Streams

```elixir
# mix.exs
{:redix, "~> 1.4"}

defmodule MyApp.RedisStream do
  @stream "events"
  
  def publish(type, data) do
    fields = [
      "type", type,
      "data", Jason.encode!(data),
      "timestamp", DateTime.utc_now() |> DateTime.to_iso8601()
    ]
    
    Redix.command(:redis, ["XADD", @stream, "*" | fields])
  end
  
  def read_from(last_id \\ "0", count \\ 100) do
    {:ok, [[_, entries]]} = Redix.command(:redis, [
      "XREAD", "COUNT", count, "STREAMS", @stream, last_id
    ])
    
    Enum.map(entries, fn [id, fields] ->
      data = fields |> Enum.chunk_every(2) |> Map.new(fn [k, v] -> {k, v} end)
      {id, data}
    end)
  end
  
  def consume_group(group, consumer, count \\ 10) do
    case Redix.command(:redis, [
      "XREADGROUP", "GROUP", group, consumer,
      "COUNT", count, "STREAMS", @stream, ">"
    ]) do
      {:ok, nil} -> []
      {:ok, [[_, entries]]} ->
        Enum.map(entries, fn [id, fields] ->
          data = fields |> Enum.chunk_every(2) |> Map.new(fn [k, v] -> {k, v} end)
          {id, data}
        end)
    end
  end
  
  def ack(group, message_id) do
    Redix.command(:redis, ["XACK", @stream, group, message_id])
  end
  
  def create_group(group) do
    Redix.command(:redis, ["XGROUP", "CREATE", @stream, group, "$", "MKSTREAM"])
  end
end
```

---

## Step 367: Dead Letter Queue

```elixir
defmodule MyApp.DeadLetterQueue do
  @dlq_key "dlq:failed_messages"
  
  def add_failed(message, reason) do
    entry = Jason.encode!(%{
      message:    message,
      reason:     to_string(reason),
      failed_at:  DateTime.utc_now() |> DateTime.to_iso8601(),
      attempts:   Map.get(message, :attempts, 0) + 1
    })
    
    Redix.pipeline(:redis, [
      ["LPUSH", @dlq_key, entry],
      ["LTRIM", @dlq_key, 0, 9999]  # keep last 10000
    ])
  end
  
  def replay_failed(limit \\ 10) do
    {:ok, entries} = Redix.command(:redis, ["LRANGE", @dlq_key, 0, limit - 1])
    
    Enum.map(entries, fn entry ->
      %{"message" => msg} = Jason.decode!(entry)
      MyApp.MessageRouter.route(msg)
    end)
  end
  
  def count do
    {:ok, count} = Redix.command(:redis, ["LLEN", @dlq_key])
    count
  end
end
```

---

## Step 368: Message Schema และ Validation

```elixir
defmodule MyApp.Events.Schema do
  @schemas %{
    "OrderPlaced"   => [:order_id, :user_id, :items, :total],
    "PaymentMade"   => [:payment_id, :order_id, :amount, :method],
    "UserCreated"   => [:user_id, :email, :name]
  }
  
  def validate(type, data) when is_map(data) do
    case Map.get(@schemas, type) do
      nil ->
        {:error, "Unknown event type: #{type}"}
      
      required_fields ->
        missing = Enum.filter(required_fields, &(not Map.has_key?(data, to_string(&1))))
        
        if missing == [] do
          {:ok, data}
        else
          {:error, "Missing fields: #{inspect(missing)}"}
        end
    end
  end
  
  def validate(type, data) do
    {:error, "Invalid data format for #{type}: #{inspect(data)}"}
  end
end

# Wrap in typed struct
defmodule MyApp.Events.OrderPlaced do
  use TypedStruct
  
  typedstruct enforce: true do
    field :order_id,  :string
    field :user_id,   :integer
    field :items,     list()
    field :total,     :decimal
    field :placed_at, DateTime.t()
    field :metadata,  map(), default: %{}
  end
  
  def from_map(data) do
    {:ok, struct!(__MODULE__, %{
      order_id:  data["order_id"],
      user_id:   data["user_id"],
      items:     data["items"],
      total:     Decimal.new(data["total"]),
      placed_at: DateTime.from_iso8601!(data["placed_at"])
    })}
  rescue
    e -> {:error, Exception.message(e)}
  end
end
```

---

## Step 369: Circuit Breaker

```elixir
defmodule MyApp.CircuitBreaker do
  use GenServer
  
  @failure_threshold 5
  @reset_timeout 30_000  # 30s
  
  defstruct [:name, :state, :failure_count, :last_failure_at]
  
  def start_link(name) do
    GenServer.start_link(__MODULE__, name, name: via(name))
  end
  
  def call(name, fun) do
    GenServer.call(via(name), {:call, fun})
  end
  
  def init(name) do
    {:ok, %__MODULE__{name: name, state: :closed, failure_count: 0}}
  end
  
  def handle_call({:call, _fun}, _from, %{state: :open, last_failure_at: t} = state) do
    if :timer.now_diff(:os.timestamp(), t) > @reset_timeout * 1000 do
      {:reply, {:error, :circuit_open}, %{state | state: :half_open}}
    else
      {:reply, {:error, :circuit_open}, state}
    end
  end
  
  def handle_call({:call, fun}, _from, state) do
    try do
      result = fun.()
      {:reply, {:ok, result}, %{state | state: :closed, failure_count: 0}}
    rescue
      e ->
        new_state = record_failure(state)
        {:reply, {:error, e}, new_state}
    end
  end
  
  defp record_failure(%{failure_count: count} = state) when count + 1 >= @failure_threshold do
    %{state | state: :open, failure_count: count + 1, last_failure_at: :os.timestamp()}
  end
  defp record_failure(%{failure_count: count} = state) do
    %{state | failure_count: count + 1}
  end
  
  defp via(name), do: {:via, Registry, {MyApp.Registry, {:circuit_breaker, name}}}
end

# Usage
MyApp.CircuitBreaker.call(:payment_service, fn ->
  ExternalPayment.charge(order.total)
end)
```

---

## Step 370: Monitoring Message Queues

```elixir
defmodule MyApp.QueueMonitor do
  use GenServer
  
  @check_interval 60_000  # 1 minute
  @lag_warning_threshold 1000
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def init(_) do
    schedule_check()
    {:ok, %{}}
  end
  
  def handle_info(:check_queues, state) do
    check_kafka_lag()
    check_oban_queue()
    
    schedule_check()
    {:noreply, state}
  end
  
  defp check_kafka_lag do
    # Get consumer group lag
    # If lag > threshold, alert
    require Logger
    Logger.info("Kafka consumer lag check")
  end
  
  defp check_oban_queue do
    import Ecto.Query
    
    waiting = MyApp.Repo.aggregate(
      from(j in Oban.Job, where: j.state == "available"),
      :count,
      :id
    )
    
    if waiting > @lag_warning_threshold do
      :telemetry.execute([:my_app, :queue, :lag_high], %{count: waiting}, %{})
      require Logger
      Logger.warning("High job queue depth: #{waiting}")
    end
  end
  
  defp schedule_check do
    Process.send_after(self(), :check_queues, @check_interval)
  end
end
```

---

## สรุป Part 34

✅ **Step 361** - Kafka concepts  
✅ **Step 362** - Brod producer/consumer  
✅ **Step 363** - Consumer groups  
✅ **Step 364** - Broadway stream processing  
✅ **Step 365** - RabbitMQ with AMQP  
✅ **Step 366** - Redis Streams  
✅ **Step 367** - Dead letter queue  
✅ **Step 368** - Message schema validation  
✅ **Step 369** - Circuit breaker  
✅ **Step 370** - Queue monitoring  

➡️ [Part 35: Microservices](./part-35-microservices.md)
