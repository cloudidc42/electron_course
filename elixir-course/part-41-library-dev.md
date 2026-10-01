# Part 41: Open Source Library Development (Steps 451-470)

## Step 451: Library Design Principles

```
Elixir library design:
1. Follow conventions (mix, hex, docs)
2. Clear public API (minimal surface)
3. Configurable but opinionated defaults
4. Supervision tree (if GenServer needed)
5. Behaviours for extensibility
6. Telemetry for observability
7. Thorough documentation + doctests
8. Property-based tests

Naming: snake_case for functions/modules
Modules: MyLib.Core, MyLib.Server, MyLib.Client
```

---

## Step 452: Library Structure

```
my_library/
├── lib/
│   ├── my_library.ex          ← main module + public API
│   ├── my_library/
│   │   ├── application.ex     ← supervision tree
│   │   ├── config.ex          ← config struct
│   │   ├── server.ex          ← GenServer (if needed)
│   │   └── utils.ex           ← internal helpers
├── test/
│   ├── my_library_test.exs
│   └── support/
├── mix.exs
├── README.md
└── CHANGELOG.md
```

```elixir
# mix.exs
defmodule MyLibrary.MixProject do
  use Mix.Project
  
  @version "1.0.0"
  @source_url "https://github.com/me/my_library"
  
  def project do
    [
      app:             :my_library,
      version:         @version,
      elixir:          "~> 1.14",
      elixirc_paths:   elixirc_paths(Mix.env()),
      start_permanent: Mix.env() == :prod,
      deps:            deps(),
      description:     "A useful Elixir library",
      package:         package(),
      docs:            docs(),
      test_coverage:   [tool: ExCoveralls],
      preferred_cli_env: ["coveralls": :test]
    ]
  end
  
  def application do
    [extra_applications: [:logger], mod: {MyLibrary.Application, []}]
  end
  
  defp deps do
    [
      # Core
      {:telemetry, "~> 1.2"},
      # Dev/test only
      {:ex_doc,         "~> 0.31", only: :dev,  runtime: false},
      {:credo,          "~> 1.7",  only: :dev,  runtime: false},
      {:dialyxir,       "~> 1.4",  only: :dev,  runtime: false},
      {:excoveralls,    "~> 0.18", only: :test, runtime: false},
      {:stream_data,    "~> 0.6",  only: :test}
    ]
  end
  
  defp package do
    [
      name:        "my_library",
      files:       ~w(lib mix.exs README.md CHANGELOG.md LICENSE),
      licenses:    ["Apache-2.0"],
      links:       %{"GitHub" => @source_url}
    ]
  end
  
  defp docs do
    [
      main:            "MyLibrary",
      source_url:      @source_url,
      source_ref:      "v#{@version}",
      extras:          ["README.md", "CHANGELOG.md"],
      groups_for_modules: [
        Core: [MyLibrary, MyLibrary.Config],
        Internal: [~r/MyLibrary\..+/]
      ]
    ]
  end
end
```

---

## Step 453: Public API Design

```elixir
defmodule MyLibrary do
  @moduledoc """
  A library for doing useful things.

  ## Installation

      def deps do
        [{:my_library, "~> 1.0"}]
      end

  ## Usage

      {:ok, result} = MyLibrary.do_thing("input")

  ## Configuration

  You can configure the library in `config.exs`:

      config :my_library,
        timeout: 5_000,
        max_retries: 3
  """
  
  alias MyLibrary.{Config, Server}
  
  @doc """
  Do the main thing.

  ## Examples

      iex> MyLibrary.do_thing("hello")
      {:ok, "HELLO"}

      iex> MyLibrary.do_thing("")
      {:error, :empty_input}

  """
  @spec do_thing(String.t(), keyword()) :: {:ok, String.t()} | {:error, atom()}
  def do_thing(input, opts \\ []) do
    with {:ok, validated} <- validate(input),
         config            <- Config.build(opts),
         {:ok, result}     <- Server.process(validated, config) do
      {:ok, result}
    end
  end
  
  @doc """
  Do thing, raising on error.
  """
  @spec do_thing!(String.t(), keyword()) :: String.t()
  def do_thing!(input, opts \\ []) do
    case do_thing(input, opts) do
      {:ok, result}    -> result
      {:error, reason} -> raise MyLibrary.Error, reason: reason
    end
  end
  
  defp validate(""), do: {:error, :empty_input}
  defp validate(s) when is_binary(s), do: {:ok, s}
  defp validate(_), do: {:error, :invalid_input}
end
```

---

## Step 454: Configuration

```elixir
defmodule MyLibrary.Config do
  @moduledoc "Configuration for MyLibrary"
  
  defstruct [
    timeout:      5_000,
    max_retries:  3,
    adapter:      MyLibrary.DefaultAdapter,
    telemetry:    true
  ]
  
  @type t :: %__MODULE__{
    timeout:     pos_integer(),
    max_retries: non_neg_integer(),
    adapter:     module(),
    telemetry:   boolean()
  }
  
  def build(opts \\ []) do
    app_config = Application.get_env(:my_library, :config, [])
    all_opts   = Keyword.merge(app_config, opts)
    
    %__MODULE__{
      timeout:     Keyword.get(all_opts, :timeout, 5_000),
      max_retries: Keyword.get(all_opts, :max_retries, 3),
      adapter:     Keyword.get(all_opts, :adapter, MyLibrary.DefaultAdapter),
      telemetry:   Keyword.get(all_opts, :telemetry, true)
    }
  end
  
  def validate!(%__MODULE__{} = config) do
    cond do
      config.timeout <= 0  -> raise ArgumentError, "timeout must be positive"
      config.max_retries < 0 -> raise ArgumentError, "max_retries cannot be negative"
      not is_atom(config.adapter) -> raise ArgumentError, "adapter must be a module"
      true -> config
    end
  end
end
```

---

## Step 455: Behaviours

```elixir
defmodule MyLibrary.Adapter do
  @moduledoc """
  Behaviour for MyLibrary adapters.

  Implement this to create a custom adapter:

      defmodule MyCustomAdapter do
        @behaviour MyLibrary.Adapter

        @impl MyLibrary.Adapter
        def process(input, opts) do
          # your implementation
        end
      end
  """
  
  @doc """
  Process the input and return a result.
  """
  @callback process(input :: String.t(), opts :: keyword()) ::
    {:ok, String.t()} | {:error, term()}
  
  @doc """
  Initialize the adapter. Called once at startup.
  Optional callback.
  """
  @callback init(config :: MyLibrary.Config.t()) :: :ok | {:error, term()}
  
  @optional_callbacks [init: 1]
end

defmodule MyLibrary.DefaultAdapter do
  @behaviour MyLibrary.Adapter
  
  @impl MyLibrary.Adapter
  def process(input, _opts) do
    {:ok, String.upcase(input)}
  end
end
```

---

## Step 456: Application (Supervision Tree)

```elixir
defmodule MyLibrary.Application do
  use Application
  
  def start(_type, _args) do
    config = MyLibrary.Config.build()
    
    children = [
      {MyLibrary.Server, config}
    ]
    
    opts = [strategy: :one_for_one, name: MyLibrary.Supervisor]
    Supervisor.start_link(children, opts)
  end
end

defmodule MyLibrary.Server do
  use GenServer
  
  def start_link(config) do
    GenServer.start_link(__MODULE__, config, name: __MODULE__)
  end
  
  def process(input, config) do
    GenServer.call(__MODULE__, {:process, input, config}, config.timeout)
  end
  
  def init(config) do
    # Initialize adapter if it has init/1
    if function_exported?(config.adapter, :init, 1) do
      config.adapter.init(config)
    end
    
    {:ok, %{config: config, stats: %{processed: 0, errors: 0}}}
  end
  
  def handle_call({:process, input, config}, _from, state) do
    start = System.monotonic_time()
    
    result = do_process_with_retry(input, config.adapter, config.max_retries)
    
    duration = System.monotonic_time() - start
    emit_telemetry(result, duration, config)
    
    new_stats = update_stats(state.stats, result)
    {:reply, result, %{state | stats: new_stats}}
  end
  
  defp do_process_with_retry(input, adapter, retries) do
    case adapter.process(input, []) do
      {:ok, _} = ok -> ok
      {:error, _} when retries > 0 -> do_process_with_retry(input, adapter, retries - 1)
      error -> error
    end
  end
  
  defp emit_telemetry(result, duration, %{telemetry: true}) do
    status = if match?({:ok, _}, result), do: :ok, else: :error
    :telemetry.execute([:my_library, :process], %{duration: duration}, %{status: status})
  end
  defp emit_telemetry(_result, _duration, _config), do: :ok
  
  defp update_stats(stats, {:ok, _}), do: Map.update!(stats, :processed, &(&1 + 1))
  defp update_stats(stats, {:error, _}), do: Map.update!(stats, :errors, &(&1 + 1))
end
```

---

## Step 457: Documentation

```elixir
defmodule MyLibrary do
  @moduledoc """
  Main documentation here.

  See `MyLibrary.Config` for configuration options.
  """
  
  @doc """
  Process input.

  ## Parameters

    * `input` - The string to process (must be non-empty)
    * `opts` - Optional keyword list:
      * `:timeout` - timeout in milliseconds (default: 5000)
      * `:adapter` - custom adapter module

  ## Return Values

    * `{:ok, result}` - success
    * `{:error, :empty_input}` - empty string given
    * `{:error, :timeout}` - operation timed out

  ## Examples

      iex> MyLibrary.process("hello")
      {:ok, "HELLO"}

      iex> MyLibrary.process("")
      {:error, :empty_input}

      iex> MyLibrary.process("hello", timeout: 1)
      {:error, :timeout}

  """
  def process(input, opts \\ []) do
    # ...
  end
end

# Generate docs
# mix docs
# opens doc/index.html

# Publish to hex.pm
# mix hex.publish
```

---

## Step 458: Versioning (Semantic Versioning)

```
SemVer: MAJOR.MINOR.PATCH

- PATCH: bug fix, no API change
- MINOR: new features, backwards compatible
- MAJOR: breaking changes

CHANGELOG.md:
# Changelog

## [Unreleased]

## [1.1.0] - 2024-01-15
### Added
- New `process_batch/2` function

### Changed
- `process/2` now accepts atoms as input

## [1.0.0] - 2024-01-01
### Initial release

[Unreleased]: https://github.com/me/my_lib/compare/v1.1.0...HEAD
[1.1.0]: https://github.com/me/my_lib/compare/v1.0.0...v1.1.0
```

```bash
# Bump version
# mix hex.version patch/minor/major

# Check for breaking changes
mix dialyzer

# Tag release
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0

# Publish
mix hex.publish
```

---

## Step 459: Testing Library

```elixir
defmodule MyLibraryTest do
  use ExUnit.Case, async: true
  use ExUnitProperties
  
  doctest MyLibrary
  
  describe "process/2" do
    test "uppercases input" do
      assert {:ok, "HELLO"} = MyLibrary.process("hello")
    end
    
    test "returns error for empty input" do
      assert {:error, :empty_input} = MyLibrary.process("")
    end
    
    test "respects custom adapter" do
      defmodule TestAdapter do
        @behaviour MyLibrary.Adapter
        def process(input, _opts), do: {:ok, "custom:#{input}"}
      end
      
      assert {:ok, "custom:hello"} = MyLibrary.process("hello", adapter: TestAdapter)
    end
    
    property "never crashes on string input" do
      check all input <- string(:utf8) do
        result = MyLibrary.process(input)
        assert match?({:ok, _}, result) or match?({:error, _}, result)
      end
    end
  end
  
  describe "process!/2" do
    test "raises on error" do
      assert_raise MyLibrary.Error, fn ->
        MyLibrary.process!("")
      end
    end
  end
end
```

---

## Step 460: Publishing to Hex.pm

```bash
# Register on hex.pm
mix hex.user register

# Add metadata to mix.exs (done in step 452)

# Check for issues
mix hex.build

# Publish
mix hex.publish

# Publish docs separately
mix hex.publish docs

# Update package
# bump version in mix.exs
# update CHANGELOG.md
mix hex.publish

# Retire old version (if security issue)
mix hex.retire my_library 0.1.0 security "XSS vulnerability"
```

---

## สรุป Part 41

✅ **Step 451** - Library design principles  
✅ **Step 452** - Library structure + mix.exs  
✅ **Step 453** - Public API design  
✅ **Step 454** - Configuration  
✅ **Step 455** - Behaviours  
✅ **Step 456** - Application/supervision  
✅ **Step 457** - Documentation  
✅ **Step 458** - Semantic versioning  
✅ **Step 459** - Testing library  
✅ **Step 460** - Publishing to Hex.pm  

➡️ [Part 42: Advanced Macros](./part-42-macros.md)
