# Part 69: Event Sourcing & CQRS (Steps 751-770)

## Step 751: Event Sourcing Fundamentals

```
Traditional CRUD:
  ┌────────┐
  │ users  │
  │ id     │ ← Only current state
  │ email  │
  │ plan   │
  └────────┘

Event Sourcing:
  ┌──────────────────────────────────────────────┐
  │ events (append-only)                         │
  │ UserRegistered  { email: "a@a.com" }         │
  │ UserUpgraded    { plan: "pro" }              │
  │ UserEmailChanged { email: "b@b.com" }        │  ← Full history
  └──────────────────────────────────────────────┘
  
  Current state = fold over events
  
Benefits:
  ✅ Complete audit log
  ✅ Temporal queries ("what was state on date X?")
  ✅ Event replay → rebuild projections
  ✅ Debugging: replay events to reproduce issues
  ✅ Multiple read models from same events
  
Tradeoffs:
  ❌ More complex to implement
  ❌ Eventual consistency in projections
  ❌ Event schema evolution needs care
```

---

## Step 752: Event Schema

```elixir
defmodule MyApp.Events do
  # All events as structs with version

  defmodule UserRegistered do
    @enforce_keys [:user_id, :email, :name]
    defstruct [:user_id, :email, :name, :version]
    def version, do: 1
  end

  defmodule UserEmailChanged do
    @enforce_keys [:user_id, :old_email, :new_email]
    defstruct [:user_id, :old_email, :new_email, :version]
    def version, do: 1
  end

  defmodule UserUpgraded do
    @enforce_keys [:user_id, :old_plan, :new_plan]
    defstruct [:user_id, :old_plan, :new_plan, :version]
    def version, do: 1
  end

  defmodule OrderPlaced do
    @enforce_keys [:order_id, :user_id, :items, :total]
    defstruct [:order_id, :user_id, :items, :total, :version]
    def version, do: 1
  end

  defmodule OrderShipped do
    @enforce_keys [:order_id, :tracking_number]
    defstruct [:order_id, :tracking_number, :version]
    def version, do: 1
  end
end
```

---

## Step 753: Event Store

```elixir
defmodule MyApp.EventStore do
  alias MyApp.Repo
  alias MyApp.EventStore.StoredEvent

  def append(stream_id, events, expected_version \\ :any) do
    Repo.transaction(fn ->
      current_version = get_stream_version(stream_id)

      case expected_version do
        :any  -> :ok
        v when v == current_version -> :ok
        _ -> Repo.rollback({:wrong_expected_version, current_version})
      end

      stored = events
        |> Enum.with_index(current_version + 1)
        |> Enum.map(fn {event, version} ->
          %{
            stream_id:  stream_id,
            event_type: event.__struct__ |> Module.split() |> List.last(),
            data:       serialize(event),
            version:    version,
            inserted_at: DateTime.utc_now()
          }
        end)

      {count, _} = Repo.insert_all(StoredEvent, stored)

      Phoenix.PubSub.broadcast(MyApp.PubSub, "events:#{stream_id}", {:events, events})

      {:ok, count}
    end)
  end

  def read_stream(stream_id, from_version \\ 0) do
    import Ecto.Query

    StoredEvent
    |> where([e], e.stream_id == ^stream_id and e.version > ^from_version)
    |> order_by([e], e.version)
    |> Repo.all()
    |> Enum.map(&deserialize/1)
  end

  def read_all_events(from_position \\ 0) do
    import Ecto.Query

    StoredEvent
    |> where([e], e.id > ^from_position)
    |> order_by([e], e.id)
    |> Repo.all()
    |> Enum.map(&deserialize/1)
  end

  defp get_stream_version(stream_id) do
    import Ecto.Query

    StoredEvent
    |> where([e], e.stream_id == ^stream_id)
    |> select([e], max(e.version))
    |> Repo.one() || 0
  end

  defp serialize(event) do
    event
    |> Map.from_struct()
    |> Jason.encode!()
  end

  defp deserialize(%{event_type: type, data: data}) do
    module = Module.concat([MyApp.Events, type])
    data = Jason.decode!(data, keys: :atoms)
    struct(module, data)
  end
end

defmodule MyApp.EventStore.StoredEvent do
  use Ecto.Schema

  schema "stored_events" do
    field :stream_id,  :string
    field :event_type, :string
    field :data,       :string
    field :version,    :integer
    timestamps(updated_at: false)
  end
end
```

---

## Step 754: Aggregates

```elixir
defmodule MyApp.Aggregates.User do
  defstruct [:id, :email, :name, :plan, :version, events: []]

  alias MyApp.Events.{UserRegistered, UserEmailChanged, UserUpgraded}

  # Command handlers: validate and produce events
  def register(id, email, name) do
    user = %__MODULE__{id: id}
    event = %UserRegistered{user_id: id, email: email, name: name}
    {:ok, apply_event(user, event), [event]}
  end

  def change_email(%__MODULE__{} = user, new_email) do
    if user.email == new_email do
      {:error, :same_email}
    else
      event = %UserEmailChanged{
        user_id:   user.id,
        old_email: user.email,
        new_email: new_email
      }
      {:ok, apply_event(user, event), [event]}
    end
  end

  def upgrade(%__MODULE__{} = user, new_plan) do
    event = %UserUpgraded{
      user_id:  user.id,
      old_plan: user.plan,
      new_plan: new_plan
    }
    {:ok, apply_event(user, event), [event]}
  end

  # Event application: pure state transition
  def apply_event(user, %UserRegistered{} = event) do
    %{user | id: event.user_id, email: event.email, name: event.name}
  end

  def apply_event(user, %UserEmailChanged{} = event) do
    %{user | email: event.new_email}
  end

  def apply_event(user, %UserUpgraded{} = event) do
    %{user | plan: event.new_plan}
  end

  # Rebuild from event history
  def from_events(events) do
    Enum.reduce(events, %__MODULE__{}, &apply_event(&2, &1))
  end
end
```

---

## Step 755: Aggregate Repository

```elixir
defmodule MyApp.Repository do
  alias MyApp.EventStore

  def load(module, aggregate_id) do
    events = EventStore.read_stream("#{module_name(module)}-#{aggregate_id}")
    
    if Enum.empty?(events) do
      {:error, :not_found}
    else
      aggregate = module.from_events(events)
      {:ok, %{aggregate | version: length(events)}}
    end
  end

  def save(%{id: id, version: version} = aggregate, new_events) do
    stream_id = "#{module_name(aggregate.__struct__)}-#{id}"
    
    case EventStore.append(stream_id, new_events, version) do
      {:ok, _}         -> {:ok, %{aggregate | version: version + length(new_events)}}
      {:error, reason} -> {:error, reason}
    end
  end

  defp module_name(module) do
    module
    |> Module.split()
    |> List.last()
    |> Macro.underscore()
  end
end

# Usage
defmodule MyApp.UserService do
  alias MyApp.{Repository, Aggregates}

  def register_user(email, name) do
    id = Ecto.UUID.generate()
    {:ok, user, events} = Aggregates.User.register(id, email, name)
    {:ok, saved} = Repository.save(user, events)
    {:ok, saved}
  end

  def change_email(user_id, new_email) do
    with {:ok, user} <- Repository.load(Aggregates.User, user_id),
         {:ok, updated, events} <- Aggregates.User.change_email(user, new_email),
         {:ok, saved} <- Repository.save(updated, events) do
      {:ok, saved}
    end
  end
end
```

---

## Step 756: Projections (Read Models)

```elixir
defmodule MyApp.Projections.UserProjection do
  use GenServer

  alias MyApp.Events.{UserRegistered, UserEmailChanged, UserUpgraded}

  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)

  def init(_) do
    MyApp.EventStore.subscribe_to_all(self())
    replay_history()
    {:ok, %{}}
  end

  def handle_info({:event, event}, state) do
    apply_event(event)
    {:noreply, state}
  end

  defp replay_history do
    MyApp.EventStore.read_all_events()
    |> Enum.each(&apply_event/1)
  end

  defp apply_event(%UserRegistered{} = event) do
    MyApp.Repo.insert!(%MyApp.UserView{
      id:    event.user_id,
      email: event.email,
      name:  event.name,
      plan:  "free"
    }, on_conflict: :nothing)
  end

  defp apply_event(%UserEmailChanged{} = event) do
    import Ecto.Query
    MyApp.Repo.update_all(
      from(u in MyApp.UserView, where: u.id == ^event.user_id),
      set: [email: event.new_email]
    )
  end

  defp apply_event(%UserUpgraded{} = event) do
    import Ecto.Query
    MyApp.Repo.update_all(
      from(u in MyApp.UserView, where: u.id == ^event.user_id),
      set: [plan: event.new_plan]
    )
  end

  defp apply_event(_), do: :ok
end
```

---

## Step 757: CQRS Command Bus

```elixir
defmodule MyApp.CommandBus do
  # Route commands to their handlers

  @handlers %{
    MyApp.Commands.RegisterUser   => MyApp.Handlers.UserHandler,
    MyApp.Commands.ChangeEmail    => MyApp.Handlers.UserHandler,
    MyApp.Commands.PlaceOrder     => MyApp.Handlers.OrderHandler,
    MyApp.Commands.ShipOrder      => MyApp.Handlers.OrderHandler,
  }

  def dispatch(command) do
    module = command.__struct__

    case Map.get(@handlers, module) do
      nil     -> {:error, {:no_handler, module}}
      handler -> handler.handle(command)
    end
  end

  def dispatch!(command) do
    case dispatch(command) do
      {:ok, result}    -> result
      {:error, reason} -> raise "Command failed: #{inspect(reason)}"
    end
  end
end

defmodule MyApp.Handlers.UserHandler do
  alias MyApp.{Repository, Aggregates, Commands}

  def handle(%Commands.RegisterUser{} = cmd) do
    MyApp.UserService.register_user(cmd.email, cmd.name)
  end

  def handle(%Commands.ChangeEmail{} = cmd) do
    MyApp.UserService.change_email(cmd.user_id, cmd.new_email)
  end
end

defmodule MyApp.Commands.RegisterUser do
  @enforce_keys [:email, :name]
  defstruct [:email, :name]
end

defmodule MyApp.Commands.ChangeEmail do
  @enforce_keys [:user_id, :new_email]
  defstruct [:user_id, :new_email]
end
```

---

## Step 758: Temporal Queries

```elixir
defmodule MyApp.TemporalQuery do
  # Reconstruct state at any point in time

  def state_at(aggregate_id, datetime) do
    import Ecto.Query

    events = from(e in MyApp.EventStore.StoredEvent,
      where: e.stream_id == ^aggregate_id
         and e.inserted_at <= ^datetime,
      order_by: e.version
    )
    |> MyApp.Repo.all()
    |> Enum.map(&MyApp.EventStore.deserialize/1)

    MyApp.Aggregates.User.from_events(events)
  end

  def changes_between(aggregate_id, from_dt, to_dt) do
    import Ecto.Query

    from(e in MyApp.EventStore.StoredEvent,
      where: e.stream_id == ^aggregate_id
         and e.inserted_at >= ^from_dt
         and e.inserted_at <= ^to_dt,
      order_by: e.version
    )
    |> MyApp.Repo.all()
    |> Enum.map(&MyApp.EventStore.deserialize/1)
  end
end
```

---

## Step 759: Event Upcasting

```elixir
defmodule MyApp.EventUpcaster do
  # Handle schema evolution: upgrade old events to current version

  def upcast(%{event_type: "UserRegistered", version: 1, data: data}) do
    decoded = Jason.decode!(data, keys: :atoms)
    # v1 → v2: add default_locale field
    upgraded = Map.put_new(decoded, :locale, "en")
    %{event_type: "UserRegistered", version: 2, data: Jason.encode!(upgraded)}
  end

  def upcast(%{event_type: "OrderPlaced", version: 1, data: data}) do
    decoded = Jason.decode!(data, keys: :atoms)
    # v1 used "products" key, v2 uses "items"
    upgraded = decoded
      |> Map.put(:items, decoded[:products])
      |> Map.delete(:products)
    %{event_type: "OrderPlaced", version: 2, data: Jason.encode!(upgraded)}
  end

  def upcast(event), do: event

  def upcast_all(events) do
    Enum.map(events, &upcast/1)
  end
end
```

---

## Step 760: Saga / Process Manager

```elixir
defmodule MyApp.Sagas.OrderFulfillment do
  # Coordinates multi-step business process across aggregates
  use GenServer

  defstruct [:order_id, :state, :payment_id, :shipment_id]

  def start(order_id) do
    GenServer.start_link(__MODULE__, order_id)
  end

  def init(order_id) do
    MyApp.PubSub.subscribe("order:#{order_id}")
    state = %__MODULE__{order_id: order_id, state: :awaiting_payment}
    {:ok, state}
  end

  def handle_info({:event, %MyApp.Events.PaymentSucceeded{} = event}, state) do
    if event.order_id == state.order_id do
      # Payment succeeded, now create shipment
      {:ok, shipment} = MyApp.Shipping.create_shipment(state.order_id)
      
      new_state = %{state |
        state:       :awaiting_shipment,
        payment_id:  event.payment_id,
        shipment_id: shipment.id
      }
      {:noreply, new_state}
    else
      {:noreply, state}
    end
  end

  def handle_info({:event, %MyApp.Events.PaymentFailed{} = event}, state) do
    if event.order_id == state.order_id do
      # Compensate: cancel the order
      MyApp.Orders.cancel(state.order_id, :payment_failed)
      {:stop, :normal, state}
    else
      {:noreply, state}
    end
  end

  def handle_info({:event, %MyApp.Events.OrderShipped{} = event}, state) do
    if event.order_id == state.order_id do
      # Order complete
      MyApp.Orders.complete(state.order_id)
      {:stop, :normal, state}
    else
      {:noreply, state}
    end
  end
end
```

---

## สรุป Part 69

✅ **Step 751** - Event sourcing fundamentals  
✅ **Step 752** - Event schema  
✅ **Step 753** - Event store  
✅ **Step 754** - Aggregates  
✅ **Step 755** - Aggregate repository  
✅ **Step 756** - Projections  
✅ **Step 757** - CQRS command bus  
✅ **Step 758** - Temporal queries  
✅ **Step 759** - Event upcasting  
✅ **Step 760** - Saga / process manager  

➡️ [Part 70: Multi-Tenancy](./part-70-multi-tenancy.md)
