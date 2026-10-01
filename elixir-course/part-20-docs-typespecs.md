# Part 20: Documentation และ Typespecs (Steps 211-230)

## Step 211: @moduledoc และ @doc

```elixir
defmodule MathUtils do
  @moduledoc """
  Utility functions for mathematical operations.
  
  ## Examples
  
      iex> MathUtils.factorial(5)
      120
      
      iex> MathUtils.fibonacci(10)
      55
  
  ## Notes
  
  All functions work with positive integers unless stated otherwise.
  """
  
  @doc """
  Calculate the factorial of a number.
  
  ## Parameters
  
  - `n` - A non-negative integer
  
  ## Returns
  
  The factorial of `n`.
  
  ## Examples
  
      iex> MathUtils.factorial(0)
      1
      
      iex> MathUtils.factorial(5)
      120
      
      iex> MathUtils.factorial(10)
      3628800
  
  ## Raises
  
  `ArgumentError` if `n` is negative.
  """
  def factorial(0), do: 1
  def factorial(n) when n > 0 do
    n * factorial(n - 1)
  end
  def factorial(n) when n < 0 do
    raise ArgumentError, "factorial requires non-negative integer, got: #{n}"
  end
  
  @doc """
  Calculate the nth Fibonacci number.
  
  Uses tail-call optimization for efficiency.
  
  ## Examples
  
      iex> MathUtils.fibonacci(0)
      0
      
      iex> MathUtils.fibonacci(1)
      1
      
      iex> MathUtils.fibonacci(10)
      55
  """
  def fibonacci(n), do: fib(n, 0, 1)
  
  defp fib(0, a, _b), do: a
  defp fib(n, a, b),  do: fib(n - 1, b, a + b)
end
```

---

## Step 212: @doc false และ Private Documentation

```elixir
defmodule InternalModule do
  @moduledoc """
  Public module documentation.
  """
  
  # Document public functions
  @doc """
  Public function that is exported.
  """
  def public_function(x) do
    internal_helper(x) * 2
  end
  
  # Hide private details from docs
  @doc false
  def semi_private_function(x) do
    # This won't show in generated docs
    x + 1
  end
  
  # Private functions don't need @doc
  defp internal_helper(x) do
    x * x
  end
end
```

---

## Step 213: Doctests

```elixir
defmodule StringUtils do
  @moduledoc """
  String manipulation utilities.
  """
  
  @doc """
  Capitalize each word in a string.
  
  ## Examples
  
      iex> StringUtils.title_case("hello world")
      "Hello World"
      
      iex> StringUtils.title_case("the quick brown fox")
      "The Quick Brown Fox"
      
      iex> StringUtils.title_case("")
      ""
  """
  def title_case(""), do: ""
  def title_case(str) do
    str
    |> String.split()
    |> Enum.map(&String.capitalize/1)
    |> Enum.join(" ")
  end
  
  @doc """
  Truncate a string to a given length.
  
  ## Examples
  
      iex> StringUtils.truncate("Hello World", 5)
      "Hello..."
      
      iex> StringUtils.truncate("Hi", 10)
      "Hi"
      
      iex> StringUtils.truncate("Hello World", 5, suffix: "")
      "Hello"
  """
  def truncate(str, max_length, opts \\ []) do
    suffix = Keyword.get(opts, :suffix, "...")
    
    if String.length(str) > max_length do
      String.slice(str, 0, max_length) <> suffix
    else
      str
    end
  end
end

# test/string_utils_test.exs
defmodule StringUtilsTest do
  use ExUnit.Case, async: true
  
  doctest StringUtils  # auto-run all @doc examples as tests
end
```

---

## Step 214: @spec - Type Specifications

```elixir
defmodule TypedFunctions do
  @doc "Add two numbers"
  @spec add(number(), number()) :: number()
  def add(a, b), do: a + b
  
  @doc "Convert to string"
  @spec to_string_safe(term()) :: String.t()
  def to_string_safe(value), do: "#{inspect(value)}"
  
  @doc "Find element in list"
  @spec find(list(a), (a -> boolean())) :: a | nil when a: term()
  def find(list, pred) do
    Enum.find(list, pred)
  end
  
  @doc "Parse integer safely"
  @spec parse_int(String.t()) :: {:ok, integer()} | {:error, :not_a_number}
  def parse_int(str) do
    case Integer.parse(str) do
      {n, ""}  -> {:ok, n}
      _        -> {:error, :not_a_number}
    end
  end
  
  @doc "Get map value with default"
  @spec get_or_default(map(), term(), term()) :: term()
  def get_or_default(map, key, default) do
    Map.get(map, key, default)
  end
end
```

---

## Step 215: @type และ @typep

```elixir
defmodule Types do
  # Public type alias
  @type user_id :: pos_integer()
  @type username :: String.t()
  @type email :: String.t()
  
  # Struct type
  @type t :: %__MODULE__{
    id:    user_id(),
    name:  username(),
    email: email(),
    age:   non_neg_integer(),
    role:  :admin | :user | :guest
  }
  
  defstruct [:id, :name, :email, :age, role: :user]
  
  # Type for function arguments
  @type create_attrs :: %{
    required(:name)  => username(),
    required(:email) => email(),
    optional(:age)   => non_neg_integer(),
    optional(:role)  => :admin | :user | :guest
  }
  
  @doc "Create a new user"
  @spec new(create_attrs()) :: {:ok, t()} | {:error, String.t()}
  def new(attrs) do
    with {:ok, id}    <- generate_id(),
         {:ok, name}  <- validate_name(attrs[:name]),
         {:ok, email} <- validate_email(attrs[:email]) do
      user = %__MODULE__{
        id: id,
        name: name,
        email: email,
        age: attrs[:age],
        role: attrs[:role] || :user
      }
      {:ok, user}
    end
  end
  
  @spec generate_id() :: {:ok, user_id()}
  defp generate_id, do: {:ok, :rand.uniform(999_999)}
  
  @spec validate_name(String.t() | nil) :: {:ok, String.t()} | {:error, String.t()}
  defp validate_name(nil), do: {:error, "name is required"}
  defp validate_name(""),  do: {:error, "name cannot be empty"}
  defp validate_name(name) when is_binary(name), do: {:ok, String.trim(name)}
  
  @spec validate_email(String.t() | nil) :: {:ok, String.t()} | {:error, String.t()}
  defp validate_email(nil), do: {:error, "email is required"}
  defp validate_email(email) when is_binary(email) do
    if email =~ ~r/@/, do: {:ok, email}, else: {:error, "invalid email"}
  end
end
```

---

## Step 216: Dialyzer

```elixir
# Dialyzer ตรวจสอบ type errors ตอน compile/analyze time

# mix.exs
defp deps do
  [{:dialyxir, "~> 1.4", only: [:dev, :test], runtime: false}]
end

# ตรวจสอบ types
# mix dialyzer

defmodule TypeUnsafe do
  @spec add(integer(), integer()) :: integer()
  def add(a, b), do: a + b
  
  def bad_call do
    # Dialyzer จะ warn เพราะ passing string to add/2
    add("hello", "world")  # ❌ type error
  end
end

defmodule BetterTyped do
  @spec process(binary() | integer()) :: String.t()
  def process(value) when is_binary(value), do: value
  def process(value) when is_integer(value), do: Integer.to_string(value)
end
```

---

## Step 217: ExDoc Configuration

```elixir
# mix.exs
def project do
  [
    name: "My Library",
    source_url: "https://github.com/myorg/my_library",
    homepage_url: "https://mylib.io",
    docs: [
      main: "MyLibrary",
      logo: "priv/logo.png",
      extras: ["README.md", "CHANGELOG.md", "guides/introduction.md"],
      groups_extras: [
        "Guides": ["guides/introduction.md"]
      ],
      groups_for_modules: [
        "Core": [MyLibrary, MyLibrary.Core],
        "Adapters": [MyLibrary.Adapters.HTTP, MyLibrary.Adapters.DB],
        "Internals": [~r/MyLibrary\.Internal\..*/]
      ],
      source_ref: "v#{@version}",
      source_url_pattern: "https://github.com/myorg/my_library/blob/%{ref}/%{path}#L%{line}"
    ]
  ]
end

defp deps do
  [{:ex_doc, "~> 0.31", only: :dev, runtime: false}]
end
```

---

## Step 218: @callback Documentation

```elixir
defmodule PaymentGateway do
  @moduledoc """
  Behaviour for payment gateway adapters.
  
  ## Available Adapters
  
  - `PaymentGateway.Stripe` - Stripe payment processing
  - `PaymentGateway.PayPal` - PayPal integration
  - `PaymentGateway.Mock` - Testing mock
  
  ## Implementing a Custom Adapter
  
      defmodule MyGateway do
        @behaviour PaymentGateway
        
        @impl PaymentGateway
        def charge(amount, currency, source) do
          # Your implementation
          {:ok, %{transaction_id: "txn_123"}}
        end
      end
  """
  
  @typedoc "Amount in cents"
  @type amount :: pos_integer()
  
  @typedoc "ISO 4217 currency code"
  @type currency :: String.t()
  
  @typedoc "Payment transaction record"
  @type transaction :: %{
    required(:transaction_id) => String.t(),
    required(:amount)         => amount(),
    required(:currency)       => currency(),
    required(:status)         => :pending | :completed | :failed
  }
  
  @doc """
  Charge a customer.
  
  ## Parameters
  
  - `amount` - Amount in cents (e.g., 1000 = $10.00)
  - `currency` - ISO currency code (e.g., "USD")
  - `source` - Payment source token
  
  ## Returns
  
  `{:ok, transaction}` on success, `{:error, reason}` on failure.
  """
  @callback charge(amount(), currency(), source :: String.t()) ::
    {:ok, transaction()} | {:error, term()}
  
  @doc "Refund a previous transaction"
  @callback refund(transaction_id :: String.t(), amount :: amount()) ::
    {:ok, transaction()} | {:error, term()}
  
  @doc "Check transaction status"
  @callback status(transaction_id :: String.t()) ::
    {:ok, :pending | :completed | :failed} | {:error, term()}
  
  @optional_callbacks [status: 1]
end
```

---

## Step 219: Livebook Integration

```elixir
# Livebook = Jupyter-like notebooks สำหรับ Elixir
# ใช้ Mix.install ใน Livebook

Mix.install([
  {:req, "~> 0.4"},
  {:jason, "~> 1.4"},
  {:kino, "~> 0.12"}
])

# Interactive widgets
input = Kino.Input.text("Your name")
name = Kino.Input.read(input)
IO.puts("Hello, #{name}!")

# Data visualization
data = [
  %{month: "Jan", sales: 100},
  %{month: "Feb", sales: 150},
  %{month: "Mar", sales: 200}
]

Kino.DataTable.new(data)

# Charts
chart = VegaLite.new()
  |> VegaLite.data_from_values(data)
  |> VegaLite.mark(:bar)
  |> VegaLite.encode_field(:x, "month", type: :ordinal)
  |> VegaLite.encode_field(:y, "sales", type: :quantitative)

Kino.VegaLite.new(chart)
```

---

## Step 220: Complete Documentation Example

```elixir
defmodule DataStore do
  @moduledoc """
  A high-performance in-memory data store with TTL support.
  
  ## Features
  
  - O(1) reads and writes
  - Automatic TTL expiration
  - Thread-safe concurrent reads
  - Statistics tracking
  
  ## Quick Start
  
      # Start the store
      {:ok, _pid} = DataStore.start_link()
      
      # Store a value with 1 minute TTL
      DataStore.put("key", "value", ttl: 60_000)
      
      # Retrieve
      {:ok, "value"} = DataStore.get("key")
      
      # Delete
      :ok = DataStore.delete("key")
  
  ## Configuration
  
  | Option | Default | Description |
  |--------|---------|-------------|
  | `:cleanup_interval` | 30_000 | Cleanup interval in ms |
  | `:max_size` | 10_000 | Maximum number of entries |
  
  """
  
  use GenServer
  
  @typedoc "Store key"
  @type key :: term()
  
  @typedoc "Store value"
  @type value :: term()
  
  @typedoc "Time-to-live in milliseconds"
  @type ttl :: pos_integer()
  
  @typedoc "Store statistics"
  @type stats :: %{
    size:      non_neg_integer(),
    hits:      non_neg_integer(),
    misses:    non_neg_integer(),
    evictions: non_neg_integer()
  }
  
  @default_ttl 60_000
  @cleanup_interval 30_000
  
  # Client API
  
  @doc """
  Start the data store.
  
  ## Options
  
  - `:name` - Register with a name (default: `#{__MODULE__}`)
  - `:cleanup_interval` - How often to clean expired entries (ms)
  
  ## Examples
  
      iex> {:ok, pid} = DataStore.start_link()
      iex> is_pid(pid)
      true
  """
  @spec start_link(keyword()) :: GenServer.on_start()
  def start_link(opts \\ []) do
    name = Keyword.get(opts, :name, __MODULE__)
    GenServer.start_link(__MODULE__, opts, name: name)
  end
  
  @doc """
  Store a value with optional TTL.
  
  ## Examples
  
      iex> DataStore.put("key", "value")
      :ok
      
      iex> DataStore.put("temp_key", "value", ttl: 1000)
      :ok
  """
  @spec put(key(), value(), keyword()) :: :ok
  def put(key, value, opts \\ []) do
    ttl = Keyword.get(opts, :ttl, @default_ttl)
    GenServer.cast(__MODULE__, {:put, key, value, ttl})
  end
  
  @doc """
  Retrieve a value by key.
  
  Returns `{:ok, value}` if found and not expired, `:miss` otherwise.
  
  ## Examples
  
      iex> DataStore.put("key", "hello")
      :ok
      iex> DataStore.get("key")
      {:ok, "hello"}
      
      iex> DataStore.get("nonexistent")
      :miss
  """
  @spec get(key()) :: {:ok, value()} | :miss
  def get(key) do
    GenServer.call(__MODULE__, {:get, key})
  end
  
  @doc """
  Delete a key from the store.
  
  ## Examples
  
      iex> DataStore.put("key", "value")
      :ok
      iex> DataStore.delete("key")
      :ok
      iex> DataStore.get("key")
      :miss
  """
  @spec delete(key()) :: :ok
  def delete(key) do
    GenServer.cast(__MODULE__, {:delete, key})
  end
  
  @doc """
  Get store statistics.
  
  ## Examples
  
      iex> DataStore.stats()
      %{size: 0, hits: 0, misses: 0, evictions: 0}
  """
  @spec stats() :: stats()
  def stats do
    GenServer.call(__MODULE__, :stats)
  end
  
  # Server Callbacks
  
  @impl GenServer
  def init(opts) do
    interval = Keyword.get(opts, :cleanup_interval, @cleanup_interval)
    :timer.send_interval(interval, self(), :cleanup)
    
    {:ok, %{
      entries: %{},
      hits: 0,
      misses: 0,
      evictions: 0
    }}
  end
  
  @impl GenServer
  def handle_cast({:put, key, value, ttl}, state) do
    expires_at = System.monotonic_time(:millisecond) + ttl
    {:noreply, put_in(state, [:entries, key], {value, expires_at})}
  end
  
  @impl GenServer
  def handle_cast({:delete, key}, state) do
    {:noreply, update_in(state.entries, &Map.delete(&1, key))}
  end
  
  @impl GenServer
  def handle_call({:get, key}, _from, state) do
    now = System.monotonic_time(:millisecond)
    
    case Map.get(state.entries, key) do
      nil ->
        {:reply, :miss, %{state | misses: state.misses + 1}}
      
      {_value, expires_at} when expires_at < now ->
        new_state = %{state |
          entries: Map.delete(state.entries, key),
          misses: state.misses + 1,
          evictions: state.evictions + 1
        }
        {:reply, :miss, new_state}
      
      {value, _expires_at} ->
        {:reply, {:ok, value}, %{state | hits: state.hits + 1}}
    end
  end
  
  @impl GenServer
  def handle_call(:stats, _from, state) do
    stats = %{
      size:      map_size(state.entries),
      hits:      state.hits,
      misses:    state.misses,
      evictions: state.evictions
    }
    {:reply, stats, state}
  end
  
  @impl GenServer
  def handle_info(:cleanup, state) do
    now = System.monotonic_time(:millisecond)
    
    {valid, expired} = Enum.split_with(state.entries, fn {_, {_, expires_at}} ->
      expires_at > now
    end)
    
    {:noreply, %{state |
      entries: Map.new(valid),
      evictions: state.evictions + length(expired)
    }}
  end
end
```

---

## สรุป Part 20

✅ **Step 211** - @moduledoc และ @doc  
✅ **Step 212** - @doc false  
✅ **Step 213** - Doctests  
✅ **Step 214** - @spec  
✅ **Step 215** - @type  
✅ **Step 216** - Dialyzer  
✅ **Step 217** - ExDoc configuration  
✅ **Step 218** - @callback documentation  
✅ **Step 219** - Livebook  
✅ **Step 220** - Complete documented module  

---

## จบระดับ Intermediate (Steps 101-220)

ในระดับนี้คุณได้เรียนรู้:
- **OTP**: GenServer, Supervisor, Agent, Task
- **Storage**: ETS, Mnesia
- **Project**: Mix, Config, Releases
- **Testing**: ExUnit, Mox, Property testing
- **Design**: Protocols, Behaviours
- **Advanced**: Macros, Metaprogramming
- **Production**: Error handling, Documentation, Typespecs

---

➡️ [Part 21: Phoenix Framework - Introduction](./part-21-phoenix-intro.md)
