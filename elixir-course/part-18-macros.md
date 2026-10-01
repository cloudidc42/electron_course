# Part 18: Metaprogramming และ Macros (Steps 191-210)

## Step 191: Metaprogramming คืออะไร?

```
Metaprogramming = เขียน code ที่สร้าง code
- Elixir compile ผ่าน AST (Abstract Syntax Tree)
- Macros ทำงานตอน compile time
- ขยาย syntax ของภาษาได้
- Elixir ส่วนใหญ่เขียนด้วย macros เอง (if, unless, defstruct, etc.)
```

### 191.1 AST คืออะไร?

```elixir
# quote/1 แปลง expression เป็น AST
quote do: 1 + 2
# {:+, [context: Elixir, imports: [...]], [1, 2]}

quote do: if(true, do: "yes", else: "no")
# {:if, [...], [true, [do: "yes", else: "no"]]}

quote do: x = 1 + 2
# {:=, [...], [{:x, [...], Elixir}, {:+, [...], [1, 2]}]}

# unquote/1 แทรก value เข้าใน AST
x = 42
quote do: unquote(x) + 1
# {:+, [...], [42, 1]}

# Inspect AST
Macro.to_string(quote do: 1 + 2 * 3)
# "1 + 2 * 3"
```

---

## Step 192: Simple Macros

```elixir
defmodule MyMacros do
  # Macro ต้องใช้ defmacro
  defmacro say_hello(name) do
    quote do
      IO.puts("Hello, #{unquote(name)}!")
    end
  end
  
  # Macro ที่สร้าง function
  defmacro def_constant(name, value) do
    quote do
      def unquote(name)(), do: unquote(value)
    end
  end
  
  # unless (เหมือน Elixir built-in)
  defmacro my_unless(condition, do: body) do
    quote do
      if !unquote(condition) do
        unquote(body)
      end
    end
  end
end

defmodule MyModule do
  require MyMacros
  import MyMacros
  
  def_constant :pi, 3.14159
  def_constant :e,  2.71828
  
  def demo do
    say_hello("World")
    my_unless false, do: IO.puts("This prints!")
    
    IO.puts("pi = #{pi()}")
    IO.puts("e = #{e()}")
  end
end

MyModule.demo()
# Hello, World!
# This prints!
# pi = 3.14159
# e = 2.71828
```

---

## Step 193: use Macro

```elixir
defmodule Printable do
  defmacro __using__(_opts) do
    quote do
      def print do
        IO.puts("I am #{__MODULE__}")
      end
      
      def debug do
        IO.inspect(__MODULE__, label: "Module")
      end
    end
  end
end

defmodule MyStruct do
  use Printable
  
  defstruct [:name, :value]
end

MyStruct.print()   # I am MyStruct
MyStruct.debug()   # Module: MyStruct
```

---

## Step 194: Module Attributes กับ Macros

```elixir
defmodule RouteBuilder do
  @routes []
  
  defmacro __using__(_) do
    quote do
      import RouteBuilder
      Module.register_attribute(__MODULE__, :routes, accumulate: true)
      @before_compile RouteBuilder
    end
  end
  
  defmacro get(path, handler) do
    quote do
      @routes {:get, unquote(path), unquote(handler)}
    end
  end
  
  defmacro post(path, handler) do
    quote do
      @routes {:post, unquote(path), unquote(handler)}
    end
  end
  
  defmacro __before_compile__(env) do
    routes = Module.get_attribute(env.module, :routes)
    
    route_functions = Enum.map(routes, fn {method, path, handler} ->
      quote do
        def match(unquote(method), unquote(path)), do: unquote(handler)
      end
    end)
    
    quote do
      unquote(route_functions)
      def match(_, _), do: {:error, :not_found}
      
      def routes, do: unquote(routes)
    end
  end
end

defmodule MyRouter do
  use RouteBuilder
  
  get  "/users",      {UserController, :index}
  get  "/users/:id",  {UserController, :show}
  post "/users",      {UserController, :create}
end

MyRouter.match(:get, "/users")      # {UserController, :index}
MyRouter.match(:post, "/users")     # {UserController, :create}
MyRouter.match(:get, "/missing")    # {:error, :not_found}
MyRouter.routes()                   # [{:get, "/users", ...}, ...]
```

---

## Step 195: Hygiene ใน Macros

```elixir
defmodule HygieneMacros do
  # Hygienic macro - ตัวแปรใน macro ไม่ clash กับ caller
  defmacro safe_compute(expr) do
    quote do
      result = unquote(expr)  # result ใน macro scope
      result * 2
    end
  end
  
  # Non-hygienic - ใช้ var! เพื่อ access caller's scope
  defmacro inject_var(name, value) do
    quote do
      var!(unquote(name)) = unquote(value)
    end
  end
  
  # Bind quote - explicitly bind to specific module
  defmacro with_logging(do: block) do
    quote do
      require Logger
      Logger.debug("Starting computation")
      result = unquote(block)
      Logger.debug("Done: #{inspect(result)}")
      result
    end
  end
end

import HygieneMacros

result = 10
safe_compute(result + 5)  # (10 + 5) * 2 = 30
# result ใน caller ยังเป็น 10 (ไม่ถูก overwrite)

inject_var(x, 42)
IO.puts(x)  # 42 - ใช้ caller's variable
```

---

## Step 196: DSL ด้วย Macros

```elixir
defmodule StateMachine do
  defmacro __using__(_) do
    quote do
      import StateMachine
      Module.register_attribute(__MODULE__, :states, accumulate: true)
      Module.register_attribute(__MODULE__, :transitions, accumulate: true)
      @before_compile StateMachine
    end
  end
  
  defmacro state(name) do
    quote do
      @states unquote(name)
    end
  end
  
  defmacro transition(from, event, to, guard \\ nil) do
    quote do
      @transitions {unquote(from), unquote(event), unquote(to), unquote(guard)}
    end
  end
  
  defmacro __before_compile__(env) do
    states = Module.get_attribute(env.module, :states)
    transitions = Module.get_attribute(env.module, :transitions)
    
    transition_fns = Enum.map(transitions, fn {from, event, to, guard} ->
      guard_check = if guard do
        quote do: unquote(guard).(context)
      else
        quote do: true
      end
      
      quote do
        def transition(%{state: unquote(from)} = ctx, unquote(event)) do
          if unquote(guard_check) do
            {:ok, %{ctx | state: unquote(to)}}
          else
            {:error, :guard_failed}
          end
        end
      end
    end)
    
    quote do
      unquote(transition_fns)
      def transition(_ctx, _event), do: {:error, :invalid_transition}
      
      def valid_states, do: unquote(states)
    end
  end
end

# สร้าง Order state machine
defmodule OrderFSM do
  use StateMachine
  
  state :pending
  state :confirmed
  state :shipped
  state :delivered
  state :cancelled
  
  transition :pending,   :confirm,   :confirmed
  transition :confirmed, :ship,      :shipped
  transition :shipped,   :deliver,   :delivered
  transition :pending,   :cancel,    :cancelled
  transition :confirmed, :cancel,    :cancelled
end

# Usage
order = %{state: :pending, id: 1}

{:ok, order} = OrderFSM.transition(order, :confirm)
order.state  # :confirmed

{:ok, order} = OrderFSM.transition(order, :ship)
order.state  # :shipped

OrderFSM.transition(order, :confirm)  # {:error, :invalid_transition}
```

---

## Step 197: Macros สำหรับ Testing DSL

```elixir
defmodule TestDSL do
  defmacro describe(description, do: block) do
    quote do
      IO.puts("\n  #{unquote(description)}")
      unquote(block)
    end
  end
  
  defmacro it(description, do: block) do
    quote do
      result = try do
        unquote(block)
        :passed
      rescue
        e -> {:failed, Exception.message(e)}
      end
      
      case result do
        :passed          -> IO.puts("    ✓ #{unquote(description)}")
        {:failed, msg}   -> IO.puts("    ✗ #{unquote(description)}: #{msg}")
      end
    end
  end
  
  defmacro expect(actual, :to_equal, expected) do
    quote do
      actual = unquote(actual)
      expected = unquote(expected)
      
      unless actual == expected do
        raise "Expected #{inspect(actual)} to equal #{inspect(expected)}"
      end
    end
  end
end

# Usage
import TestDSL

describe "Math operations" do
  it "adds numbers" do
    expect 1 + 1, :to_equal, 2
  end
  
  it "multiplies numbers" do
    expect 3 * 4, :to_equal, 12
  end
end
```

---

## Step 198: Code Generation กับ Macros

```elixir
defmodule CRUD do
  defmacro __using__(opts) do
    schema = Keyword.fetch!(opts, :schema)
    repo   = Keyword.get(opts, :repo, MyApp.Repo)
    
    quote do
      alias unquote(schema)
      
      def list do
        unquote(repo).all(unquote(schema))
      end
      
      def get(id) do
        case unquote(repo).get(unquote(schema), id) do
          nil    -> {:error, :not_found}
          record -> {:ok, record}
        end
      end
      
      def get!(id) do
        unquote(repo).get!(unquote(schema), id)
      end
      
      def create(attrs) do
        unquote(schema).changeset(struct(unquote(schema)), attrs)
        |> unquote(repo).insert()
      end
      
      def update(record, attrs) do
        unquote(schema).changeset(record, attrs)
        |> unquote(repo).update()
      end
      
      def delete(record) do
        unquote(repo).delete(record)
      end
      
      defoverridable [list: 0, get: 1, create: 1, update: 2, delete: 1]
    end
  end
end

defmodule UserContext do
  use CRUD, schema: User, repo: MyApp.Repo
  
  # Override list to add preload
  def list do
    User
    |> MyApp.Repo.all()
    |> MyApp.Repo.preload(:posts)
  end
  
  # Add custom functions
  def find_by_email(email) do
    case MyApp.Repo.get_by(User, email: email) do
      nil  -> {:error, :not_found}
      user -> {:ok, user}
    end
  end
end
```

---

## Step 199: Macro Hygiene Advanced

```elixir
defmodule SafeMacros do
  # ใช้ Macro.unique_var สำหรับ fresh variable names
  defmacro memoize(do: block) do
    cache_var = Macro.unique_var(:cache, __MODULE__)
    
    quote do
      unquote(cache_var) = case Process.get(:memoize_cache, %{}) do
        cache -> cache
      end
      
      key = unquote(block) |> :erlang.phash2()
      
      case Map.get(unquote(cache_var), key) do
        nil ->
          result = unquote(block)
          Process.put(:memoize_cache, Map.put(unquote(cache_var), key, result))
          result
        cached ->
          cached
      end
    end
  end
  
  # Macro.expand - ดู AST หลัง expand
  def show_expansion(ast) do
    ast
    |> Macro.expand(__ENV__)
    |> Macro.to_string()
    |> IO.puts()
  end
end
```

---

## Step 200: Complete DSL Example - Config Builder

```elixir
defmodule ConfigBuilder do
  defmacro __using__(_opts) do
    quote do
      import ConfigBuilder
      Module.register_attribute(__MODULE__, :config_fields, accumulate: true)
      @before_compile ConfigBuilder
    end
  end
  
  defmacro field(name, type, opts \\ []) do
    quote do
      @config_fields {unquote(name), unquote(type), unquote(opts)}
    end
  end
  
  defmacro __before_compile__(env) do
    fields = Module.get_attribute(env.module, :config_fields)
    
    struct_fields = Enum.map(fields, fn {name, _type, opts} ->
      default = Keyword.get(opts, :default)
      {name, default}
    end)
    
    validations = Enum.map(fields, fn {name, type, opts} ->
      required = Keyword.get(opts, :required, false)
      
      quote do
        defp validate_field(config, unquote(name), unquote(type), unquote(required)) do
          value = Map.get(config, unquote(name))
          
          cond do
            unquote(required) && value == nil ->
              {:error, "#{unquote(name)} is required"}
            
            value != nil && !valid_type?(value, unquote(type)) ->
              {:error, "#{unquote(name)} must be #{unquote(type)}"}
            
            true -> :ok
          end
        end
      end
    end)
    
    field_names = Enum.map(fields, fn {name, type, opts} ->
      {name, type, opts}
    end)
    
    quote do
      defstruct unquote(struct_fields)
      
      unquote(validations)
      
      def new(attrs \\ %{}) do
        config = struct(__MODULE__, attrs)
        
        errors = unquote(field_names)
          |> Enum.map(fn {name, type, opts} ->
               required = Keyword.get(opts, :required, false)
               validate_field(config, name, type, required)
             end)
          |> Enum.filter(&(&1 != :ok))
          |> Enum.map(fn {:error, msg} -> msg end)
        
        if errors == [] do
          {:ok, config}
        else
          {:error, errors}
        end
      end
      
      defp valid_type?(val, :string), do: is_binary(val)
      defp valid_type?(val, :integer), do: is_integer(val)
      defp valid_type?(val, :boolean), do: is_boolean(val)
      defp valid_type?(val, :list), do: is_list(val)
      defp valid_type?(val, :map), do: is_map(val)
      defp valid_type?(_val, _type), do: true
    end
  end
end

# สร้าง Config struct ด้วย DSL
defmodule DatabaseConfig do
  use ConfigBuilder
  
  field :host,     :string,  required: true, default: "localhost"
  field :port,     :integer, required: true, default: 5432
  field :database, :string,  required: true
  field :username, :string,  required: true
  field :password, :string,  required: true
  field :pool_size, :integer, default: 10
  field :ssl,      :boolean,  default: false
end

# Usage
DatabaseConfig.new(%{
  database: "myapp_prod",
  username: "admin",
  password: "secret"
})
# {:ok, %DatabaseConfig{host: "localhost", port: 5432, database: "myapp_prod", ...}}

DatabaseConfig.new(%{})
# {:error, ["database is required", "username is required", "password is required"]}
```

---

## สรุป Part 18

✅ **Step 191** - AST และ quote/unquote  
✅ **Step 192** - Simple macros  
✅ **Step 193** - use macro  
✅ **Step 194** - Module attributes + @before_compile  
✅ **Step 195** - Hygiene  
✅ **Step 196** - DSL สำหรับ State Machine  
✅ **Step 197** - Testing DSL  
✅ **Step 198** - Code generation ด้วย CRUD macro  
✅ **Step 199** - Advanced hygiene  
✅ **Step 200** - Config Builder DSL  

➡️ [Part 19: Error Handling](./part-19-error-handling.md)
