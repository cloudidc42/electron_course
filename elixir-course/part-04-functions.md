# Part 04: Functions และ Modules (Steps 31-40)

## Step 31: Named Functions และ Modules

### 31.1 defmodule และ def

```elixir
# ทุก function ต้องอยู่ใน Module
defmodule Calculator do
  # def = public function
  def add(a, b) do
    a + b
  end
  
  # defp = private function (เรียกได้แค่ภายใน module)
  defp validate(x) when is_number(x), do: {:ok, x}
  defp validate(_), do: {:error, :not_a_number}
  
  def safe_add(a, b) do
    with {:ok, a} <- validate(a),
         {:ok, b} <- validate(b) do
      {:ok, a + b}
    end
  end
end

Calculator.add(1, 2)         # 3
Calculator.safe_add(1, 2)    # {:ok, 3}
Calculator.safe_add(1, "a")  # {:error, :not_a_number}
# Calculator.validate(5)     # ERROR! private function
```

### 31.2 Shorthand Syntax ด้วย do:

```elixir
defmodule Greeter do
  # แบบยาว
  def greet(name) do
    "Hello, #{name}!"
  end
  
  # แบบสั้น (single expression)
  def greet_short(name), do: "Hello, #{name}!"
  
  # ใช้กับ multiple clauses ได้ดี
  def describe(:admin), do: "Administrator"
  def describe(:user), do: "Regular user"
  def describe(_), do: "Unknown role"
end
```

---

## Step 32: Function Arity

Arity คือจำนวน arguments ที่ function รับ

### 32.1 Arity กำหนด Function Identity

```elixir
defmodule Example do
  # add/2 และ add/3 เป็น FUNCTION คนละตัว!
  def add(a, b) do
    a + b
  end
  
  def add(a, b, c) do
    a + b + c
  end
  
  # greet/0 และ greet/1 ก็คนละตัว
  def greet do
    "Hello!"
  end
  
  def greet(name) do
    "Hello, #{name}!"
  end
end

Example.add(1, 2)     # เรียก add/2
Example.add(1, 2, 3)  # เรียก add/3
Example.greet()       # เรียก greet/0
Example.greet("Alice") # เรียก greet/1
```

### 32.2 Default Arguments

```elixir
defmodule Server do
  # ใช้ \\ สำหรับ default value
  def connect(host, port \\ 80) do
    IO.puts("Connecting to #{host}:#{port}")
  end
  
  def create_user(name, role \\ :user, active \\ true) do
    %{name: name, role: role, active: active}
  end
end

Server.connect("localhost")       # Connecting to localhost:80
Server.connect("localhost", 8080) # Connecting to localhost:8080

Server.create_user("Alice")
# %{active: true, name: "Alice", role: :user}

Server.create_user("Bob", :admin)
# %{active: true, name: "Bob", role: :admin}

Server.create_user("Charlie", :moderator, false)
# %{active: false, name: "Charlie", role: :moderator}
```

### 32.3 Default Arguments กับ Multiple Clauses

```elixir
# WARNING: default arguments กับ multiple clauses ต้องระวัง!

defmodule Tricky do
  # ต้องแยก default clause ออก
  def greet(name, greeting \\ "Hello")
  
  def greet(name, "Hello"), do: "Hello, #{name}!"
  def greet(name, "Hi"),    do: "Hi there, #{name}!"
  def greet(name, greeting), do: "#{greeting}, #{name}!"
end

Tricky.greet("Alice")         # "Hello, Alice!"
Tricky.greet("Bob", "Hi")     # "Hi there, Bob!"
Tricky.greet("Charlie", "Hey") # "Hey, Charlie!"
```

---

## Step 33: Anonymous Functions (Lambda)

### 33.1 Anonymous Function พื้นฐาน

```elixir
# สร้างด้วย fn ... end
iex> add = fn a, b -> a + b end
iex> add.(1, 2)
3

# สังเกต: เรียก anonymous function ด้วย . (dot notation)

# Shorthand ด้วย & และ &1, &2, ...
iex> add = &(&1 + &2)
iex> add.(1, 2)
3

iex> double = &(&1 * 2)
iex> double.(5)
10

iex> greet = &("Hello, #{&1}!")
iex> greet.("World")
"Hello, World!"
```

### 33.2 & Capture Operator

```elixir
# Capture named function
iex> upcase = &String.upcase/1
iex> upcase.("hello")
"HELLO"

# ใช้ใน Enum.map
iex> Enum.map(["a", "b", "c"], &String.upcase/1)
["A", "B", "C"]

# ใช้กับ module function ตัวเอง
iex> Enum.map([1, 2, 3], &(&1 * 2))
[2, 4, 6]

# Capture function กับ multiple args
iex> divide = &div/2
iex> divide.(10, 3)
3
```

### 33.3 Closures

```elixir
# Anonymous functions ใน Elixir เป็น closure
# สามารถ capture variables จาก scope รอบนอกได้

iex> multiplier = 3
iex> multiply = fn x -> x * multiplier end
iex> multiply.(5)
15

# หรือสร้าง function factory
defmodule MathFactory do
  def make_adder(n) do
    fn x -> x + n end
  end
  
  def make_multiplier(n) do
    fn x -> x * n end
  end
end

add5    = MathFactory.make_adder(5)
triple  = MathFactory.make_multiplier(3)

add5.(10)    # 15
triple.(4)   # 12

# ใช้ closure สำหรับ memoization
defmodule Memo do
  def memoize(fun) do
    cache = %{}
    fn args ->
      case Map.get(cache, args) do
        nil ->
          result = apply(fun, args)
          # Note: ไม่สามารถ mutate cache ได้จริงๆ (immutable)
          # ต้องใช้ Process/Agent สำหรับ mutable state
          result
        result ->
          result
      end
    end
  end
end
```

---

## Step 34: The Pipe Operator |>

Pipe operator คือ feature ที่ทำให้โค้ด Elixir อ่านง่ายมาก

### 34.1 Pipe พื้นฐาน

```elixir
# โดยไม่ใช้ pipe
String.upcase(String.trim("  hello  "))  # "HELLO"

# ใช้ pipe - อ่านได้จากซ้ายไปขวา
"  hello  "
|> String.trim()
|> String.upcase()
# "HELLO"

# |> ส่ง ค่าซ้ายเป็น argument แรกของฟังก์ชันขวา
"hello" |> String.upcase()
# เหมือนกับ String.upcase("hello")
```

### 34.2 Pipe ใช้งานจริง

```elixir
# Processing pipeline
result = "  Hello, World!  "
|> String.trim()
|> String.downcase()
|> String.split(", ")
|> Enum.map(&String.capitalize/1)
|> Enum.join(", ")

# result = "Hello, World!"

# Data transformation
users = [
  %{name: "Alice", age: 30, active: true},
  %{name: "Bob", age: 25, active: false},
  %{name: "Charlie", age: 35, active: true}
]

active_users = users
|> Enum.filter(& &1.active)
|> Enum.map(& &1.name)
|> Enum.sort()

# active_users = ["Alice", "Charlie"]
```

### 34.3 Pipe กับ Anonymous Functions

```elixir
# ถ้าต้องการ argument ไม่ใช่ตัวแรก ต้องใช้ fn wrapper
result = [1, 2, 3, 4, 5]
|> Enum.filter(fn x -> x > 2 end)
|> Enum.map(fn x -> x * 2 end)
|> Enum.reduce(0, fn x, acc -> x + acc end)

# หรือใช้ & shorthand
result = [1, 2, 3, 4, 5]
|> Enum.filter(&(&1 > 2))
|> Enum.map(&(&1 * 2))
|> Enum.sum()
```

---

## Step 35: Module Attributes

Module Attributes เป็น metadata ของ module

### 35.1 Attributes พื้นฐาน

```elixir
defmodule Config do
  # Module attributes ใช้ @
  @max_retries 3
  @timeout 5000
  @app_name "MyApp"
  
  def connect(url) do
    IO.puts("Connecting to #{url}")
    IO.puts("Timeout: #{@timeout}ms")
    IO.puts("Max retries: #{@max_retries}")
  end
  
  def app_name, do: @app_name
end

Config.app_name()  # "MyApp"
```

### 35.2 Built-in Attributes

```elixir
defmodule MyModule do
  # Documentation
  @moduledoc """
  Module documentation
  """
  
  @doc """
  Function documentation
  """
  def my_function do
    :ok
  end
  
  # Type specification (Dialyzer)
  @spec add(integer(), integer()) :: integer()
  def add(a, b), do: a + b
  
  # Module-level annotations
  @author "Alice"
  @version "1.0.0"
  
  def version, do: @version
end
```

### 35.3 Compile-time vs Runtime

```elixir
defmodule EnvironmentConfig do
  # อ่านค่าจาก environment ตอน compile time
  @env Mix.env()
  
  def environment, do: @env
  
  # แต่ถ้าต้องการ runtime value ให้ใช้ function
  def runtime_env do
    System.get_env("APP_ENV", "development")
  end
end
```

---

## Step 36: import, alias, use, require

### 36.1 alias - ตั้งชื่อย่อ

```elixir
defmodule MyApp.UserController do
  # alias ทำให้เรียก MyApp.User.Account ด้วย Account แทน
  alias MyApp.User.Account
  alias MyApp.User.Profile
  
  def show(id) do
    user = Account.find(id)
    profile = Profile.load(user)
    {:ok, user, profile}
  end
end

# Alias หลาย module พร้อมกัน
alias MyApp.{User, Product, Order}

# Alias พร้อมเปลี่ยนชื่อ
alias MyApp.VeryLongModuleName, as: Short
```

### 36.2 import - นำ function เข้ามาใช้โดยตรง

```elixir
defmodule MathUtils do
  import Integer, only: [is_even: 1, is_odd: 1]
  import Enum, only: [map: 2, filter: 2, reduce: 3]
  
  def process_numbers(numbers) do
    numbers
    |> filter(&is_even/1)  # ไม่ต้องพิมพ์ Integer.is_even
    |> map(&(&1 * 2))
    |> reduce(0, &+/2)
  end
end

# import ทั้ง module
import List   # import ทุก public function ของ List

# import เฉพาะ functions
import String, only: [upcase: 1, downcase: 1]

# import ยกเว้น functions
import String, except: [to_integer: 1]
```

### 36.3 require - สำหรับ Macros

```elixir
# Macros ต้อง require ก่อนใช้
defmodule Logger do
  require Logger
  
  def log_info(message) do
    Logger.info(message)    # Logger.info เป็น macro
  end
end

# Kernel module ถูก require อัตโนมัติ
# แต่ module อื่นที่มี macros ต้อง require เอง
```

### 36.4 use - Code Injection

```elixir
# use ทำให้ module อื่น inject code เข้ามา
defmodule MyController do
  use Phoenix.Controller  # inject Phoenix controller behaviors
  
  def index(conn, _params) do
    render(conn, "index.html")  # render ถูก inject มาจาก Phoenix
  end
end

# ทำงานอย่างไร?
defmodule MyBehaviour do
  defmacro __using__(opts) do
    quote do
      def hello, do: "Hello from #{__MODULE__}"
      # inject code อื่นๆ
    end
  end
end

defmodule User do
  use MyBehaviour
  # ตอนนี้ User มี hello/0 function แล้ว
end

User.hello()  # "Hello from Elixir.User"
```

---

## Step 37: Recursion แทน Loops

Elixir ไม่มี for/while loops แบบ imperative แต่ใช้ recursion แทน

### 37.1 Tail Recursion

```elixir
defmodule MyEnum do
  # Non-tail recursive (อาจ stack overflow)
  def sum([]), do: 0
  def sum([head | tail]) do
    head + sum(tail)  # การคำนวณยังรอ recursive call อยู่
  end
  
  # Tail recursive (BEAM optimize ให้อัตโนมัติ - ไม่ stack overflow)
  def sum_tail(list, acc \\ 0)
  def sum_tail([], acc), do: acc
  def sum_tail([head | tail], acc) do
    sum_tail(tail, acc + head)  # recursive call คือสิ่งสุดท้าย
  end
end

MyEnum.sum([1, 2, 3, 4, 5])       # 15
MyEnum.sum_tail([1, 2, 3, 4, 5])  # 15

# Tail recursion ไม่มี stack overflow แม้ list จะมีล้าน elements
large_list = Enum.to_list(1..1_000_000)
MyEnum.sum_tail(large_list)  # ทำงานได้ปกติ
```

### 37.2 Pattern Recursion

```elixir
defmodule ListOps do
  # map
  def my_map([], _func), do: []
  def my_map([head | tail], func) do
    [func.(head) | my_map(tail, func)]
  end
  
  # filter
  def my_filter([], _pred), do: []
  def my_filter([head | tail], pred) do
    if pred.(head) do
      [head | my_filter(tail, pred)]
    else
      my_filter(tail, pred)
    end
  end
  
  # reduce
  def my_reduce([], acc, _func), do: acc
  def my_reduce([head | tail], acc, func) do
    my_reduce(tail, func.(head, acc), func)
  end
end

ListOps.my_map([1, 2, 3], fn x -> x * 2 end)
# [2, 4, 6]

ListOps.my_filter([1, 2, 3, 4, 5], fn x -> x > 3 end)
# [4, 5]

ListOps.my_reduce([1, 2, 3, 4, 5], 0, fn x, acc -> x + acc end)
# 15
```

---

## Step 38: Higher-Order Functions

### 38.1 Functions เป็น First-class Values

```elixir
# ส่ง function เป็น argument
defmodule Transformer do
  def apply_twice(func, value) do
    func.(func.(value))
  end
  
  def compose(f, g) do
    fn x -> f.(g.(x)) end
  end
end

double = fn x -> x * 2 end
add_one = fn x -> x + 1 end

Transformer.apply_twice(double, 3)   # double(double(3)) = 12

triple_then_add_one = Transformer.compose(add_one, fn x -> x * 3 end)
triple_then_add_one.(4)  # (4 * 3) + 1 = 13
```

### 38.2 Function สร้าง Function

```elixir
defmodule FunctionFactory do
  def power_of(n) do
    fn x -> :math.pow(x, n) end
  end
  
  def add_n(n) do
    fn x -> x + n end
  end
  
  def between(min, max) do
    fn x -> x >= min and x <= max end
  end
end

square  = FunctionFactory.power_of(2)
cube    = FunctionFactory.power_of(3)
add10   = FunctionFactory.add_n(10)
in_range = FunctionFactory.between(1, 10)

square.(5)     # 25.0
cube.(3)       # 27.0
add10.(5)      # 15
in_range.(5)   # true
in_range.(15)  # false
```

---

## Step 39: Module Composition และ Behaviours

### 39.1 Protocols vs Behaviours

```elixir
# Behaviour = Interface (สำหรับ struct/module เดียวกัน)
# Protocol = Polymorphism (สำหรับหลาย data types)

# ตัวอย่าง Behaviour
defmodule PaymentProcessor do
  @callback process_payment(amount :: float(), currency :: String.t()) :: 
    {:ok, String.t()} | {:error, String.t()}
  
  @callback refund_payment(transaction_id :: String.t()) ::
    {:ok, String.t()} | {:error, String.t()}
end

defmodule StripeProcessor do
  @behaviour PaymentProcessor
  
  def process_payment(amount, currency) do
    # Stripe API integration
    {:ok, "stripe_txn_#{:rand.uniform(9999)}"}
  end
  
  def refund_payment(transaction_id) do
    {:ok, "refunded_#{transaction_id}"}
  end
end

defmodule PayPalProcessor do
  @behaviour PaymentProcessor
  
  def process_payment(amount, currency) do
    {:ok, "paypal_txn_#{:rand.uniform(9999)}"}
  end
  
  def refund_payment(transaction_id) do
    {:ok, "refunded_#{transaction_id}"}
  end
end
```

### 39.2 Callbacks และ Optional Callbacks

```elixir
defmodule Plugin do
  @callback init(opts :: keyword()) :: {:ok, state :: any()} | {:error, reason :: any()}
  @callback handle_event(event :: any(), state :: any()) :: {:ok, state :: any()}
  
  # Optional callback
  @optional_callbacks [terminate: 1]
  @callback terminate(state :: any()) :: :ok
end

defmodule LoggerPlugin do
  @behaviour Plugin
  
  def init(opts) do
    level = Keyword.get(opts, :level, :info)
    {:ok, %{level: level, count: 0}}
  end
  
  def handle_event({:log, message}, state) do
    IO.puts("[#{state.level}] #{message}")
    {:ok, %{state | count: state.count + 1}}
  end
  
  # Optional - ไม่จำเป็นต้อง implement
  def terminate(state) do
    IO.puts("Logger stopped. Total logs: #{state.count}")
    :ok
  end
end
```

---

## Step 40: Comprehensive Module Example

สร้าง Module ที่ครบวงจร - User Management System:

```elixir
defmodule UserManagement do
  @moduledoc """
  Module สำหรับจัดการ Users ในระบบ
  
  ## ตัวอย่างการใช้งาน
  
      iex> UserManagement.create_user(%{name: "Alice", email: "alice@example.com"})
      {:ok, %{id: 1, name: "Alice", email: "alice@example.com", role: :user}}
  """
  
  # Module attributes
  @min_password_length 8
  @default_role :user
  @valid_roles [:admin, :moderator, :user, :guest]
  
  # Type specs
  @type user :: %{
    id: pos_integer(),
    name: String.t(),
    email: String.t(),
    role: atom()
  }
  
  @type result(t) :: {:ok, t} | {:error, String.t()}
  
  # ----- Public API -----
  
  @doc """
  สร้าง user ใหม่
  """
  @spec create_user(map()) :: result(user())
  def create_user(attrs) do
    with {:ok, attrs} <- validate_required(attrs, [:name, :email]),
         {:ok, attrs} <- validate_email(attrs),
         {:ok, attrs} <- set_defaults(attrs) do
      user = Map.put(attrs, :id, generate_id())
      {:ok, user}
    end
  end
  
  @doc """
  อัปเดต user
  """
  @spec update_user(user(), map()) :: result(user())
  def update_user(user, attrs) do
    with {:ok, attrs} <- validate_update_attrs(attrs) do
      updated_user = Map.merge(user, attrs)
      {:ok, updated_user}
    end
  end
  
  @doc """
  ตรวจสอบว่า user มี permission หรือไม่
  """
  @spec can_do?(user(), atom()) :: boolean()
  def can_do?(%{role: :admin}, _action), do: true
  def can_do?(%{role: :moderator}, action) when action in [:read, :write, :moderate], do: true
  def can_do?(%{role: :user}, action) when action in [:read, :write], do: true
  def can_do?(%{role: :guest}, :read), do: true
  def can_do?(_, _), do: false
  
  # ----- Private Functions -----
  
  defp validate_required(attrs, required_fields) do
    missing = Enum.filter(required_fields, fn field ->
      not Map.has_key?(attrs, field) or 
      attrs[field] == nil or
      attrs[field] == ""
    end)
    
    case missing do
      [] -> {:ok, attrs}
      fields -> {:error, "Missing required fields: #{Enum.join(fields, ", ")}"}
    end
  end
  
  defp validate_email(attrs) do
    email = attrs[:email] || attrs["email"]
    
    if String.contains?(email, "@") and String.contains?(email, ".") do
      {:ok, attrs}
    else
      {:error, "Invalid email format"}
    end
  end
  
  defp set_defaults(attrs) do
    defaults = %{
      role: @default_role,
      active: true,
      created_at: DateTime.utc_now()
    }
    {:ok, Map.merge(defaults, attrs)}
  end
  
  defp validate_update_attrs(attrs) do
    # ตรวจสอบว่าไม่มีการเปลี่ยน id หรือ email (ในตัวอย่างนี้)
    forbidden = [:id]
    has_forbidden = Enum.any?(forbidden, &Map.has_key?(attrs, &1))
    
    if has_forbidden do
      {:error, "Cannot update protected fields"}
    else
      {:ok, attrs}
    end
  end
  
  defp generate_id do
    :erlang.unique_integer([:positive])
  end
end

# การใช้งาน
case UserManagement.create_user(%{name: "Alice", email: "alice@example.com"}) do
  {:ok, user} ->
    IO.puts("Created user: #{user.name}")
    
    # ตรวจสอบ permissions
    IO.puts("Can read: #{UserManagement.can_do?(user, :read)}")    # true
    IO.puts("Can admin: #{UserManagement.can_do?(user, :admin)}")  # false
    
  {:error, reason} ->
    IO.puts("Error: #{reason}")
end
```

---

## สรุป Part 04: Functions และ Modules

✅ **Step 31** - Named Functions และ Module structure  
✅ **Step 32** - Function Arity และ Default Arguments  
✅ **Step 33** - Anonymous Functions และ Closures  
✅ **Step 34** - Pipe Operator |>  
✅ **Step 35** - Module Attributes  
✅ **Step 36** - import, alias, use, require  
✅ **Step 37** - Recursion แทน Loops  
✅ **Step 38** - Higher-Order Functions  
✅ **Step 39** - Behaviours  
✅ **Step 40** - Comprehensive Example  

---

## แบบฝึกหัด Part 04

1. สร้าง Module `MathUtils` ที่มี:
   - `factorial/1` - หา factorial ด้วย recursion
   - `gcd/2` - Greatest Common Divisor
   - `is_prime/1` - ตรวจสอบว่าเป็นจำนวนเฉพาะ
   - `primes_up_to/1` - หาจำนวนเฉพาะทั้งหมดไม่เกิน n

2. สร้าง Function Factory `Validator` ที่ return functions:
   - `min_length(n)` - สร้าง validator ที่ตรวจ string length >= n
   - `max_value(n)` - สร้าง validator ที่ตรวจ number <= n
   - `matches(regex)` - สร้าง validator ที่ตรวจ string ด้วย regex

3. แปลง code นี้ให้ใช้ Pipe Operator:
   ```elixir
   Enum.join(Enum.map(Enum.filter([1,2,3,4,5], fn x -> rem(x, 2) == 0 end), fn x -> to_string(x) end), ", ")
   ```

---

## ไปต่อ

➡️ [Part 05: Control Flow](./part-05-control-flow.md)
