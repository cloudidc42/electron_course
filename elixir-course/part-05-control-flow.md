# Part 05: Control Flow (Steps 41-50)

## Step 41: if และ unless

### 41.1 if Expression

```elixir
# if เป็น expression ที่ return ค่า
result = if true do
  "this is true"
end
# result = "this is true"

result = if false do
  "this is true"
end
# result = nil  (if ที่ไม่ผ่านจะ return nil)

# if/else
status = if age >= 18 do
  :adult
else
  :minor
end

# Inline if
greeting = if time < 12, do: "Good morning", else: "Good afternoon"
```

### 41.2 unless Expression

```elixir
# unless = if not
result = unless false do
  "this executes"
end
# result = "this executes"

# ใช้เมื่อ condition เป็นแบบ negative
unless user.banned? do
  IO.puts("Welcome, #{user.name}!")
end

# เทียบกับ if
if not user.banned? do
  IO.puts("Welcome, #{user.name}!")
end
```

### 41.3 if กับ Side Effects

```elixir
# ใน Elixir ทุกอย่างเป็น expression (return value)
# แต่บางครั้งเราใช้ if เพื่อ side effects
if should_log? do
  Logger.info("Processing request")
end

# Best practice: ถ้าต้องการ return value ให้ใช้ case
# ถ้า side effect เท่านั้นใช้ if ได้
```

---

## Step 42: case Expression (ลึกขึ้น)

### 42.1 case กับ Complex Patterns

```elixir
def process_payment(amount, method) do
  case {amount, method} do
    {amount, _} when amount <= 0 ->
      {:error, "Amount must be positive"}
    
    {amount, :credit_card} when amount > 10_000 ->
      {:error, "Credit card limit exceeded"}
    
    {amount, :credit_card} ->
      charge_credit_card(amount)
    
    {amount, :bank_transfer} when amount > 1_000_000 ->
      {:requires_approval, amount}
    
    {amount, :bank_transfer} ->
      initiate_bank_transfer(amount)
    
    {_, method} ->
      {:error, "Unknown payment method: #{method}"}
  end
end
```

### 42.2 case ที่ catch ทุกกรณี

```elixir
# ควรมี catch-all case เสมอ
def handle_event(event) do
  case event do
    {:user_created, user} ->
      send_welcome_email(user)
    
    {:user_updated, user, changes} ->
      notify_user_of_changes(user, changes)
    
    {:user_deleted, user_id} ->
      cleanup_user_data(user_id)
    
    unknown ->
      Logger.warning("Unknown event: #{inspect(unknown)}")
      {:error, :unknown_event}
  end
end
```

---

## Step 43: cond Expression (ลึกขึ้น)

### 43.1 cond กับ Multiple Conditions

```elixir
def categorize_temperature(temp) do
  cond do
    temp < 0   -> :freezing
    temp < 10  -> :very_cold
    temp < 20  -> :cold
    temp < 30  -> :comfortable
    temp < 40  -> :hot
    true       -> :very_hot  # catch-all (ต้องมีเสมอ!)
  end
end

categorize_temperature(-5)  # :freezing
categorize_temperature(25)  # :comfortable
categorize_temperature(45)  # :very_hot
```

### 43.2 cond กับ Guards

```elixir
def classify_number(n) do
  cond do
    not is_number(n)           -> {:error, "Not a number"}
    n < 0                      -> :negative
    n == 0                     -> :zero
    rem(n, 2) == 0             -> :positive_even
    true                       -> :positive_odd
  end
end
```

---

## Step 44: with Expression

`with` เป็น control flow ที่ทรงพลังมากสำหรับ happy path

### 44.1 with พื้นฐาน

```elixir
# โดยไม่ใช้ with (callback hell)
case validate_user(user) do
  {:ok, user} ->
    case check_permissions(user) do
      {:ok, user} ->
        case load_resource(resource_id) do
          {:ok, resource} ->
            process(user, resource)
          {:error, reason} ->
            {:error, reason}
        end
      {:error, reason} ->
        {:error, reason}
    end
  {:error, reason} ->
    {:error, reason}
end

# ใช้ with - อ่านง่ายกว่ามาก!
with {:ok, user}     <- validate_user(user),
     {:ok, user}     <- check_permissions(user),
     {:ok, resource} <- load_resource(resource_id) do
  process(user, resource)
end
```

### 44.2 with กับ else

```elixir
def create_order(user_id, items) do
  with {:ok, user}  <- fetch_user(user_id),
       {:ok, _}     <- check_user_active(user),
       {:ok, prices} <- get_prices(items),
       total         = calculate_total(prices),
       {:ok, _}      <- check_sufficient_balance(user, total),
       {:ok, order}  <- place_order(user, items, total) do
    {:ok, order}
  else
    {:error, :user_not_found} ->
      {:error, "User #{user_id} not found"}
    
    {:error, :user_inactive} ->
      {:error, "User account is not active"}
    
    {:error, :insufficient_balance} ->
      {:error, "Insufficient balance"}
    
    {:error, reason} ->
      {:error, "Unexpected error: #{inspect(reason)}"}
  end
end
```

### 44.3 with กับ Pattern Variables

```elixir
# ตัวแปรที่ bind ใน with สามารถใช้ใน pattern ถัดไปได้
def process_data(raw_data) do
  with {:ok, parsed}      <- parse_json(raw_data),
       user_id = parsed["user_id"],         # bind โดยไม่ match
       {:ok, user}        <- fetch_user(user_id),
       {:ok, processed}   <- transform(parsed, user) do
    {:ok, processed}
  end
end
```

---

## Step 45: for Comprehension

`for` ใน Elixir ไม่ใช่ loop แบบ imperative แต่เป็น List Comprehension

### 45.1 for พื้นฐาน

```elixir
# สร้าง list ด้วย for
iex> for x <- [1, 2, 3], do: x * 2
[2, 4, 6]

# เหมือนกับ Enum.map
iex> Enum.map([1, 2, 3], &(&1 * 2))
[2, 4, 6]

# for กับ multiple generators
iex> for x <- [1, 2, 3], y <- [10, 20], do: {x, y}
[{1, 10}, {1, 20}, {2, 10}, {2, 20}, {3, 10}, {3, 20}]

# Cartesian product
iex> for x <- 1..3, y <- 1..3, do: x * y
[1, 2, 3, 2, 4, 6, 3, 6, 9]
```

### 45.2 for กับ Filters

```elixir
# เพิ่ม filter ด้วย when
iex> for x <- 1..10, rem(x, 2) == 0, do: x
[2, 4, 6, 8, 10]

# หลาย filters
iex> for x <- 1..100, 
         rem(x, 3) == 0,
         rem(x, 5) == 0,
         do: x
[15, 30, 45, 60, 75, 90]

# FizzBuzz ด้วย for
for x <- 1..20 do
  cond do
    rem(x, 15) == 0 -> "FizzBuzz"
    rem(x, 3)  == 0 -> "Fizz"
    rem(x, 5)  == 0 -> "Buzz"
    true            -> x
  end
end
```

### 45.3 for กับ into

```elixir
# into ใช้กำหนด output type
# สร้าง Map
iex> for {key, val} <- [a: 1, b: 2, c: 3], into: %{}, do: {key, val * 2}
%{a: 2, b: 4, c: 6}

# สร้าง String
iex> for char <- "hello", into: "", do: String.upcase(<<char::utf8>>)
"HELLO"

# สร้าง Map จาก List of Maps
users = [%{name: "Alice"}, %{name: "Bob"}]
iex> for user <- users, into: %{}, do: {user.name, user}
%{"Alice" => %{name: "Alice"}, "Bob" => %{name: "Bob"}}
```

### 45.4 for กับ Pattern Matching

```elixir
# for รองรับ pattern matching
responses = [
  {:ok, "result1"},
  {:error, "error1"},
  {:ok, "result2"},
  {:error, "error2"},
  {:ok, "result3"}
]

# ดึงเฉพาะ :ok results
for {:ok, result} <- responses, do: result
# ["result1", "result2", "result3"]
# pattern ที่ไม่ match จะถูก skip โดยอัตโนมัติ
```

---

## Step 46: receive Expression (Process Messages)

### 46.1 receive พื้นฐาน

```elixir
# receive ใช้ในการรับ messages ใน Process
def listen do
  receive do
    {:hello, from} ->
      IO.puts("Got hello from #{inspect(from)}")
      listen()  # recursive เพื่อรับ message ต่อไป
    
    {:stop} ->
      IO.puts("Stopping...")
    
    msg ->
      IO.puts("Unknown message: #{inspect(msg)}")
      listen()
  end
end

# เริ่ม process
pid = spawn(fn -> listen() end)

# ส่ง messages
send(pid, {:hello, self()})
send(pid, {:stop})
```

### 46.2 receive กับ timeout

```elixir
def wait_for_response(timeout \\ 5000) do
  receive do
    {:response, data} ->
      {:ok, data}
    
    {:error, reason} ->
      {:error, reason}
  after
    timeout ->
      {:error, :timeout}
  end
end

# รอ response แค่ 1 วินาที
case wait_for_response(1000) do
  {:ok, data}     -> process_data(data)
  {:error, :timeout} -> IO.puts("Request timed out!")
  {:error, reason}   -> IO.puts("Error: #{reason}")
end
```

---

## Step 47: try/rescue/catch

### 47.1 try/rescue

```elixir
# rescue จัดการ exceptions
def safe_divide(a, b) do
  try do
    result = a / b
    {:ok, result}
  rescue
    ArithmeticError ->
      {:error, "Division by zero"}
    
    error ->
      {:error, "Unexpected error: #{inspect(error)}"}
  end
end

safe_divide(10, 2)  # {:ok, 5.0}
safe_divide(10, 0)  # {:error, "Division by zero"}

# ดึง error message
try do
  raise "Something went wrong"
rescue
  e in RuntimeError ->
    IO.puts("RuntimeError: #{e.message}")
  
  e ->
    IO.puts("Error: #{inspect(e)}")
end
```

### 47.2 try/catch

```elixir
# catch จัดการ throws
def process_with_early_exit(data) do
  try do
    result = data
    |> validate()
    |> transform()
    |> save()
    
    {:ok, result}
  catch
    :throw, {:early_exit, reason} ->
      {:skipped, reason}
    
    :exit, reason ->
      {:process_exited, reason}
    
    kind, value ->
      {:error, {kind, value}}
  end
end

defp validate(data) do
  if data[:skip] do
    throw({:early_exit, :marked_as_skip})
  end
  data
end
```

### 47.3 try/after

```elixir
# after รันเสมอ ไม่ว่าจะ error หรือไม่
def read_file(path) do
  file = File.open!(path)
  
  try do
    IO.read(file, :all)
  rescue
    error -> {:error, error}
  after
    File.close(file)  # ปิด file เสมอ
  end
end
```

### 47.4 Elixir Philosophy: ไม่ค่อยใช้ try/rescue

```elixir
# Elixir ส่งเสริมการใช้ {:ok}/{:error} tuples
# มากกว่าการ throw exceptions

# ไม่ดี (exception-heavy)
def get_user(id) do
  user = Database.find!(id)  # raise ถ้าไม่เจอ
  user
rescue
  NotFoundError -> nil
end

# ดีกว่า (explicit error handling)
def get_user(id) do
  case Database.find(id) do
    {:ok, user}  -> user
    {:error, _} -> nil
  end
end

# หรือสั้นกว่า
def get_user(id) do
  Database.find(id)
  |> case do
    {:ok, user}  -> user
    {:error, _} -> nil
  end
end
```

---

## Step 48: Boolean Operators ขั้นสูง

### 48.1 && และ || สำหรับ Short-circuit

```elixir
# && (and) - return ค่าขวาถ้าซ้ายเป็น truthy
# หรือ return ค่าซ้ายถ้าซ้ายเป็น falsy
iex> nil && "hello"
nil

iex> false && "hello"
false

iex> 1 && "hello"
"hello"

iex> true && 42
42

# || (or) - return ค่าซ้ายถ้าซ้ายเป็น truthy
# หรือ return ค่าขวาถ้าซ้ายเป็น falsy
iex> nil || "default"
"default"

iex> false || 42
42

iex> "value" || "default"
"value"

# ใช้งานจริง
def get_config(key) do
  System.get_env(key) || Application.get_env(:my_app, key) || "default"
end

def process(opts) do
  timeout = opts[:timeout] || 5000
  retries = opts[:retries] || 3
  # ...
end
```

### 48.2 and และ or (strict boolean)

```elixir
# and/or ต้องการ boolean จริงๆ ไม่ใช่ truthy/falsy
iex> true and false
false

iex> 1 and true
# ** (BadBooleanError) expected a boolean on left-side of "and", got: 1

# ใช้ and/or ใน guards เท่านั้น (ที่นิยม)
def process(x) when is_integer(x) and x > 0 do
  x
end
```

---

## Step 49: Special Control Flow Patterns

### 49.1 Early Return Pattern

```elixir
# Elixir ไม่มี return statement
# แต่ใช้ pattern matching เพื่อ "early return"

defmodule Validator do
  def validate(data) do
    # Multiple function clauses เป็น "early return"
    validate_not_empty(data)
  end
  
  defp validate_not_empty(nil), do: {:error, "Data is nil"}
  defp validate_not_empty(%{} = data) when map_size(data) == 0 do
    {:error, "Data is empty"}
  end
  defp validate_not_empty(data) do
    validate_required_fields(data)
  end
  
  defp validate_required_fields(%{name: nil}), do: {:error, "Name is required"}
  defp validate_required_fields(%{email: nil}), do: {:error, "Email is required"}
  defp validate_required_fields(data) do
    {:ok, data}
  end
end
```

### 49.2 Railway Oriented Programming

```elixir
defmodule Railway do
  # ทุก function return {:ok, value} หรือ {:error, reason}
  # และ "รถไฟ" วิ่งบน track ที่ถูกต้องตลอด
  
  def run(input) do
    input
    |> validate()
    |> transform()
    |> persist()
    |> notify()
  end
  
  defp validate({:error, _} = error), do: error
  defp validate(data) do
    if data[:valid], do: {:ok, data}, else: {:error, :invalid_data}
  end
  
  defp transform({:error, _} = error), do: error
  defp transform({:ok, data}) do
    {:ok, Map.put(data, :transformed, true)}
  end
  
  defp persist({:error, _} = error), do: error
  defp persist({:ok, data}) do
    # Save to DB
    {:ok, data}
  end
  
  defp notify({:error, reason}) do
    Logger.error("Process failed: #{inspect(reason)}")
    {:error, reason}
  end
  defp notify({:ok, data}) do
    Logger.info("Process succeeded")
    {:ok, data}
  end
end
```

---

## Step 50: Real-world Control Flow Examples

### 50.1 Form Validation System

```elixir
defmodule FormValidator do
  def validate_registration(params) do
    with {:ok, email}    <- validate_email(params["email"]),
         {:ok, password} <- validate_password(params["password"]),
         {:ok, name}     <- validate_name(params["name"]),
         {:ok, age}      <- validate_age(params["age"]) do
      
      {:ok, %{
        email: email,
        password: hash_password(password),
        name: name,
        age: age
      }}
    else
      {:error, field, reason} ->
        {:error, %{field: field, message: reason}}
    end
  end
  
  defp validate_email(nil), do: {:error, :email, "Email is required"}
  defp validate_email(email) when is_binary(email) do
    cond do
      String.length(email) < 5 ->
        {:error, :email, "Email too short"}
      
      not String.contains?(email, "@") ->
        {:error, :email, "Invalid email format"}
      
      true ->
        {:ok, String.downcase(email)}
    end
  end
  
  defp validate_password(nil), do: {:error, :password, "Password is required"}
  defp validate_password(pwd) when byte_size(pwd) < 8 do
    {:error, :password, "Password must be at least 8 characters"}
  end
  defp validate_password(pwd) do
    has_upper  = String.match?(pwd, ~r/[A-Z]/)
    has_digit  = String.match?(pwd, ~r/[0-9]/)
    
    cond do
      not has_upper -> {:error, :password, "Must contain uppercase letter"}
      not has_digit -> {:error, :password, "Must contain a digit"}
      true          -> {:ok, pwd}
    end
  end
  
  defp validate_name(nil), do: {:error, :name, "Name is required"}
  defp validate_name(name) when byte_size(name) < 2 do
    {:error, :name, "Name too short"}
  end
  defp validate_name(name), do: {:ok, String.trim(name)}
  
  defp validate_age(nil), do: {:error, :age, "Age is required"}
  defp validate_age(age) when is_binary(age) do
    case Integer.parse(age) do
      {n, ""} -> validate_age(n)
      _       -> {:error, :age, "Age must be a number"}
    end
  end
  defp validate_age(age) when age < 13 do
    {:error, :age, "Must be at least 13 years old"}
  end
  defp validate_age(age) when age > 150 do
    {:error, :age, "Invalid age"}
  end
  defp validate_age(age), do: {:ok, age}
  
  defp hash_password(pwd) do
    # ใช้ bcrypt ในการผลิต
    :crypto.hash(:sha256, pwd) |> Base.encode16()
  end
end

# การใช้งาน
params = %{
  "email" => "alice@example.com",
  "password" => "SecurePass123",
  "name" => "Alice",
  "age" => "25"
}

case FormValidator.validate_registration(params) do
  {:ok, user_data}          -> IO.puts("Valid! #{inspect(user_data)}")
  {:error, %{field: f, message: m}} -> IO.puts("Error in #{f}: #{m}")
end
```

### 50.2 State Machine

```elixir
defmodule OrderStateMachine do
  @valid_transitions %{
    :pending    => [:confirmed, :cancelled],
    :confirmed  => [:processing, :cancelled],
    :processing => [:shipped, :failed],
    :shipped    => [:delivered, :returned],
    :delivered  => [:returned],
    :cancelled  => [],
    :failed     => [:pending],
    :returned   => [:refunded]
  }
  
  def transition(current_state, new_state) do
    allowed = Map.get(@valid_transitions, current_state, [])
    
    cond do
      current_state == new_state ->
        {:error, "Already in #{new_state} state"}
      
      new_state in allowed ->
        {:ok, new_state}
      
      true ->
        {:error, "Cannot transition from #{current_state} to #{new_state}"}
    end
  end
  
  def can_cancel?(state) do
    :cancelled in Map.get(@valid_transitions, state, [])
  end
end

OrderStateMachine.transition(:pending, :confirmed)     # {:ok, :confirmed}
OrderStateMachine.transition(:pending, :delivered)     # {:error, "Cannot transition..."}
OrderStateMachine.transition(:confirmed, :confirmed)   # {:error, "Already in confirmed state"}
OrderStateMachine.can_cancel?(:pending)                # true
OrderStateMachine.can_cancel?(:delivered)              # false
```

---

## สรุป Part 05: Control Flow

✅ **Step 41** - if/unless  
✅ **Step 42** - case (ลึกขึ้น)  
✅ **Step 43** - cond  
✅ **Step 44** - with (happy path)  
✅ **Step 45** - for comprehension  
✅ **Step 46** - receive (Process messages)  
✅ **Step 47** - try/rescue/catch  
✅ **Step 48** - Boolean short-circuit operators  
✅ **Step 49** - Special control flow patterns  
✅ **Step 50** - Real-world examples  

### เมื่อไหร่ใช้อะไร?

| Situation | ใช้ |
|-----------|-----|
| Single condition | `if`/`unless` |
| Pattern match 1 value | `case` |
| Multiple conditions | `cond` |
| Chain of {:ok}/{:error} | `with` |
| Transform/filter list | `for` |
| Process message handling | `receive` |
| Exception handling | `try/rescue` |

---

## แบบฝึกหัด Part 05

1. สร้าง `calculate/3` ที่รับ operation (:add, :sub, :mul, :div) และ 2 numbers
   - ใช้ `case` กับ pattern matching
   - Handle division by zero ด้วย `with`

2. เขียน FizzBuzz 1-100 ด้วย `for` comprehension

3. สร้าง simple password validator ด้วย `with`:
   - ความยาวอย่างน้อย 8 ตัว
   - มีตัวพิมพ์ใหญ่
   - มีตัวเลข
   - มีอักขระพิเศษ

4. สร้าง Traffic Light state machine ด้วย `case`:
   - :red → :green
   - :green → :yellow
   - :yellow → :red

---

## ไปต่อ

➡️ [Part 06: Recursion และ Loops](./part-06-recursion.md)
