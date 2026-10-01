# Part 17: Protocols และ Behaviours (Steps 181-200)

## Step 181: Protocols คืออะไร?

```
Protocol = Polymorphism ใน Elixir
- คล้าย Interface ใน Java/TypeScript
- เลือก implementation ตาม data type
- ไม่ต้องแก้ไข type เดิม (open for extension)
- สามารถ implement สำหรับ type ใดๆ ก็ได้
```

### 181.1 ตัวอย่าง Protocol ใน Elixir Standard Library

```elixir
# Inspect protocol - ทุก type implement ได้
inspect(42)         # "42"
inspect([1, 2, 3])  # "[1, 2, 3]"
inspect(%{a: 1})    # "%{a: 1}"

# Enumerable protocol
Enum.count([1, 2, 3])  # 3
Enum.count(1..10)      # 10

# String.Chars protocol
to_string(42)   # "42"
to_string(:foo) # "foo"
"Value: #{42}"  # works via String.Chars
```

---

## Step 182: สร้าง Protocol

```elixir
defprotocol Serializable do
  @doc "Serialize to JSON-compatible map"
  def to_map(value)
  
  @doc "Deserialize from map"
  def from_map(value, map)
end

# Implement สำหรับ custom type
defmodule User do
  defstruct [:id, :name, :email, :age]
end

defimpl Serializable, for: User do
  def to_map(%User{id: id, name: name, email: email, age: age}) do
    %{
      "id"    => id,
      "name"  => name,
      "email" => email,
      "age"   => age,
      "type"  => "user"
    }
  end
  
  def from_map(_, map) do
    %User{
      id:    Map.get(map, "id"),
      name:  Map.get(map, "name"),
      email: Map.get(map, "email"),
      age:   Map.get(map, "age")
    }
  end
end

defmodule Product do
  defstruct [:id, :name, :price, :sku]
end

defimpl Serializable, for: Product do
  def to_map(%Product{id: id, name: name, price: price, sku: sku}) do
    %{"id" => id, "name" => name, "price" => price, "sku" => sku, "type" => "product"}
  end
  
  def from_map(_, map) do
    %Product{
      id:    map["id"],
      name:  map["name"],
      price: map["price"],
      sku:   map["sku"]
    }
  end
end

# การใช้งาน
alice = %User{id: 1, name: "Alice", email: "alice@example.com", age: 30}
widget = %Product{id: 1, name: "Widget", price: 9.99, sku: "WGT-001"}

Serializable.to_map(alice)   # %{"id" => 1, "name" => "Alice", ...}
Serializable.to_map(widget)  # %{"id" => 1, "name" => "Widget", ...}
```

---

## Step 183: Protocol สำหรับ Built-in Types

```elixir
defprotocol Printable do
  def pretty_print(value)
end

# Implement สำหรับ built-in types
defimpl Printable, for: Integer do
  def pretty_print(n) when n >= 1_000 do
    n
    |> Integer.to_string()
    |> String.graphemes()
    |> Enum.reverse()
    |> Enum.chunk_every(3)
    |> Enum.join(",")
    |> String.reverse()
  end
  def pretty_print(n), do: Integer.to_string(n)
end

defimpl Printable, for: Float do
  def pretty_print(f), do: :erlang.float_to_binary(f, [decimals: 2])
end

defimpl Printable, for: List do
  def pretty_print(list) do
    items = Enum.map_join(list, ", ", &Printable.pretty_print/1)
    "[#{items}]"
  end
end

defimpl Printable, for: Map do
  def pretty_print(map) do
    pairs = Enum.map_join(map, "\n  ", fn {k, v} ->
      "#{k}: #{Printable.pretty_print(v)}"
    end)
    "{\n  #{pairs}\n}"
  end
end

# Catch-all with Any
defimpl Printable, for: Any do
  def pretty_print(value), do: inspect(value)
end

# ต้องระบุใน protocol:
defprotocol Printable do
  @fallback_to_any true
  def pretty_print(value)
end

# Usage
Printable.pretty_print(1_234_567)   # "1,234,567"
Printable.pretty_print(3.14159)     # "3.14"
Printable.pretty_print([1, 2, 3])   # "[1, 2, 3]"
```

---

## Step 184: Implement Enumerable Protocol

```elixir
defmodule NumberRange do
  defstruct [:from, :to, :step]
  
  def new(from, to, step \\ 1) do
    %__MODULE__{from: from, to: to, step: step}
  end
end

defimpl Enumerable, for: NumberRange do
  def count(%NumberRange{from: f, to: t, step: s}) do
    count = max(0, div(t - f, s) + 1)
    {:ok, count}
  end
  
  def member?(%NumberRange{from: f, to: t, step: s}, value) do
    result = value >= f && value <= t && rem(value - f, s) == 0
    {:ok, result}
  end
  
  def slice(%NumberRange{from: f, step: s} = range) do
    size = Enum.count(range)
    {:ok, size, fn start, len, _ ->
      Enum.map(start..(start + len - 1), fn i ->
        f + (i * s)
      end)
    end}
  end
  
  def reduce(range, acc, fun) do
    %NumberRange{from: f, to: t, step: s} = range
    do_reduce(f, t, s, acc, fun)
  end
  
  defp do_reduce(_current, _to, _step, {:halt, acc}, _fun) do
    {:halted, acc}
  end
  
  defp do_reduce(current, to, step, {:suspend, acc}, fun) do
    {:suspended, acc, fn new_acc -> do_reduce(current, to, step, new_acc, fun) end}
  end
  
  defp do_reduce(current, to, _step, acc, _fun) when current > to do
    {:done, elem(acc, 1)}
  end
  
  defp do_reduce(current, to, step, {:cont, acc}, fun) do
    do_reduce(current + step, to, step, fun.(current, acc), fun)
  end
end

# ใช้งานได้กับ Enum ทุก function!
r = NumberRange.new(1, 10, 2)  # 1, 3, 5, 7, 9

Enum.to_list(r)          # [1, 3, 5, 7, 9]
Enum.sum(r)              # 25
Enum.filter(r, &(&1 > 5))  # [7, 9]
Enum.map(r, &(&1 * 2))   # [2, 6, 10, 14, 18]
5 in r                   # true
6 in r                   # false
```

---

## Step 185: Behaviours

```elixir
# Behaviour = ระบุ interface ที่ module ต้องมี
# คล้าย abstract class หรือ interface

defmodule Storage do
  @callback put(key :: String.t(), value :: term()) :: :ok | {:error, term()}
  @callback get(key :: String.t()) :: {:ok, term()} | {:error, :not_found}
  @callback delete(key :: String.t()) :: :ok
  @callback list() :: [String.t()]
  
  @optional_callbacks [list: 0]
end

# Implement Memory storage
defmodule MemoryStorage do
  @behaviour Storage
  
  use Agent
  
  def start_link(_) do
    Agent.start_link(fn -> %{} end, name: __MODULE__)
  end
  
  @impl Storage
  def put(key, value) do
    Agent.update(__MODULE__, &Map.put(&1, key, value))
    :ok
  end
  
  @impl Storage
  def get(key) do
    case Agent.get(__MODULE__, &Map.get(&1, key)) do
      nil   -> {:error, :not_found}
      value -> {:ok, value}
    end
  end
  
  @impl Storage
  def delete(key) do
    Agent.update(__MODULE__, &Map.delete(&1, key))
    :ok
  end
  
  @impl Storage
  def list do
    Agent.get(__MODULE__, &Map.keys/1)
  end
end

# Implement File storage
defmodule FileStorage do
  @behaviour Storage
  
  @base_dir "/tmp/storage"
  
  @impl Storage
  def put(key, value) do
    File.mkdir_p!(@base_dir)
    path = Path.join(@base_dir, key)
    File.write(path, :erlang.term_to_binary(value))
  end
  
  @impl Storage
  def get(key) do
    path = Path.join(@base_dir, key)
    case File.read(path) do
      {:ok, binary} -> {:ok, :erlang.binary_to_term(binary)}
      {:error, :enoent} -> {:error, :not_found}
      {:error, reason}  -> {:error, reason}
    end
  end
  
  @impl Storage
  def delete(key) do
    path = Path.join(@base_dir, key)
    File.rm(path)
    :ok
  end
  
  @impl Storage
  def list do
    case File.ls(@base_dir) do
      {:ok, files} -> files
      {:error, _}  -> []
    end
  end
end
```

---

## Step 186: ใช้ Behaviour สำหรับ Pluggable Systems

```elixir
defmodule NotificationService do
  @behaviour MyApp.Notifier
  
  # ใช้ configured adapter
  def notify(recipient, message) do
    adapter().send(recipient, message)
  end
  
  defp adapter do
    Application.get_env(:my_app, :notifier, MyApp.EmailNotifier)
  end
end

defmodule MyApp.Notifier do
  @callback send(recipient :: String.t(), message :: String.t()) :: :ok | {:error, term()}
end

defmodule MyApp.EmailNotifier do
  @behaviour MyApp.Notifier
  
  @impl MyApp.Notifier
  def send(email, message) do
    IO.puts("Sending email to #{email}: #{message}")
    :ok
  end
end

defmodule MyApp.SmsNotifier do
  @behaviour MyApp.Notifier
  
  @impl MyApp.Notifier
  def send(phone, message) do
    IO.puts("Sending SMS to #{phone}: #{message}")
    :ok
  end
end

defmodule MyApp.MockNotifier do
  @behaviour MyApp.Notifier
  
  @impl MyApp.Notifier
  def send(_recipient, _message), do: :ok
end

# config/prod.exs
config :my_app, :notifier, MyApp.EmailNotifier

# config/test.exs
config :my_app, :notifier, MyApp.MockNotifier
```

---

## Step 187: GenServer กับ Behaviour

```elixir
defmodule JobProcessor do
  @callback process(job :: map()) :: {:ok, term()} | {:error, term()}
  @callback should_retry?(error :: term()) :: boolean()
  
  defmacro __using__(_opts) do
    quote do
      @behaviour JobProcessor
      use GenServer
      
      def start_link(opts \\ []) do
        GenServer.start_link(__MODULE__, opts, name: __MODULE__)
      end
      
      @impl GenServer
      def init(opts) do
        {:ok, %{processed: 0, failed: 0, opts: opts}}
      end
      
      @impl GenServer
      def handle_call({:process, job}, _from, state) do
        result = case process(job) do
          {:ok, _} = ok ->
            {ok, %{state | processed: state.processed + 1}}
          {:error, reason} = err ->
            if should_retry?(reason) do
              schedule_retry(job)
            end
            {err, %{state | failed: state.failed + 1}}
        end
        
        {reply, new_state} = result
        {:reply, reply, new_state}
      end
      
      @impl GenServer
      def handle_info({:retry, job}, state) do
        GenServer.call(self(), {:process, job})
        {:noreply, state}
      end
      
      # Default implementation
      @impl JobProcessor
      def should_retry?(_error), do: false
      
      defoverridable should_retry?: 1
      
      defp schedule_retry(job) do
        Process.send_after(self(), {:retry, job}, 5000)
      end
    end
  end
end

# สร้าง specific processor
defmodule EmailJobProcessor do
  use JobProcessor
  
  @impl JobProcessor
  def process(%{type: :email, to: to, subject: subject, body: body}) do
    # Send email logic
    IO.puts("Sending email to #{to}: #{subject}")
    {:ok, :sent}
  end
  def process(_), do: {:error, :invalid_job}
  
  @impl JobProcessor
  def should_retry?(:network_error), do: true
  def should_retry?(_), do: false
end
```

---

## Step 188: Protocol Consolidation

```elixir
# Protocol consolidation ทำให้เร็วขึ้นใน production
# โดย compile all implementations at once

# mix.exs
def project do
  [
    # ...
    consolidate_protocols: Mix.env() == :prod
  ]
end

# หรือใน config
# config :my_app, consolidate_protocols: true

# Dispatch เร็วขึ้นมาก (ไม่ต้อง lookup ทุกครั้ง)
# แต่ไม่สามารถเพิ่ม implementation ใหม่ได้ใน runtime
```

---

## Step 189: Advanced Protocol Patterns

```elixir
# Multi-protocol implementation
defmodule Circle do
  defstruct [:radius]
  
  def new(radius), do: %__MODULE__{radius: radius}
end

defprotocol Shape do
  def area(shape)
  def perimeter(shape)
  def contains?(shape, point)
end

defprotocol Drawable do
  def draw(shape, canvas)
  def bounding_box(shape)
end

defimpl Shape, for: Circle do
  import :math
  
  def area(%Circle{radius: r}), do: pi() * r * r
  
  def perimeter(%Circle{radius: r}), do: 2 * pi() * r
  
  def contains?(%Circle{radius: r}, {x, y}) do
    x * x + y * y <= r * r
  end
end

defimpl Drawable, for: Circle do
  def draw(%Circle{radius: r}, canvas) do
    # SVG circle
    "<circle cx='0' cy='0' r='#{r}'/>"
  end
  
  def bounding_box(%Circle{radius: r}) do
    {-r, -r, r * 2, r * 2}
  end
end

# Inspect protocol customization
defimpl Inspect, for: Circle do
  def inspect(%Circle{radius: r}, _opts) do
    "#Circle<r=#{r}, area=#{Float.round(:math.pi() * r * r, 2)}>"
  end
end

# IEx output
%Circle{radius: 5}
# #Circle<r=5, area=78.54>
```

---

## Step 190: Complete Design Pattern

```elixir
defprotocol Validator do
  @doc "Validate a value. Returns :ok or {:error, errors}"
  def validate(value)
end

defmodule Validation do
  defmodule Required do
    defstruct [:field, :message]
    
    def new(field, message \\ "is required") do
      %__MODULE__{field: field, message: message}
    end
  end
  
  defmodule Range do
    defstruct [:field, :min, :max]
    
    def new(field, min, max) do
      %__MODULE__{field: field, min: min, max: max}
    end
  end
  
  defmodule Format do
    defstruct [:field, :pattern, :message]
    
    def new(field, pattern, message \\ "has invalid format") do
      %__MODULE__{field: field, pattern: pattern, message: message}
    end
  end
end

defimpl Validator, for: Validation.Required do
  def validate(%{field: field, message: message}) do
    fn data ->
      if Map.get(data, field) in [nil, "", []] do
        {:error, [{field, message}]}
      else
        :ok
      end
    end
  end
end

defimpl Validator, for: Validation.Range do
  def validate(%{field: field, min: min, max: max}) do
    fn data ->
      value = Map.get(data, field)
      cond do
        value == nil    -> :ok
        value < min     -> {:error, [{field, "must be >= #{min}"}]}
        value > max     -> {:error, [{field, "must be <= #{max}"}]}
        true            -> :ok
      end
    end
  end
end

defimpl Validator, for: Validation.Format do
  def validate(%{field: field, pattern: pattern, message: message}) do
    fn data ->
      value = Map.get(data, field)
      if value && not (value =~ pattern) do
        {:error, [{field, message}]}
      else
        :ok
      end
    end
  end
end

defmodule Schema do
  def validate(data, rules) do
    errors = rules
      |> Enum.map(&Validator.validate/1)
      |> Enum.flat_map(fn checker ->
           case checker.(data) do
             :ok             -> []
             {:error, errs}  -> errs
           end
         end)
    
    if errors == [], do: {:ok, data}, else: {:error, errors}
  end
end

# Usage
user_rules = [
  Validation.Required.new(:name),
  Validation.Required.new(:email),
  Validation.Format.new(:email, ~r/@/, "must be valid email"),
  Validation.Range.new(:age, 0, 150)
]

Schema.validate(%{name: "Alice", email: "alice@example.com", age: 30}, user_rules)
# {:ok, %{name: "Alice", email: "alice@example.com", age: 30}}

Schema.validate(%{name: "", email: "bad-email", age: -5}, user_rules)
# {:error, [
#   {:name, "is required"},
#   {:email, "must be valid email"},
#   {:age, "must be >= 0"}
# ]}
```

---

## สรุป Part 17

✅ **Step 181** - Protocols คืออะไร  
✅ **Step 182** - สร้าง Protocol  
✅ **Step 183** - Implement สำหรับ built-in types  
✅ **Step 184** - Enumerable Protocol implementation  
✅ **Step 185** - Behaviours  
✅ **Step 186** - Pluggable systems ด้วย Behaviour  
✅ **Step 187** - GenServer + Behaviour  
✅ **Step 188** - Protocol consolidation  
✅ **Step 189** - Multi-protocol patterns  
✅ **Step 190** - Complete Validator pattern  

➡️ [Part 18: Metaprogramming และ Macros](./part-18-macros.md)
