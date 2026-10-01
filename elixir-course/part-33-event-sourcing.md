# Part 33: Event Sourcing และ CQRS (Steps 351-370)

## Step 351: Event Sourcing Concepts

```
Traditional: Store current state only
Event Sourcing: Store all events that led to current state

Benefits:
- Complete audit trail
- Time-travel debugging
- Event replay
- Read models rebuilt from events
- Decoupled write/read (CQRS)

Key Concepts:
- Event: something that happened (immutable fact)
- Aggregate: entity with state rebuilt from events
- Event Store: append-only log of events
- Projection: read model built from events
- Command: intent to do something
- CQRS: separate command (write) and query (read) sides
```

---

## Step 352: Event Store

```elixir
defmodule MyApp.EventStore do
  import Ecto.Query
  alias MyApp.Repo
  
  defmodule Event do
    use Ecto.Schema
    import Ecto.Changeset
    
    schema "events" do
      field :stream_id,      :string
      field :stream_version, :integer
      field :event_type,     :string
      field :data,           :map
      field :metadata,       :map, default: %{}
      field :causation_id,   :binary_id
      field :correlation_id, :binary_id
      timestamps()
    end
    
    def changeset(event, attrs) do
      event
      |> cast(attrs, [:stream_id, :stream_version, :event_type, :data, :metadata])
      |> validate_required([:stream_id, :stream_version, :event_type, :data])
      |> unique_constraint([:stream_id, :stream_version])
    end
  end
  
  def append(stream_id, events, expected_version \\ nil) do
    Repo.transaction(fn ->
      current_version = get_current_version(stream_id)
      
      if expected_version && current_version != expected_version do
        Repo.rollback({:wrong_expected_version, current_version})
      end
      
      events
      |> Enum.with_index(current_version + 1)
      |> Enum.map(fn {event, version} ->
        Repo.insert!(%Event{
          stream_id:      stream_id,
          stream_version: version,
          event_type:     event_type(event),
          data:           serialize(event),
          metadata:       event.metadata || %{}
        })
      end)
    end)
  end
  
  def read(stream_id, from_version \\ 0) do
    from(e in Event,
      where: e.stream_id == ^stream_id and e.stream_version >= ^from_version,
      order_by: e.stream_version
    )
    |> Repo.all()
    |> Enum.map(&deserialize/1)
  end
  
  defp get_current_version(stream_id) do
    from(e in Event,
      where: e.stream_id == ^stream_id,
      select: max(e.stream_version)
    )
    |> Repo.one() || -1
  end
  
  defp event_type(event), do: event.__struct__ |> to_string() |> String.replace("Elixir.", "")
  defp serialize(event), do: Map.from_struct(event) |> Map.delete(:metadata)
  
  defp deserialize(%Event{event_type: type, data: data}) do
    module = Module.concat([type])
    struct(module, atomize_keys(data))
  end
  
  defp atomize_keys(map) do
    Map.new(map, fn {k, v} -> {String.to_atom(k), v} end)
  end
end
```

---

## Step 353: Aggregate

```elixir
defmodule MyApp.Order do
  defstruct [:id, :user_id, :items, :total, :status, :version]
  
  # Events
  defmodule OrderPlaced do
    defstruct [:order_id, :user_id, :items, :total, :metadata]
  end
  
  defmodule OrderConfirmed do
    defstruct [:order_id, :confirmed_at, :metadata]
  end
  
  defmodule OrderCancelled do
    defstruct [:order_id, :reason, :cancelled_at, :metadata]
  end
  
  defmodule OrderShipped do
    defstruct [:order_id, :tracking_number, :shipped_at, :metadata]
  end
  
  # Commands
  def place_order(nil, command) do
    order_id = UUID.uuid4()
    
    event = %OrderPlaced{
      order_id: order_id,
      user_id:  command.user_id,
      items:    command.items,
      total:    calculate_total(command.items)
    }
    
    {:ok, [event]}
  end
  
  def confirm_order(%{status: :pending} = order, _command) do
    event = %OrderConfirmed{
      order_id:     order.id,
      confirmed_at: DateTime.utc_now()
    }
    {:ok, [event]}
  end
  def confirm_order(%{status: status}, _), do: {:error, "Cannot confirm order in #{status} status"}
  
  def cancel_order(%{status: status} = order, command) when status in [:pending, :confirmed] do
    event = %OrderCancelled{
      order_id:     order.id,
      reason:       command.reason,
      cancelled_at: DateTime.utc_now()
    }
    {:ok, [event]}
  end
  def cancel_order(_, _), do: {:error, "Cannot cancel order"}
  
  # Apply events to rebuild state
  def apply(nil, %OrderPlaced{} = e) do
    %__MODULE__{
      id:      e.order_id,
      user_id: e.user_id,
      items:   e.items,
      total:   e.total,
      status:  :pending
    }
  end
  
  def apply(order, %OrderConfirmed{}) do
    %{order | status: :confirmed}
  end
  
  def apply(order, %OrderCancelled{}) do
    %{order | status: :cancelled}
  end
  
  def apply(order, %OrderShipped{tracking_number: tracking}) do
    %{order | status: :shipped}
  end
  
  # Rebuild state from events
  def rebuild(events) do
    Enum.reduce(events, nil, &apply(&2, &1))
  end
  
  defp calculate_total(items) do
    Enum.reduce(items, 0, fn %{price: p, qty: q}, acc -> acc + p * q end)
  end
end
```

---

## Step 354: Command Handler

```elixir
defmodule MyApp.CommandHandler do
  alias MyApp.{Order, EventStore}
  
  def handle(%{type: :place_order} = cmd) do
    stream_id = "order-#{cmd.order_id || UUID.uuid4()}"
    
    with {:ok, events} <- Order.place_order(nil, cmd) do
      EventStore.append(stream_id, events)
    end
  end
  
  def handle(%{type: :confirm_order, order_id: order_id} = cmd) do
    stream_id = "order-#{order_id}"
    
    with {:ok, current, version} <- load_aggregate(stream_id),
         {:ok, events}           <- Order.confirm_order(current, cmd) do
      EventStore.append(stream_id, events, version)
    end
  end
  
  def handle(%{type: :cancel_order, order_id: order_id} = cmd) do
    stream_id = "order-#{order_id}"
    
    with {:ok, current, version} <- load_aggregate(stream_id),
         {:ok, events}           <- Order.cancel_order(current, cmd) do
      EventStore.append(stream_id, events, version)
    end
  end
  
  defp load_aggregate(stream_id) do
    events = EventStore.read(stream_id)
    
    if events == [] do
      {:error, :not_found}
    else
      aggregate = Order.rebuild(events)
      version = length(events) - 1
      {:ok, aggregate, version}
    end
  end
end
```

---

## Step 355: Projections (Read Models)

```elixir
defmodule MyApp.Projections.OrderSummary do
  use Ecto.Schema
  
  schema "order_summaries" do
    field :order_id,  :string
    field :user_id,   :integer
    field :status,    :string
    field :total,     :decimal
    field :placed_at, :utc_datetime
    field :items,     :map
    timestamps()
  end
end

defmodule MyApp.Projector do
  alias MyApp.{Repo, Projections.OrderSummary}
  alias MyApp.Order.{OrderPlaced, OrderConfirmed, OrderCancelled, OrderShipped}
  
  def project(%OrderPlaced{} = event) do
    %OrderSummary{}
    |> Ecto.Changeset.change(%{
      order_id:  event.order_id,
      user_id:   event.user_id,
      status:    "pending",
      total:     event.total,
      placed_at: DateTime.utc_now(),
      items:     event.items
    })
    |> Repo.insert!()
  end
  
  def project(%OrderConfirmed{order_id: id}) do
    Repo.get_by!(OrderSummary, order_id: id)
    |> Ecto.Changeset.change(status: "confirmed")
    |> Repo.update!()
  end
  
  def project(%OrderCancelled{order_id: id}) do
    Repo.get_by!(OrderSummary, order_id: id)
    |> Ecto.Changeset.change(status: "cancelled")
    |> Repo.update!()
  end
  
  def project(%OrderShipped{order_id: id}) do
    Repo.get_by!(OrderSummary, order_id: id)
    |> Ecto.Changeset.change(status: "shipped")
    |> Repo.update!()
  end
  
  # Rebuild projection from scratch
  def rebuild do
    Repo.delete_all(OrderSummary)
    
    MyApp.EventStore.read_all()
    |> Enum.each(&project/1)
  end
end
```

---

## Step 356: Event Subscription

```elixir
defmodule MyApp.EventSubscriber do
  use GenServer
  
  alias MyApp.{EventStore, Projector, Notifications}
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def init(_) do
    {:ok, last_position} = load_checkpoint()
    
    schedule_poll()
    {:ok, %{last_position: last_position}}
  end
  
  def handle_info(:poll, %{last_position: pos} = state) do
    new_events = EventStore.read_from(pos)
    
    Enum.each(new_events, fn {position, event} ->
      handle_event(event)
      save_checkpoint(position)
    end)
    
    new_pos = if new_events == [], do: pos, else: elem(List.last(new_events), 0)
    
    schedule_poll()
    {:noreply, %{state | last_position: new_pos}}
  end
  
  defp handle_event(event) do
    Projector.project(event)
    Notifications.notify(event)
  rescue
    e ->
      require Logger
      Logger.error("Event handling failed: #{inspect(e)}")
  end
  
  defp schedule_poll do
    Process.send_after(self(), :poll, 100)
  end
  
  defp load_checkpoint do
    pos = MyApp.Repo.get_value("event_subscriber_position") || 0
    {:ok, pos}
  end
  
  defp save_checkpoint(position) do
    MyApp.Repo.put_value("event_subscriber_position", position)
  end
end
```

---

## Step 357: Event Versioning

```elixir
defmodule MyApp.EventUpgrader do
  # Handle schema evolution of events
  
  def upgrade(%{"type" => "OrderPlaced", "version" => 1, "data" => data}) do
    # V1 didn't have currency field
    %{data | currency: "USD"}
  end
  
  def upgrade(%{"type" => "OrderPlaced", "version" => 2, "data" => data}) do
    data  # V2 is current
  end
  
  def upgrade(%{"type" => "UserRegistered", "version" => 1, "data" => data}) do
    # V1 had full_name, V2 splits it
    [first | rest] = String.split(data["full_name"], " ", parts: 2)
    last = Enum.join(rest, " ")
    
    data
    |> Map.delete("full_name")
    |> Map.put("first_name", first)
    |> Map.put("last_name", last)
  end
end
```

---

## Step 358: Snapshots

```elixir
defmodule MyApp.SnapshotStore do
  alias MyApp.Repo
  
  defmodule Snapshot do
    use Ecto.Schema
    
    schema "snapshots" do
      field :stream_id, :string
      field :version,   :integer
      field :state,     :map
      timestamps()
    end
  end
  
  def save(stream_id, version, state) do
    %Snapshot{
      stream_id: stream_id,
      version:   version,
      state:     :erlang.term_to_binary(state) |> Base.encode64()
    }
    |> Repo.insert!(on_conflict: :replace_all, conflict_target: :stream_id)
  end
  
  def load(stream_id) do
    case Repo.get_by(Snapshot, stream_id: stream_id) do
      nil -> nil
      snap ->
        state = snap.state |> Base.decode64!() |> :erlang.binary_to_term()
        {state, snap.version}
    end
  end
end

# Load aggregate with snapshot optimization
defmodule MyApp.CommandHandler do
  @snapshot_threshold 50  # snapshot every 50 events
  
  defp load_aggregate(stream_id) do
    case MyApp.SnapshotStore.load(stream_id) do
      nil ->
        events = EventStore.read(stream_id)
        {Order.rebuild(events), length(events) - 1}
      
      {snapshot_state, snapshot_version} ->
        # Only load events AFTER snapshot
        events = EventStore.read(stream_id, snapshot_version + 1)
        state = Enum.reduce(events, snapshot_state, &Order.apply(&2, &1))
        {state, snapshot_version + length(events)}
    end
  end
end
```

---

## Step 359: CQRS Query Side

```elixir
defmodule MyApp.Queries.OrderQueries do
  import Ecto.Query
  alias MyApp.{Repo, Projections.OrderSummary}
  
  def list_user_orders(user_id, opts \\ []) do
    status = Keyword.get(opts, :status)
    limit  = Keyword.get(opts, :limit, 20)
    
    query = from(o in OrderSummary,
      where: o.user_id == ^user_id,
      order_by: [desc: o.placed_at],
      limit: ^limit
    )
    
    query = if status, do: where(query, [o], o.status == ^status), else: query
    Repo.all(query)
  end
  
  def get_order_details(order_id) do
    case Repo.get_by(OrderSummary, order_id: order_id) do
      nil   -> {:error, :not_found}
      order -> {:ok, order}
    end
  end
  
  def order_stats do
    from(o in OrderSummary,
      group_by: o.status,
      select: {o.status, count(o.id), sum(o.total)}
    )
    |> Repo.all()
    |> Map.new(fn {status, count, total} ->
      {status, %{count: count, revenue: total}}
    end)
  end
end
```

---

## Step 360: Commanded (Event Sourcing Library)

```elixir
# mix.exs
{:commanded, "~> 1.4"},
{:commanded_ecto_projections, "~> 1.4"},
{:commanded_eventstore_adapter, "~> 1.4"},
{:eventstore, "~> 1.4"}

# aggregate using Commanded
defmodule MyApp.Commanded.Order do
  use Commanded.Aggregates.Aggregate
  
  defstruct [:id, :status, :total]
  
  # Command handler
  def execute(%{id: nil}, %PlaceOrder{} = cmd) do
    %OrderPlaced{
      order_id: cmd.order_id,
      user_id:  cmd.user_id,
      items:    cmd.items,
      total:    cmd.total
    }
  end
  
  def execute(%{status: :pending}, %ConfirmOrder{order_id: id}) do
    %OrderConfirmed{order_id: id, confirmed_at: DateTime.utc_now()}
  end
  
  # State mutators
  def apply(state, %OrderPlaced{order_id: id, total: total}) do
    %{state | id: id, status: :pending, total: total}
  end
  
  def apply(state, %OrderConfirmed{}) do
    %{state | status: :confirmed}
  end
end

# Dispatch command
defmodule MyApp.OrderService do
  alias MyApp.{App, Commanded.Order}
  
  def place_order(params) do
    cmd = %PlaceOrder{
      order_id: UUID.uuid4(),
      user_id:  params.user_id,
      items:    params.items,
      total:    params.total
    }
    
    App.dispatch(cmd, consistency: :strong)
  end
end
```

---

## สรุป Part 33

✅ **Step 351** - Event sourcing concepts  
✅ **Step 352** - Event store  
✅ **Step 353** - Aggregate  
✅ **Step 354** - Command handler  
✅ **Step 355** - Projections  
✅ **Step 356** - Event subscriptions  
✅ **Step 357** - Event versioning  
✅ **Step 358** - Snapshots  
✅ **Step 359** - CQRS query side  
✅ **Step 360** - Commanded library  

➡️ [Part 34: Kafka และ Message Queues](./part-34-kafka.md)
