# Part 98: Advanced Patterns (Steps 1041-1050)

## Step 1041: Event Sourcing

```elixir
defmodule MyApp.EventStore do
  use Ecto.Schema
  import Ecto.Query

  schema "events" do
    field :aggregate_id,   :string
    field :aggregate_type, :string
    field :event_type,     :string
    field :event_data,     :map
    field :metadata,       :map, default: %{}
    field :version,        :integer
    field :correlation_id, :string

    timestamps(updated_at: false)
  end

  def append(aggregate_id, type, events, expected_version \\ nil) do
    MyApp.Repo.transaction(fn ->
      current_version = get_current_version(aggregate_id)

      if expected_version && current_version != expected_version do
        MyApp.Repo.rollback({:concurrency_error, current_version, expected_version})
      end

      events
      |> Enum.with_index(current_version + 1)
      |> Enum.map(fn {event, version} ->
        %__MODULE__{
          aggregate_id:   aggregate_id,
          aggregate_type: type,
          event_type:     event.__struct__ |> to_string() |> String.split(".") |> List.last(),
          event_data:     Map.from_struct(event),
          version:        version,
          correlation_id: get_correlation_id()
        }
        |> MyApp.Repo.insert!()
      end)
    end)
  end

  def load(aggregate_id, from_version \\ 0) do
    from(e in __MODULE__,
      where: e.aggregate_id == ^aggregate_id and e.version > ^from_version,
      order_by: e.version
    ) |> MyApp.Repo.all()
  end

  defp get_current_version(aggregate_id) do
    from(e in __MODULE__,
      where: e.aggregate_id == ^aggregate_id,
      select: max(e.version)
    ) |> MyApp.Repo.one() || 0
  end

  defp get_correlation_id do
    Logger.metadata()[:request_id] || Ecto.UUID.generate()
  end
end

defmodule MyApp.Order.Aggregate do
  defstruct [:id, :status, :items, :total, :version]

  def new(id), do: %__MODULE__{id: id, status: :new, items: [], total: 0, version: 0}

  def apply(%__MODULE__{} = order, %{event_type: "OrderPlaced", event_data: data}) do
    %{order | status: :placed, items: data["items"], total: data["total"]}
  end
  def apply(%__MODULE__{} = order, %{event_type: "OrderShipped"}) do
    %{order | status: :shipped}
  end
  def apply(%__MODULE__{} = order, %{event_type: "OrderCanceled"}) do
    %{order | status: :canceled}
  end

  def rebuild(id) do
    events = MyApp.EventStore.load(id)
    Enum.reduce(events, new(id), &apply(&2, &1))
  end
end
```

---

## Step 1042: CQRS Pattern

```elixir
defmodule MyApp.CQRS do
  # Command side - writes only
  defmodule Commands do
    defmodule PlaceOrder do
      @enforce_keys [:user_id, :items]
      defstruct [:user_id, :items, :shipping_address]
    end

    defmodule CancelOrder do
      @enforce_keys [:order_id, :reason]
      defstruct [:order_id, :reason, :user_id]
    end
  end

  # Command handler
  defmodule CommandHandler do
    def handle(%Commands.PlaceOrder{} = cmd) do
      order_id = Ecto.UUID.generate()
      
      event = %MyApp.Events.OrderPlaced{
        order_id:         order_id,
        user_id:          cmd.user_id,
        items:            cmd.items,
        total:            calculate_total(cmd.items),
        shipping_address: cmd.shipping_address,
        placed_at:        DateTime.utc_now()
      }

      MyApp.EventStore.append(order_id, "Order", [event])
      {:ok, order_id}
    end

    def handle(%Commands.CancelOrder{} = cmd) do
      order = MyApp.Order.Aggregate.rebuild(cmd.order_id)

      if order.status in [:placed, :confirmed] do
        event = %MyApp.Events.OrderCanceled{
          order_id: cmd.order_id,
          reason:   cmd.reason,
          user_id:  cmd.user_id
        }
        MyApp.EventStore.append(cmd.order_id, "Order", [event])
        {:ok, :canceled}
      else
        {:error, :cannot_cancel, order.status}
      end
    end

    defp calculate_total(items) do
      Enum.sum(Enum.map(items, fn item -> item.price * item.quantity end))
    end
  end

  # Query side - reads only (from projected read models)
  defmodule Queries do
    def get_order(order_id) do
      MyApp.Repo.get(MyApp.ReadModels.OrderView, order_id)
    end

    def list_user_orders(user_id, opts \\ []) do
      import Ecto.Query
      from(o in MyApp.ReadModels.OrderView,
        where: o.user_id == ^user_id,
        order_by: [desc: o.placed_at],
        limit: ^Keyword.get(opts, :limit, 20)
      ) |> MyApp.Repo.all()
    end
  end
end
```

---

## Step 1043: Saga Pattern

```elixir
defmodule MyApp.Sagas.CheckoutSaga do
  @moduledoc "Coordinates checkout: inventory → payment → fulfillment"

  use GenServer

  defstruct [:id, :order_id, :state, :steps_completed, :compensation_stack]

  def start(order_id) do
    saga_id = Ecto.UUID.generate()
    state   = %__MODULE__{
      id: saga_id, order_id: order_id,
      state: :started, steps_completed: [], compensation_stack: []
    }
    
    GenServer.start_link(__MODULE__, state)
  end

  def init(state) do
    send(self(), :execute)
    {:ok, state}
  end

  def handle_info(:execute, state) do
    execute_saga(state)
  end

  defp execute_saga(state) do
    steps = [
      {:reserve_inventory,   &step_reserve/1,   &comp_release_inventory/1},
      {:charge_payment,      &step_charge/1,    &comp_refund/1},
      {:create_shipment,     &step_ship/1,      &comp_cancel_shipment/1},
      {:send_confirmation,   &step_email/1,     fn _ -> :ok end}
    ]

    result = Enum.reduce_while(steps, state, fn {name, step_fn, comp_fn}, acc ->
      case step_fn.(acc) do
        {:ok, new_state} ->
          updated = %{new_state |
            steps_completed: [name | acc.steps_completed],
            compensation_stack: [{name, comp_fn} | acc.compensation_stack]
          }
          {:cont, updated}
        {:error, reason} ->
          Logger.error("Saga step #{name} failed: #{inspect(reason)}")
          compensate(acc)
          {:halt, {:error, reason, acc}}
      end
    end)

    case result do
      %__MODULE__{} = final_state ->
        {:stop, :normal, %{final_state | state: :completed}}
      {:error, reason, _state} ->
        {:stop, :normal, {:failed, reason}}
    end
  end

  defp compensate(state) do
    Enum.each(state.compensation_stack, fn {name, comp_fn} ->
      case comp_fn.(state) do
        :ok -> Logger.info("Compensated: #{name}")
        {:error, r} -> Logger.error("Compensation failed for #{name}: #{inspect(r)}")
      end
    end)
  end

  defp step_reserve(state) do
    case MyApp.Inventory.reserve(state.order_id) do
      {:ok, reservation} -> {:ok, Map.put(state, :reservation_id, reservation.id)}
      error              -> error
    end
  end

  defp step_charge(state), do: {:ok, state}
  defp step_ship(state),   do: {:ok, state}
  defp step_email(state),  do: {:ok, state}

  defp comp_release_inventory(state), do: MyApp.Inventory.release(state.reservation_id)
  defp comp_refund(state),            do: MyApp.Payments.refund(state.order_id)
  defp comp_cancel_shipment(state),   do: MyApp.Shipping.cancel(state.shipment_id)
end
```

---

## Step 1044: Projection Workers

```elixir
defmodule MyApp.Projections.OrderSummary do
  use GenServer
  import Ecto.Query

  @checkpoint_interval 100  # Save checkpoint every 100 events

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    position = load_checkpoint()
    send(self(), {:process_from, position})
    {:ok, %{position: position, processed: 0}}
  end

  def handle_info({:process_from, position}, state) do
    events = load_events_after(position)

    new_position = Enum.reduce(events, position, fn event, pos ->
      apply_event(event)
      event.id
    end)

    count = state.processed + length(events)

    if rem(count, @checkpoint_interval) == 0 do
      save_checkpoint(new_position)
    end

    # Schedule next processing
    if length(events) > 0 do
      send(self(), {:process_from, new_position})
    else
      Process.send_after(self(), {:process_from, new_position}, 1000)
    end

    {:noreply, %{state | position: new_position, processed: count}}
  end

  defp apply_event(%{aggregate_type: "Order", event_type: "OrderPlaced"} = event) do
    MyApp.Repo.insert(%MyApp.ReadModels.OrderView{
      id:       event.aggregate_id,
      user_id:  event.event_data["user_id"],
      status:   "placed",
      total:    event.event_data["total"],
      placed_at: event.inserted_at
    }, on_conflict: :replace_all, conflict_target: :id)
  end

  defp apply_event(%{aggregate_type: "Order", event_type: "OrderShipped"} = event) do
    MyApp.Repo.update_all(
      from(o in MyApp.ReadModels.OrderView, where: o.id == ^event.aggregate_id),
      set: [status: "shipped"]
    )
  end

  defp apply_event(_), do: :ok

  defp load_events_after(nil), do: load_events_after(0)
  defp load_events_after(last_id) do
    from(e in MyApp.EventStore,
      where: e.id > ^last_id,
      order_by: e.id,
      limit: 500
    ) |> MyApp.Repo.all()
  end

  defp load_checkpoint do
    case MyApp.Repo.get(MyApp.ProjectionCheckpoint, "order_summary") do
      nil      -> 0
      %{position: pos} -> pos
    end
  end

  defp save_checkpoint(position) do
    MyApp.Repo.insert(%MyApp.ProjectionCheckpoint{name: "order_summary", position: position},
      on_conflict: :replace_all, conflict_target: :name)
  end
end
```

---

## Step 1045: Plugin Architecture

```elixir
defmodule MyApp.Plugins do
  # Dynamic plugin loading

  @type plugin :: module()

  def load_plugins do
    Application.get_env(:my_app, :plugins, [])
    |> Enum.each(fn plugin ->
      case Code.ensure_loaded(plugin) do
        {:module, _} ->
          plugin.init()
          Logger.info("Plugin loaded: #{plugin}")
        {:error, reason} ->
          Logger.error("Failed to load plugin #{plugin}: #{inspect(reason)}")
      end
    end)
  end

  def execute_hook(hook_name, data) do
    plugins = Application.get_env(:my_app, :plugins, [])
    
    Enum.reduce(plugins, {:ok, data}, fn plugin, {:ok, acc} ->
      if function_exported?(plugin, hook_name, 1) do
        apply(plugin, hook_name, [acc])
      else
        {:ok, acc}
      end
    end)
  end
end

# Plugin behaviour
defmodule MyApp.Plugin do
  @callback init() :: :ok | {:error, term()}
  @callback name() :: String.t()
  @callback version() :: String.t()
  @optional_callbacks [
    on_order_placed: 1,
    on_user_registered: 1,
    before_send_email: 1
  ]
end

# Example plugin
defmodule MyApp.Plugins.SlackNotifier do
  @behaviour MyApp.Plugin

  def init, do: :ok
  def name, do: "Slack Notifier"
  def version, do: "1.0.0"

  def on_order_placed(order) do
    webhook = Application.get_env(:my_app, :slack_webhook)
    Req.post(webhook, json: %{
      text: "New order ##{order.id}: $#{order.total / 100}"
    })
    {:ok, order}
  end
end
```

---

## Step 1046: Pipeline Pattern

```elixir
defmodule MyApp.Pipeline do
  @moduledoc "Composable pipeline for data transformations"

  def new(initial_value), do: {:ok, initial_value, []}

  def step(pipeline, name, fun) when is_function(fun, 1) do
    case pipeline do
      {:ok, value, log} ->
        try do
          case fun.(value) do
            {:ok, new_value} ->
              {:ok, new_value, [{name, :ok} | log]}
            {:error, reason} ->
              {:error, reason, [{name, :error} | log]}
          end
        rescue
          e -> {:error, {:exception, e}, [{name, :exception} | log]}
        end
      error -> error
    end
  end

  def run(pipeline) do
    case pipeline do
      {:ok, value, _log}    -> {:ok, value}
      {:error, reason, log} -> {:error, reason, Enum.reverse(log)}
    end
  end
end

# Usage
def process_order(params) do
  MyApp.Pipeline.new(params)
  |> MyApp.Pipeline.step(:validate,         &validate_order_params/1)
  |> MyApp.Pipeline.step(:reserve_stock,    &reserve_inventory/1)
  |> MyApp.Pipeline.step(:calculate_price,  &apply_pricing/1)
  |> MyApp.Pipeline.step(:apply_coupons,    &apply_discount_codes/1)
  |> MyApp.Pipeline.step(:calculate_tax,    &calculate_tax/1)
  |> MyApp.Pipeline.step(:process_payment,  &process_payment/1)
  |> MyApp.Pipeline.step(:create_shipment,  &create_shipment/1)
  |> MyApp.Pipeline.run()
end
```

---

## Step 1047: Middleware Pattern

```elixir
defmodule MyApp.Middleware do
  @type middleware_fun :: (term() -> term())
  @type handler_fun   :: (term() -> term())

  def compose(middlewares) do
    fn handler ->
      Enum.reduce(Enum.reverse(middlewares), handler, fn middleware, next ->
        fn ctx -> middleware.(ctx, next) end
      end)
    end
  end

  # Logging middleware
  def logging(ctx, next) do
    start  = System.monotonic_time(:microsecond)
    result = next.(ctx)
    time   = System.monotonic_time(:microsecond) - start
    Logger.debug("#{ctx.action} completed in #{time}µs")
    result
  end

  # Auth middleware
  def authenticate(ctx, next) do
    if ctx.user do
      next.(ctx)
    else
      {:error, :unauthenticated}
    end
  end

  # Caching middleware
  def cache(ctx, next) do
    cache_key = "#{ctx.action}:#{Jason.encode!(ctx.params)}"
    case Redix.command(:redix, ["GET", cache_key]) do
      {:ok, nil}     ->
        result = next.(ctx)
        if match?({:ok, _}, result) do
          Redix.command(:redix, ["SETEX", cache_key, 300, Jason.encode!(elem(result, 1))])
        end
        result
      {:ok, cached}  ->
        {:ok, Jason.decode!(cached)}
    end
  end
end

# Build a middleware stack
handler = MyApp.Middleware.compose([
  &MyApp.Middleware.logging/2,
  &MyApp.Middleware.authenticate/2,
  &MyApp.Middleware.cache/2
]).(fn ctx -> MyApp.Products.list(ctx.params) end)

handler.(%{action: "list_products", user: current_user, params: %{}})
```

---

## Step 1048: Observer Pattern

```elixir
defmodule MyApp.Observer do
  @moduledoc "Type-safe observer with pattern matching"

  defmacro __using__(_opts) do
    quote do
      import MyApp.Observer
      Module.register_attribute(__MODULE__, :subscriptions, accumulate: true)
      @before_compile MyApp.Observer
    end
  end

  defmacro __before_compile__(_env) do
    quote do
      def __subscriptions__, do: @subscriptions
    end
  end

  defmacro on(event_pattern, do: block) do
    quote do
      @subscriptions {unquote(Macro.escape(event_pattern)), fn event ->
        var!(event) = event
        unquote(block)
      end}
    end
  end
end

defmodule MyApp.OrderObserver do
  use MyApp.Observer

  on %{type: :order_placed, total: total} when total > 10_000 do
    Logger.info("Large order placed: #{event.id}")
    MyApp.VIPNotifier.notify(event)
  end

  on %{type: :order_shipped} do
    MyApp.Emails.shipping_notification(event.order_id)
  end
end

# Observer bus
defmodule MyApp.EventBus do
  def dispatch(event) do
    observers = Application.get_env(:my_app, :observers, [])

    Enum.each(observers, fn observer ->
      observer.__subscriptions__()
      |> Enum.each(fn {pattern, handler} ->
        if event_matches?(event, pattern) do
          Task.start(fn -> handler.(event) end)
        end
      end)
    end)
  end

  defp event_matches?(event, pattern) do
    try do
      match?(^pattern, event)
    rescue
      _ -> false
    end
  end
end
```

---

## Step 1049: Decorator Pattern via Macros

```elixir
defmodule MyApp.Decorators do
  defmacro __using__(_opts) do
    quote do
      import MyApp.Decorators
      Module.register_attribute(__MODULE__, :decorators, accumulate: true)
    end
  end

  defmacro decorate(decorator) do
    quote do
      @decorators unquote(decorator)
    end
  end

  # Memoize decorator
  defmacro memoize(opts \\ []) do
    ttl = Keyword.get(opts, :ttl, 300)
    quote do
      {:memoize, unquote(ttl)}
    end
  end

  # Apply decorators to function
  defmacro defdecorated(name, args, do: body) do
    quote do
      def unquote(name)(unquote_splicing(args)) do
        decorators = Module.get_attribute(__MODULE__, :decorators) || []
        Module.delete_attribute(__MODULE__, :decorators)
        apply_decorators(decorators, fn -> unquote(body) end, unquote(args))
      end
    end
  end

  def apply_decorators([], fun, _args), do: fun.()
  def apply_decorators([{:memoize, ttl} | rest], fun, args) do
    cache_key = :erlang.phash2({fun, args}) |> to_string()
    case Redix.command(:redix, ["GET", cache_key]) do
      {:ok, nil}    ->
        result = apply_decorators(rest, fun, args)
        Redix.command(:redix, ["SETEX", cache_key, ttl, :erlang.term_to_binary(result)])
        result
      {:ok, cached} ->
        :erlang.binary_to_term(cached)
    end
  end
end
```

---

## Step 1050: Circuit Breaker Pattern

```elixir
defmodule MyApp.CircuitBreaker do
  use GenServer

  @failure_threshold 5
  @success_threshold 2
  @timeout           30_000

  defstruct [:name, :state, :failures, :successes, :last_failure_at]

  def start_link(name) do
    GenServer.start_link(__MODULE__, name, name: via(name))
  end

  def call(name, fun, timeout \\ 5_000) do
    case GenServer.call(via(name), :check) do
      :open ->
        {:error, :circuit_open}
      :closed ->
        result = execute(fun, timeout)
        GenServer.cast(via(name), {:result, result})
        result
      :half_open ->
        result = execute(fun, timeout)
        GenServer.cast(via(name), {:result, result})
        result
    end
  end

  def init(name) do
    {:ok, %__MODULE__{name: name, state: :closed, failures: 0, successes: 0}}
  end

  def handle_call(:check, _from, %{state: :open} = state) do
    if DateTime.diff(DateTime.utc_now(), state.last_failure_at, :millisecond) > @timeout do
      {:reply, :half_open, %{state | state: :half_open}}
    else
      {:reply, :open, state}
    end
  end
  def handle_call(:check, _from, state), do: {:reply, state.state, state}

  def handle_cast({:result, {:ok, _}}, %{state: :half_open} = state) do
    if state.successes + 1 >= @success_threshold do
      {:noreply, %{state | state: :closed, failures: 0, successes: 0}}
    else
      {:noreply, %{state | successes: state.successes + 1}}
    end
  end

  def handle_cast({:result, {:ok, _}}, state) do
    {:noreply, %{state | failures: 0}}
  end

  def handle_cast({:result, {:error, _}}, state) do
    failures = state.failures + 1
    new_state = if failures >= @failure_threshold do
      Logger.warning("Circuit breaker #{state.name} opened!")
      :open
    else
      state.state
    end
    {:noreply, %{state | state: new_state, failures: failures, last_failure_at: DateTime.utc_now()}}
  end

  defp execute(fun, timeout) do
    task = Task.async(fun)
    case Task.yield(task, timeout) || Task.shutdown(task) do
      {:ok, result} -> result
      nil           -> {:error, :timeout}
      {:exit, e}    -> {:error, e}
    end
  end

  defp via(name), do: {:via, Registry, {MyApp.CircuitBreakerRegistry, name}}
end
```

---

## สรุป Part 98

✅ **Step 1041** - Event sourcing  
✅ **Step 1042** - CQRS pattern  
✅ **Step 1043** - Saga pattern  
✅ **Step 1044** - Projection workers  
✅ **Step 1045** - Plugin architecture  
✅ **Step 1046** - Pipeline pattern  
✅ **Step 1047** - Middleware pattern  
✅ **Step 1048** - Observer pattern  
✅ **Step 1049** - Decorator pattern  
✅ **Step 1050** - Circuit breaker pattern  

➡️ [Part 99: Ecosystem & Libraries](./part-99-ecosystem.md)
