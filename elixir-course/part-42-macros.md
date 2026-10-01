# Part 42: Advanced Macros (Steps 461-480)

## Step 461: Metaprogramming Concepts

```
Elixir metaprogramming:
- Code is data (AST = Abstract Syntax Tree)
- Macros transform code at compile time
- quote/2  → code → AST
- unquote/1 → inject value into quoted code
- Macro.expand/2 → expand macro
- use/require/import → module hooks

When to use macros:
✅ DSLs (Ecto schema, Phoenix router)
✅ Boilerplate reduction
✅ Compile-time optimization
❌ Runtime logic (use functions instead)
❌ Over-abstraction
```

---

## Step 462: AST Basics

```elixir
# Inspect AST
quote do
  1 + 2
end
# => {:+, [context: ...], [1, 2]}

quote do
  x = 1 + 2
end
# => {:=, [], [{:x, [], Elixir}, {:+, [], [1, 2]}]}

# Function call
quote do
  IO.puts("hello")
end
# => {{:., [], [{:__aliases__, [alias: false], [:IO]}, :puts]}, [], ["hello"]}

# Pattern match
iex> ast = quote do: x + 1
iex> Macro.to_string(ast)
"x + 1"

# Evaluate
iex> {result, _bindings} = Code.eval_quoted(quote do: 1 + 2)
iex> result
3
```

---

## Step 463: Simple Macros

```elixir
defmodule MyMacros do
  # Swap two variables
  defmacro swap({a, _, _} = left, {b, _, _} = right) do
    quote do
      tmp = unquote(left)
      unquote(left) = unquote(right)
      unquote(right) = tmp
    end
  end
  
  # Unless (opposite of if)
  defmacro unless(condition, do: block) do
    quote do
      if !unquote(condition), do: unquote(block)
    end
  end
  
  # While loop
  defmacro while(condition, do: block) do
    quote do
      Enum.reduce_while(Stream.cycle([nil]), nil, fn _, _ ->
        if unquote(condition) do
          unquote(block)
          {:cont, nil}
        else
          {:halt, nil}
        end
      end)
    end
  end
  
  # Retry on failure
  defmacro retry(times, do: block) do
    quote do
      Enum.reduce_while(1..unquote(times), nil, fn attempt, _ ->
        try do
          result = unquote(block)
          {:halt, {:ok, result}}
        rescue
          e ->
            if attempt == unquote(times) do
              {:halt, {:error, e}}
            else
              {:cont, nil}
            end
        end
      end)
    end
  end
end

# Usage
import MyMacros

unless false, do: IO.puts("runs")

retry 3 do
  flaky_network_call()
end
```

---

## Step 464: Module Attributes + Compile-time

```elixir
defmodule MyValidations do
  @rules []
  
  defmacro validates(field, opts) do
    quote do
      @rules [{unquote(field), unquote(opts)} | @rules]
    end
  end
  
  defmacro __before_compile__(_env) do
    quote do
      def validate(changeset) do
        Enum.reduce(@rules, changeset, fn {field, opts}, cs ->
          apply_validation(cs, field, opts)
        end)
      end
    end
  end
end

defmodule MySchema do
  use Ecto.Schema
  import MyValidations
  
  @before_compile MyValidations
  
  validates :name, required: true, min_length: 2
  validates :email, required: true, format: :email
  validates :age, min: 0, max: 150
end
```

---

## Step 465: DSL with Macros

```elixir
defmodule MyApp.StateMachine do
  defmacro __using__(_opts) do
    quote do
      Module.register_attribute(__MODULE__, :states, accumulate: true)
      Module.register_attribute(__MODULE__, :transitions, accumulate: true)
      
      import MyApp.StateMachine, only: [state: 2, transition: 3]
      
      @before_compile MyApp.StateMachine
    end
  end
  
  defmacro state(name, opts \\ []) do
    quote do
      @states {unquote(name), unquote(opts)}
    end
  end
  
  defmacro transition(from, to, on: event) do
    quote do
      @transitions {unquote(from), unquote(to), unquote(event)}
    end
  end
  
  defmacro __before_compile__(_env) do
    quote do
      def states, do: @states |> Enum.map(fn {name, _} -> name end)
      
      def can_transition?(current, event) do
        Enum.any?(@transitions, fn {from, _to, ev} ->
          from == current && ev == event
        end)
      end
      
      def transition(current, event) do
        case Enum.find(@transitions, fn {from, _to, ev} ->
          from == current && ev == event
        end) do
          {_from, to, _event} -> {:ok, to}
          nil -> {:error, :invalid_transition}
        end
      end
    end
  end
end

defmodule OrderStateMachine do
  use MyApp.StateMachine
  
  state :pending
  state :confirmed
  state :shipped
  state :delivered
  state :cancelled
  
  transition :pending,   :confirmed, on: :confirm
  transition :pending,   :cancelled, on: :cancel
  transition :confirmed, :shipped,   on: :ship
  transition :shipped,   :delivered, on: :deliver
end

# Usage
OrderStateMachine.states()
# [:pending, :confirmed, :shipped, :delivered, :cancelled]

OrderStateMachine.transition(:pending, :confirm)
# {:ok, :confirmed}
```

---

## Step 466: Code Generation

```elixir
defmodule MyApp.CRUD do
  defmacro __using__(opts) do
    schema_module = Keyword.fetch!(opts, :schema)
    
    quote do
      alias unquote(schema_module)
      alias MyApp.Repo
      import Ecto.Query
      
      @schema_module unquote(schema_module)
      
      def list(opts \\ []) do
        filters = Keyword.get(opts, :where, [])
        order   = Keyword.get(opts, :order_by, :id)
        
        @schema_module
        |> where(^filters)
        |> order_by(^order)
        |> Repo.all()
      end
      
      def get(id) do
        Repo.get(@schema_module, id)
      end
      
      def get!(id) do
        Repo.get!(@schema_module, id)
      end
      
      def create(attrs) do
        %@schema_module{}
        |> @schema_module.changeset(attrs)
        |> Repo.insert()
      end
      
      def update(%@schema_module{} = record, attrs) do
        record
        |> @schema_module.changeset(attrs)
        |> Repo.update()
      end
      
      def delete(%@schema_module{} = record) do
        Repo.delete(record)
      end
      
      defoverridable [list: 0, list: 1, get: 1, create: 1, update: 2, delete: 1]
    end
  end
end

defmodule MyApp.Users do
  use MyApp.CRUD, schema: MyApp.Accounts.User
  
  # Override to add custom logic
  def list(opts \\ []) do
    opts
    |> Keyword.put_new(:order_by, :name)
    |> super()
  end
end
```

---

## Step 467: Macro Hygiene

```elixir
defmodule SafeMacros do
  # SAFE: vars defined in macro don't leak into caller's scope
  defmacro safe_macro(value) do
    quote do
      # This `result` won't conflict with caller's `result`
      result = unquote(value) * 2
      result + 1
    end
  end
  
  # UNSAFE: var leaks into caller scope (avoid)
  defmacro unsafe_macro(value) do
    quote hygiene: [vars: false] do
      result = unquote(value) * 2  # BAD: leaks `result`
      result + 1
    end
  end
  
  # Use var! to intentionally interact with caller's variables
  defmacro set_x(value) do
    quote do
      var!(x) = unquote(value)
    end
  end
end

# Test hygiene
defmodule Test do
  import SafeMacros
  
  def test do
    result = "original"
    safe_result = safe_macro(5)
    IO.puts(result)       # => "original" (not changed)
    IO.puts(safe_result)  # => 11
  end
end
```

---

## Step 468: __using__ Hook

```elixir
defmodule MyApp.Service do
  defmacro __using__(opts) do
    service_name = Keyword.get(opts, :name, __CALLER__.module)
    
    quote do
      use GenServer
      require Logger
      
      @service_name unquote(service_name)
      
      def start_link(opts \\ []) do
        GenServer.start_link(__MODULE__, opts, name: __MODULE__)
      end
      
      def init(opts) do
        Logger.info("Starting service: #{@service_name}")
        
        state = initialize(opts)
        
        :telemetry.execute(
          [:my_app, :service, :started],
          %{},
          %{service: @service_name}
        )
        
        {:ok, state}
      end
      
      # Default implementation (can be overridden)
      def initialize(_opts), do: %{}
      
      def handle_info(:health_check, state) do
        Logger.debug("Health check: #{@service_name}")
        {:noreply, state}
      end
      
      defoverridable [initialize: 1]
    end
  end
end

defmodule MyApp.PaymentService do
  use MyApp.Service, name: :payment_service
  
  def initialize(_opts) do
    %{processed: 0, failed: 0}
  end
  
  def handle_call({:charge, amount}, _from, state) do
    # ...
    {:reply, :ok, %{state | processed: state.processed + 1}}
  end
end
```

---

## Step 469: Protocol Macros

```elixir
defprotocol MyApp.Serializable do
  @doc "Serialize to map"
  def to_map(data)
  
  @doc "Deserialize from map"
  def from_map(data, module)
end

defmacro derive_serializable(fields) do
  quote do
    defimpl MyApp.Serializable do
      def to_map(struct) do
        Map.take(struct, unquote(fields))
      end
      
      def from_map(map, module) do
        struct(module, map)
      end
    end
  end
end

defmodule User do
  defstruct [:id, :name, :email, :password_hash]
  
  derive_serializable([:id, :name, :email])  # excludes password_hash
end

# Usage
user = %User{id: 1, name: "Alice", email: "alice@example.com", password_hash: "secret"}
map = MyApp.Serializable.to_map(user)
# %{id: 1, name: "Alice", email: "alice@example.com"}
```

---

## Step 470: Macro Testing

```elixir
defmodule MyMacrosTest do
  use ExUnit.Case
  
  describe "unless/2" do
    test "executes block when condition is false" do
      result = unless false, do: "ran"
      assert result == "ran"
    end
    
    test "does not execute block when condition is true" do
      result = unless true, do: "ran"
      assert result == nil
    end
  end
  
  describe "state machine DSL" do
    test "generates state list" do
      assert :pending in OrderStateMachine.states()
    end
    
    test "valid transition" do
      assert {:ok, :confirmed} = OrderStateMachine.transition(:pending, :confirm)
    end
    
    test "invalid transition" do
      assert {:error, :invalid_transition} = OrderStateMachine.transition(:pending, :ship)
    end
  end
  
  # Test macro expansion
  test "unless expands correctly" do
    ast = quote do
      unless condition, do: block
    end
    
    expanded = Macro.expand(ast, __ENV__)
    expanded_str = Macro.to_string(expanded)
    
    assert String.contains?(expanded_str, "if")
  end
end
```

---

## สรุป Part 42

✅ **Step 461** - Metaprogramming concepts  
✅ **Step 462** - AST basics  
✅ **Step 463** - Simple macros  
✅ **Step 464** - Module attributes + compile-time  
✅ **Step 465** - DSL with macros  
✅ **Step 466** - Code generation  
✅ **Step 467** - Macro hygiene  
✅ **Step 468** - __using__ hook  
✅ **Step 469** - Protocol macros  
✅ **Step 470** - Macro testing  

➡️ [Part 43: Real-Time Systems](./part-43-realtime.md)
