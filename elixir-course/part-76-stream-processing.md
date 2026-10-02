# Part 76: Stream Processing (Steps 821-840)

## Step 821: Broadway

```elixir
# mix.exs: {:broadway, "~> 1.1"}

defmodule MyApp.EventProcessor do
  use Broadway

  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name:     __MODULE__,
      producer: [
        module: {BroadwayKafka.Producer, [
          hosts:  [localhost: 9092],
          group_id: "my-app-consumer",
          topics:   ["events"]
        ]},
        concurrency: 1
      ],
      processors: [
        default: [concurrency: 10]
      ],
      batchers: [
        db:      [concurrency: 2, batch_size: 100, batch_timeout: 2000],
        cache:   [concurrency: 3, batch_size: 50,  batch_timeout: 500],
        metrics: [concurrency: 1, batch_size: 200, batch_timeout: 5000]
      ]
    )
  end

  @impl true
  def handle_message(:default, message, _context) do
    event = Jason.decode!(message.data)

    message
    |> Broadway.Message.update_data(fn _ -> normalize(event) end)
    |> route_to_batcher(event["type"])
  end

  @impl true
  def handle_batch(:db, messages, _info, _context) do
    rows = Enum.map(messages, & &1.data)
    MyApp.Repo.insert_all("events", rows, on_conflict: :nothing)
    messages
  end

  @impl true
  def handle_batch(:cache, messages, _info, _context) do
    Enum.each(messages, fn msg ->
      Redix.command(:redix, ["SET", "event:#{msg.data.id}", Jason.encode!(msg.data), "EX", 3600])
    end)
    messages
  end

  @impl true
  def handle_batch(:metrics, messages, _info, _context) do
    counts = Enum.group_by(messages, & &1.data.type)
    Enum.each(counts, fn {type, msgs} ->
      :prometheus_counter.inc(:events_processed_total, [type], length(msgs))
    end)
    messages
  end

  defp route_to_batcher(msg, type) when type in ["user_action", "page_view"] do
    Broadway.Message.put_batcher(msg, :db)
    |> Broadway.Message.put_batcher(:cache)
  end
  defp route_to_batcher(msg, _), do: Broadway.Message.put_batcher(msg, :db)

  defp normalize(event) do
    %{
      id:         event["id"] || Ecto.UUID.generate(),
      type:       event["type"],
      user_id:    event["user_id"],
      payload:    event,
      timestamp:  DateTime.utc_now()
    }
  end
end
```

---

## Step 822: Kafka Integration

```elixir
# mix.exs: {:brod, "~> 3.17"}, {:broadway_kafka, "~> 0.4"}

defmodule MyApp.Kafka.Producer do
  def start_link(_opts) do
    :brod.start_client([{"localhost", 9092}], :my_client)
    :brod.start_producer(:my_client, "events", [])
    {:ok, :started}
  end

  def publish(topic, key, value) do
    message = %{
      key:   key,
      value: Jason.encode!(value)
    }

    :brod.produce_sync(:my_client, topic, :hash, key, message.value)
  end

  def publish_batch(topic, messages) do
    kafka_messages = Enum.map(messages, fn {key, value} ->
      %{key: key, value: Jason.encode!(value)}
    end)

    :brod.produce_sync_offset(:my_client, topic, 0, "batch", kafka_messages)
  end
end

# Consumer with Broadway
defmodule MyApp.Kafka.EventConsumer do
  use Broadway

  def start_link(_) do
    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {BroadwayKafka.Producer, [
          hosts:         [{"localhost", 9092}],
          group_id:      "event-processor",
          topics:        ["events", "user-events"],
          offset_commit_on_ack: true
        ]},
        concurrency: 2
      ],
      processors: [
        default: [concurrency: 5, max_demand: 10]
      ]
    )
  end

  @impl true
  def handle_message(:default, message, _context) do
    event = Jason.decode!(message.data)
    process_event(event)
    message
  end

  @impl true
  def handle_failed([message | _] = messages, _context) do
    # Dead letter queue
    Enum.each(messages, fn msg ->
      MyApp.Kafka.Producer.publish("events-dlq", msg.metadata.key, %{
        original: msg.data,
        error:    "processing_failed",
        at:       DateTime.utc_now()
      })
    end)
    messages
  end
end
```

---

## Step 823: GenStage Pipeline

```elixir
defmodule MyApp.Pipeline.Producer do
  use GenStage

  def start_link(data) do
    GenStage.start_link(__MODULE__, data)
  end

  def init(data) do
    {:producer, data}
  end

  def handle_demand(demand, data) when demand > 0 do
    {events, remaining} = Enum.split(data, demand)
    {:noreply, events, remaining}
  end
end

defmodule MyApp.Pipeline.Transformer do
  use GenStage

  def start_link(_), do: GenStage.start_link(__MODULE__, :ok)

  def init(:ok), do: {:producer_consumer, :ok}

  def handle_events(events, _from, state) do
    transformed = Enum.map(events, &transform/1)
    {:noreply, transformed, state}
  end

  defp transform(event), do: Map.put(event, :processed_at, DateTime.utc_now())
end

defmodule MyApp.Pipeline.Consumer do
  use GenStage

  def start_link(_), do: GenStage.start_link(__MODULE__, :ok)

  def init(:ok), do: {:consumer, :ok}

  def handle_events(events, _from, state) do
    Enum.each(events, fn event ->
      MyApp.Repo.insert!(%MyApp.ProcessedEvent{data: event})
    end)
    {:noreply, [], state}
  end
end

# Connect the pipeline
{:ok, producer}     = MyApp.Pipeline.Producer.start_link(data)
{:ok, transformer}  = MyApp.Pipeline.Transformer.start_link([])
{:ok, consumer}     = MyApp.Pipeline.Consumer.start_link([])

GenStage.sync_subscribe(transformer, to: producer, max_demand: 100)
GenStage.sync_subscribe(consumer, to: transformer, max_demand: 10)
```

---

## Step 824: Flow (Parallel Data Processing)

```elixir
# mix.exs: {:flow, "~> 1.2"}

defmodule MyApp.Analytics.Pipeline do
  alias Experimental.Flow

  def process_events(events) do
    events
    |> Flow.from_enumerable()
    |> Flow.filter(&valid?/1)
    |> Flow.map(&normalize/1)
    |> Flow.partition(key: {:key, :user_id})  # group by user_id
    |> Flow.reduce(fn -> %{} end, fn event, acc ->
      Map.update(acc, event.type, 1, &(&1 + 1))
    end)
    |> Flow.emit(:state)
    |> Enum.to_list()
  end

  def word_count(text_files) do
    text_files
    |> Flow.from_enumerable(max_demand: 4)
    |> Flow.flat_map(fn file ->
      File.stream!(file, :line)
    end)
    |> Flow.flat_map(&String.split/1)
    |> Flow.map(&String.downcase/1)
    |> Flow.partition(key: {:identity})
    |> Flow.reduce(fn -> %{} end, fn word, acc ->
      Map.update(acc, word, 1, &(&1 + 1))
    end)
    |> Flow.emit(:state)
    |> Enum.to_list()
    |> Enum.sort_by(&elem(&1, 1), :desc)
    |> Enum.take(100)
  end
end
```

---

## Step 825: RabbitMQ Integration

```elixir
# mix.exs: {:amqp, "~> 3.3"}

defmodule MyApp.RabbitMQ do
  use GenServer

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def publish(exchange, routing_key, payload, opts \\ []) do
    GenServer.call(__MODULE__, {:publish, exchange, routing_key, payload, opts})
  end

  def init(_) do
    {:ok, conn}    = AMQP.Connection.open(amqp_url())
    {:ok, channel} = AMQP.Channel.open(conn)
    
    # Declare exchanges
    AMQP.Exchange.declare(channel, "events",   :topic,  durable: true)
    AMQP.Exchange.declare(channel, "commands", :direct, durable: true)
    
    # Declare queues
    AMQP.Queue.declare(channel, "email_queue",    durable: true)
    AMQP.Queue.declare(channel, "payment_queue",  durable: true)
    
    # Bind queues
    AMQP.Queue.bind(channel, "email_queue",   "events", routing_key: "user.#")
    AMQP.Queue.bind(channel, "payment_queue", "events", routing_key: "order.#")
    
    # Start consuming
    AMQP.Basic.consume(channel, "email_queue",   nil, no_ack: false)
    
    {:ok, %{conn: conn, channel: channel}}
  end

  def handle_call({:publish, exchange, key, payload, opts}, _from, %{channel: ch} = state) do
    result = AMQP.Basic.publish(ch, exchange, key,
      Jason.encode!(payload),
      persistent:   Keyword.get(opts, :persistent, true),
      content_type: "application/json"
    )
    {:reply, result, state}
  end

  def handle_info({:basic_deliver, payload, meta}, state) do
    event = Jason.decode!(payload)
    
    case process_event(event, meta.routing_key) do
      :ok    -> AMQP.Basic.ack(state.channel, meta.delivery_tag)
      :error -> AMQP.Basic.nack(state.channel, meta.delivery_tag, requeue: true)
    end

    {:noreply, state}
  end

  defp amqp_url, do: System.get_env("RABBITMQ_URL", "amqp://guest:guest@localhost")
end
```

---

## Step 826: Event-Driven Architecture

```elixir
defmodule MyApp.EventBus do
  # Typed event bus using Phoenix.PubSub

  def publish(event) do
    topic  = event_topic(event)
    Phoenix.PubSub.broadcast(MyApp.PubSub, topic, {:event, event})
  end

  def subscribe(event_type) do
    topic = Atom.to_string(event_type)
    Phoenix.PubSub.subscribe(MyApp.PubSub, topic)
  end

  def subscribe_to_all do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "all_events")
  end

  defp event_topic(event) do
    module_name = event.__struct__
      |> Module.split()
      |> List.last()
      |> Macro.underscore()
    
    # Broadcast to specific and wildcard topics
    Phoenix.PubSub.broadcast(MyApp.PubSub, "all_events", {:event, event})
    
    module_name
  end
end

# Event handler using use macro
defmodule MyApp.EventHandler do
  defmacro __using__(_opts) do
    quote do
      use GenServer
      import MyApp.EventBus

      def start_link(_), do: GenServer.start_link(__MODULE__, [])

      def init(_) do
        for event_type <- handled_events() do
          subscribe(event_type)
        end
        {:ok, %{}}
      end

      def handle_info({:event, event}, state) do
        handle_event(event)
        {:noreply, state}
      end
    end
  end
end

# Usage
defmodule MyApp.EmailHandler do
  use MyApp.EventHandler

  def handled_events, do: [MyApp.Events.UserRegistered, MyApp.Events.OrderPlaced]

  def handle_event(%MyApp.Events.UserRegistered{} = event) do
    MyApp.Mailer.send_welcome(event.user_id)
  end

  def handle_event(%MyApp.Events.OrderPlaced{} = event) do
    MyApp.Mailer.send_order_confirmation(event.order_id)
  end
end
```

---

## Step 827: Windowed Aggregation

```elixir
defmodule MyApp.WindowedAggregator do
  use GenServer

  @window_seconds 60

  def start_link(name) do
    GenServer.start_link(__MODULE__, name, name: name)
  end

  def record(server, value) do
    GenServer.cast(server, {:record, value, System.os_time(:second)})
  end

  def aggregate(server, type \\ :sum) do
    GenServer.call(server, {:aggregate, type})
  end

  def init(_name) do
    :timer.send_interval(10_000, :cleanup)
    {:ok, %{data: []}}
  end

  def handle_cast({:record, value, timestamp}, %{data: data} = state) do
    {:noreply, %{state | data: [{timestamp, value} | data]}}
  end

  def handle_call({:aggregate, :sum}, _from, state) do
    values = current_window_values(state.data)
    {:reply, Enum.sum(values), state}
  end

  def handle_call({:aggregate, :avg}, _from, state) do
    values = current_window_values(state.data)
    avg = if Enum.empty?(values), do: 0.0, else: Enum.sum(values) / length(values)
    {:reply, avg, state}
  end

  def handle_call({:aggregate, :count}, _from, state) do
    values = current_window_values(state.data)
    {:reply, length(values), state}
  end

  def handle_info(:cleanup, state) do
    cutoff  = System.os_time(:second) - @window_seconds
    cleaned = Enum.filter(state.data, fn {ts, _} -> ts >= cutoff end)
    {:noreply, %{state | data: cleaned}}
  end

  defp current_window_values(data) do
    cutoff = System.os_time(:second) - @window_seconds
    data
    |> Enum.filter(fn {ts, _} -> ts >= cutoff end)
    |> Enum.map(&elem(&1, 1))
  end
end
```

---

## Step 828: Backpressure Management

```elixir
defmodule MyApp.BackpressureQueue do
  use GenServer

  @max_queue_size 10_000
  @high_watermark 8_000
  @low_watermark  5_000

  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)

  def enqueue(item) do
    GenServer.call(__MODULE__, {:enqueue, item})
  end

  def dequeue(count \\ 1) do
    GenServer.call(__MODULE__, {:dequeue, count})
  end

  def init(_) do
    {:ok, %{queue: :queue.new(), size: 0, paused: false}}
  end

  def handle_call({:enqueue, item}, _from, %{size: size} = state) do
    if size >= @max_queue_size do
      {:reply, {:error, :queue_full}, state}
    else
      new_queue = :queue.in(item, state.queue)
      new_size  = size + 1

      paused = new_size >= @high_watermark
      if paused and not state.paused do
        Phoenix.PubSub.broadcast(MyApp.PubSub, "backpressure", :pause)
      end

      {:reply, :ok, %{state | queue: new_queue, size: new_size, paused: paused}}
    end
  end

  def handle_call({:dequeue, count}, _from, state) do
    {items, new_queue, new_size} = take_from_queue(state.queue, state.size, count)

    if state.paused and new_size <= @low_watermark do
      Phoenix.PubSub.broadcast(MyApp.PubSub, "backpressure", :resume)
    end

    {:reply, items, %{state | queue: new_queue, size: new_size, paused: new_size >= @high_watermark}}
  end

  defp take_from_queue(queue, size, count) do
    Enum.reduce_while(1..count, {[], queue, size}, fn _, {items, q, s} ->
      case :queue.out(q) do
        {{:value, item}, new_q} -> {:cont, {[item | items], new_q, s - 1}}
        {:empty, q}             -> {:halt, {items, q, s}}
      end
    end)
    |> then(fn {items, q, s} -> {Enum.reverse(items), q, s} end)
  end
end
```

---

## Step 829: CDC (Change Data Capture)

```elixir
# mix.exs: {:cainophile, "~> 0.1"}

defmodule MyApp.CDC do
  # Listen to PostgreSQL WAL changes via logical replication

  def start_link(_) do
    Cainophile.Adapters.Postgres.start_link(
      epgsql: %{
        host:     "localhost",
        username: "postgres",
        database: "my_app_prod"
      },
      slot:     "my_app_slot",
      wal_position: :online,
      publications: ["my_publication"]
    )
  end

  def handle_message(%Cainophile.Changes.Transaction{changes: changes}) do
    Enum.each(changes, &process_change/1)
  end

  defp process_change(%Cainophile.Changes.NewRecord{
    relation: {"public", "orders"},
    record:   record
  }) do
    order_id = record["id"]
    MyApp.EventBus.publish(%MyApp.Events.OrderCreated{order_id: order_id})
  end

  defp process_change(%Cainophile.Changes.UpdatedRecord{
    relation: {"public", "orders"},
    record:   record,
    old_record: old_record
  }) do
    if record["status"] != old_record["status"] do
      MyApp.EventBus.publish(%MyApp.Events.OrderStatusChanged{
        order_id:   record["id"],
        old_status: old_record["status"],
        new_status: record["status"]
      })
    end
  end

  defp process_change(_), do: :ok
end
```

---

## Step 830: Dead Letter Queue

```elixir
defmodule MyApp.DeadLetterQueue do
  use GenServer

  @max_retries 3

  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)

  def requeue(message, error, attempt \\ 1) do
    GenServer.cast(__MODULE__, {:requeue, message, error, attempt})
  end

  def list_failed do
    GenServer.call(__MODULE__, :list_failed)
  end

  def retry_all do
    GenServer.cast(__MODULE__, :retry_all)
  end

  def init(_), do: {:ok, %{failed: []}}

  def handle_cast({:requeue, message, error, attempt}, state) do
    if attempt <= @max_retries do
      delay = (:math.pow(2, attempt) * 1000) |> round()
      Process.send_after(self(), {:retry, message, attempt + 1}, delay)
      {:noreply, state}
    else
      failed_msg = %{
        message:    message,
        error:      inspect(error),
        attempts:   attempt,
        failed_at:  DateTime.utc_now()
      }
      Logger.error("Message permanently failed after #{attempt} attempts", message: message)
      {:noreply, %{state | failed: [failed_msg | state.failed]}}
    end
  end

  def handle_info({:retry, message, attempt}, state) do
    Task.start(fn ->
      case MyApp.MessageProcessor.process(message) do
        :ok             -> :ok
        {:error, error} -> requeue(message, error, attempt)
      end
    end)
    {:noreply, state}
  end

  def handle_call(:list_failed, _from, state) do
    {:reply, state.failed, state}
  end
end
```

---

## สรุป Part 76

✅ **Step 821** - Broadway  
✅ **Step 822** - Kafka integration  
✅ **Step 823** - GenStage pipeline  
✅ **Step 824** - Flow parallel processing  
✅ **Step 825** - RabbitMQ integration  
✅ **Step 826** - Event-driven architecture  
✅ **Step 827** - Windowed aggregation  
✅ **Step 828** - Backpressure management  
✅ **Step 829** - CDC (Change Data Capture)  
✅ **Step 830** - Dead letter queue  

➡️ [Part 77: Machine Learning & Data Science](./part-77-ml.md)
