# Part 03: Pattern Matching (Steps 21-30)

> Pattern Matching คือหัวใจของ Elixir ที่ทำให้โค้ดอ่านง่ายและทรงพลังมาก

## Step 21: = คือ Match Operator ไม่ใช่ Assignment!

ใน Elixir `=` ไม่ใช่การ assign ค่า แต่คือ **Match Operator**

### 21.1 การทำงานของ Match Operator

```elixir
# ใน Python/Ruby: x = 5 หมายถึง "กำหนดค่า 5 ให้ตัวแปร x"
# ใน Elixir: x = 5 หมายถึง "ทำให้ x match กับ 5"

iex> x = 5
5

iex> 5 = x    # x ถูก bind เป็น 5 แล้ว ดังนั้น 5 = 5 ✓
5

iex> 6 = x    # 6 ≠ 5 → MatchError!
# ** (MatchError) no match of right hand side value: 5
```

### 21.2 การ Bind Variables

```elixir
# Elixir จะ bind variable เมื่อ match สำเร็จ
iex> {a, b, c} = {1, 2, 3}
{1, 2, 3}

iex> a
1
iex> b
2
iex> c
3

# Match List
iex> [x, y, z] = [4, 5, 6]
[4, 5, 6]

iex> x
4
iex> y
5
iex> z
6
```

### 21.3 _ (Underscore) - ค่าที่ไม่สนใจ

```elixir
# _ คือ wildcard ที่ match ทุกอย่างโดยไม่ bind
iex> {_, b} = {1, 2}
{1, 2}
iex> b
2
# iex> _   # ใช้ _ ไม่ได้ (ไม่ถูก bind)

# ใช้ชื่อที่ขึ้นต้น _ สำหรับ readability
iex> {_first, second, _third} = {1, 2, 3}
iex> second
2
```

### 21.4 Pin Operator (^)

```elixir
# ^ ใช้เมื่อต้องการ match กับค่าที่ bind แล้ว (ไม่ rebind)
iex> x = 5
iex> {^x, y} = {5, 10}   # ^x หมายถึง "ตรวจสอบว่า element แรกเท่ากับ x"
{5, 10}
iex> y
10

iex> {^x, y} = {6, 10}   # 6 ≠ 5 → MatchError!
# ** (MatchError) no match of right hand side value: {6, 10}

# ใช้กับ Case
x = 5
case some_value do
  ^x -> "matches x"    # match เฉพาะถ้า some_value == 5
  y  -> "binds to y"   # match กรณีอื่น
end
```

---

## Step 22: Pattern Matching กับ Tuple

### 22.1 Tuple Matching พื้นฐาน

```elixir
# Match tuple size และ values
iex> {:ok, value} = {:ok, "success"}
{:ok, "success"}
iex> value
"success"

iex> {:ok, value} = {:error, "failed"}
# ** (MatchError) no match of right hand side value: {:error, "failed"}

# ดึงหลาย values
iex> {name, age, role} = {"Alice", 30, :admin}
iex> name
"Alice"
iex> age
30
iex> role
:admin
```

### 22.2 Convention: {:ok, value} / {:error, reason}

```elixir
# Pattern นี้ใช้ทั่ว Elixir ecosystem
defmodule UserAuth do
  def login(email, password) do
    case Database.find_user(email) do
      nil  -> {:error, :user_not_found}
      user ->
        if verify_password(password, user.password_hash) do
          {:ok, user}
        else
          {:error, :invalid_password}
        end
    end
  end
end

# การใช้งาน
case UserAuth.login("alice@example.com", "password") do
  {:ok, user}              -> redirect_to_dashboard(user)
  {:error, :user_not_found} -> render_error("User not found")
  {:error, :invalid_password} -> render_error("Wrong password")
end
```

---

## Step 23: Pattern Matching กับ List

### 23.1 List Matching

```elixir
# Match specific elements
iex> [1, 2, 3] = [1, 2, 3]
[1, 2, 3]

iex> [1, 2, 3] = [1, 2, 4]
# ** (MatchError)

# Head | Tail pattern (สำคัญมาก!)
iex> [head | tail] = [1, 2, 3, 4, 5]
iex> head
1
iex> tail
[2, 3, 4, 5]

# Multiple heads
iex> [first, second | rest] = [1, 2, 3, 4, 5]
iex> first
1
iex> second
2
iex> rest
[3, 4, 5]

# Empty list match
iex> [] = []
[]

iex> [] = [1]
# ** (MatchError)
```

### 23.2 Recursive Processing ด้วย Pattern

```elixir
defmodule MyList do
  # Base case: empty list
  def sum([]), do: 0
  
  # Recursive case: head + sum(tail)
  def sum([head | tail]) do
    head + sum(tail)
  end
end

MyList.sum([1, 2, 3, 4, 5])  # => 15

# อธิบาย:
# sum([1, 2, 3, 4, 5])
# = 1 + sum([2, 3, 4, 5])
# = 1 + 2 + sum([3, 4, 5])
# = 1 + 2 + 3 + sum([4, 5])
# = 1 + 2 + 3 + 4 + sum([5])
# = 1 + 2 + 3 + 4 + 5 + sum([])
# = 1 + 2 + 3 + 4 + 5 + 0
# = 15
```

---

## Step 24: Pattern Matching กับ Map

### 24.1 Map Matching

```elixir
# ดึงค่าจาก Map
iex> %{name: name} = %{name: "Alice", age: 30}
iex> name
"Alice"

# Map matching ไม่จำเป็นต้อง match ทุก key!
iex> %{name: name, age: age} = %{name: "Alice", age: 30, role: :admin}
# ✓ ไม่ต้องมี :role ก็ match ได้

# แต่ถ้า key ที่ระบุไม่มีใน Map → MatchError
iex> %{phone: phone} = %{name: "Alice", age: 30}
# ** (MatchError) no match of right hand side value: %{age: 30, name: "Alice"}
```

### 24.2 Pattern กับ Nested Map

```elixir
user = %{
  name: "Alice",
  address: %{
    city: "Bangkok",
    country: "Thailand"
  }
}

# ดึง nested value
%{name: name, address: %{city: city}} = user
IO.puts("#{name} lives in #{city}")
# Alice lives in Bangkok

# ใน function
def greet(%{name: name, address: %{city: city}}) do
  "Hello #{name} from #{city}!"
end

greet(user)  # "Hello Alice from Bangkok!"
```

---

## Step 25: Pattern Matching กับ String

### 25.1 String Pattern Matching

```elixir
# ต้อง match ทั้ง string
iex> "hello" = "hello"
"hello"

iex> "hello" = "world"
# ** (MatchError)

# String Concatenation Pattern
iex> "Hello, " <> name = "Hello, Alice"
iex> name
"Alice"

# Pattern ที่ใช้บ่อย: ตรวจสอบ prefix
defmodule Protocol do
  def parse("http://" <> rest), do: {:http, rest}
  def parse("https://" <> rest), do: {:https, rest}
  def parse("ftp://" <> rest), do: {:ftp, rest}
  def parse(url), do: {:unknown, url}
end

Protocol.parse("https://example.com")  # {:https, "example.com"}
Protocol.parse("ftp://files.com")      # {:ftp, "files.com"}
Protocol.parse("example.com")          # {:unknown, "example.com"}
```

### 25.2 Binary Pattern Matching

```elixir
# Pattern matching กับ binary
iex> <<r, g, b>> = <<255, 128, 0>>
iex> r
255
iex> g
128
iex> b
0

# กำหนด bit size
iex> <<r::8, g::8, b::8>> = <<255, 128, 0>>

# ใช้ใน Protocol parsing
def parse_header(<<version::4, ihl::4, _rest::binary>>) do
  {version, ihl}
end

parse_header(<<0x45, "rest of header">>)  # {4, 5}
```

---

## Step 26: case Expression

`case` ใช้สำหรับ pattern matching หลายรูปแบบ

### 26.1 case พื้นฐาน

```elixir
# case expression
result = case {1, 2, 3} do
  {4, 5, 6} ->
    "First clause"
  {1, x, 3} ->
    "Second clause: x = #{x}"
  _ ->
    "Catch-all clause"
end

# result = "Second clause: x = 2"
```

### 26.2 case กับ {:ok}/{:error}

```elixir
case File.read("config.json") do
  {:ok, content} ->
    IO.puts("File content: #{content}")
  
  {:error, :enoent} ->
    IO.puts("File not found")
  
  {:error, :eacces} ->
    IO.puts("Permission denied")
  
  {:error, reason} ->
    IO.puts("Unknown error: #{reason}")
end
```

### 26.3 Guard Clauses ใน case

```elixir
case x do
  n when n < 0 ->
    "negative"
  0 ->
    "zero"
  n when n > 0 and n < 10 ->
    "small positive"
  n when n >= 10 ->
    "large positive"
end
```

### 26.4 case กับ List

```elixir
def process_list(list) do
  case list do
    [] ->
      "empty list"
    [single] ->
      "list with one element: #{single}"
    [first, second | rest] ->
      "list with #{length(rest) + 2} elements, first: #{first}, second: #{second}"
  end
end

process_list([])        # "empty list"
process_list([42])      # "list with one element: 42"
process_list([1, 2, 3]) # "list with 3 elements, first: 1, second: 2"
```

---

## Step 27: cond Expression

`cond` เหมือน if-elseif chain

### 27.1 cond พื้นฐาน

```elixir
cond do
  2 + 2 == 5 ->
    "This is never true"
  2 * 2 == 3 ->
    "Nor this"
  1 + 1 == 2 ->
    "But this will"
end

# "But this will"
```

### 27.2 cond vs case

```elixir
# ใช้ cond เมื่อ conditions แตกต่างกันมาก
score = 85

grade = cond do
  score >= 90 -> "A"
  score >= 80 -> "B"
  score >= 70 -> "C"
  score >= 60 -> "D"
  true        -> "F"  # catch-all ต้องใส่ true
end

# grade = "B"

# ใช้ case เมื่อ pattern matching กับค่าเดียวกัน
day = :monday

activity = case day do
  :monday    -> "Standup meeting"
  :tuesday   -> "Planning session"
  :wednesday -> "Code review"
  :thursday  -> "Sprint review"
  :friday    -> "Retrospective"
  _          -> "Weekend!"
end
```

---

## Step 28: Guard Clauses (when)

Guards เป็นเงื่อนไขเพิ่มเติมสำหรับ Pattern Matching

### 28.1 Guards พื้นฐาน

```elixir
defmodule Guard do
  # Guards ใช้ when keyword
  def classify(n) when n < 0, do: "negative"
  def classify(0), do: "zero"
  def classify(n) when n > 0, do: "positive"
end

Guard.classify(-5)  # "negative"
Guard.classify(0)   # "zero"
Guard.classify(5)   # "positive"
```

### 28.2 Guard Functions ที่ใช้ได้

```elixir
# ใช้ใน guards ได้:
is_atom/1
is_binary/1
is_bitstring/1
is_boolean/1
is_float/1
is_function/1, is_function/2
is_integer/1
is_list/1
is_map/1
is_nil/1
is_number/1
is_pid/1
is_port/1
is_reference/1
is_struct/1, is_struct/2
is_tuple/1

# Math operators: +, -, *, /, abs, ceil, floor, round, trunc
# Comparison: ==, !=, ===, !==, <, >, <=, >=
# Logical: and, or, not
# Type checks: is_*
# Length: length/1, byte_size/1, map_size/1, tuple_size/1

# ตัวอย่าง
def process(x) when is_integer(x) and x > 0 do
  "positive integer: #{x}"
end

def process(x) when is_binary(x) and byte_size(x) > 0 do
  "non-empty string: #{x}"
end

def process(_), do: "other"
```

### 28.3 Custom Guards

```elixir
# ใน Elixir 1.6+ สร้าง guard เองได้ด้วย defguard
defmodule MyGuards do
  defguard is_even(n) when is_integer(n) and rem(n, 2) == 0
  defguard is_positive_string(s) when is_binary(s) and byte_size(s) > 0
end

defmodule Calculator do
  import MyGuards
  
  def double(n) when is_even(n), do: n * 2
  def double(n), do: {:error, "#{n} is not even"}
end

Calculator.double(4)  # 8
Calculator.double(3)  # {:error, "3 is not even"}
```

---

## Step 29: Pattern Matching ใน Function Definitions

### 29.1 Multiple Function Clauses

```elixir
defmodule Fibonacci do
  # Pattern matching ใน function definition
  def fib(0), do: 0
  def fib(1), do: 1
  def fib(n) when n > 1 do
    fib(n - 1) + fib(n - 2)
  end
end

Fibonacci.fib(0)   # 0
Fibonacci.fib(1)   # 1
Fibonacci.fib(10)  # 55
```

### 29.2 Pattern กับ Function Parameters

```elixir
defmodule Shape do
  # Pattern matching กับ Atom ใน tuple
  def area({:circle, radius}) do
    :math.pi() * radius * radius
  end
  
  def area({:rectangle, width, height}) do
    width * height
  end
  
  def area({:triangle, base, height}) do
    0.5 * base * height
  end
  
  def area(shape) do
    {:error, "Unknown shape: #{inspect(shape)}"}
  end
end

Shape.area({:circle, 5})         # 78.53981633974483
Shape.area({:rectangle, 4, 6})   # 24
Shape.area({:triangle, 3, 4})    # 6.0
Shape.area({:pentagon, 5})       # {:error, "Unknown shape: {:pentagon, 5}"}
```

### 29.3 Struct Pattern Matching

```elixir
defmodule User do
  defstruct [:name, :role, :age]
end

defmodule AccessControl do
  def can_access?(%User{role: :admin}), do: true
  def can_access?(%User{role: :moderator, age: age}) when age >= 18, do: true
  def can_access?(%User{}), do: false
end

alice = %User{name: "Alice", role: :admin, age: 30}
bob   = %User{name: "Bob", role: :moderator, age: 17}
charlie = %User{name: "Charlie", role: :user, age: 25}

AccessControl.can_access?(alice)    # true
AccessControl.can_access?(bob)      # false (moderator but underage)
AccessControl.can_access?(charlie)  # false (regular user)
```

---

## Step 30: Comprehensive Pattern Matching Examples

### 30.1 HTTP Status Code Handler

```elixir
defmodule HTTPHandler do
  def handle_response({status, body}) when status in 200..299 do
    {:success, body}
  end
  
  def handle_response({301, location}) do
    {:redirect, location}
  end
  
  def handle_response({302, location}) do
    {:temporary_redirect, location}
  end
  
  def handle_response({400, body}) do
    {:bad_request, body}
  end
  
  def handle_response({401, _}) do
    {:unauthorized, "Authentication required"}
  end
  
  def handle_response({403, _}) do
    {:forbidden, "Access denied"}
  end
  
  def handle_response({404, _}) do
    {:not_found, "Resource not found"}
  end
  
  def handle_response({500, body}) do
    {:server_error, body}
  end
  
  def handle_response({status, body}) do
    {:unknown_error, status, body}
  end
end

HTTPHandler.handle_response({200, "OK"})         # {:success, "OK"}
HTTPHandler.handle_response({404, "Not Found"})  # {:not_found, "Resource not found"}
HTTPHandler.handle_response({301, "/new-path"})  # {:redirect, "/new-path"}
```

### 30.2 Data Transformation Pipeline

```elixir
defmodule DataProcessor do
  def process(data) do
    data
    |> parse()
    |> validate()
    |> transform()
    |> save()
  end
  
  defp parse(raw) when is_binary(raw) do
    case Jason.decode(raw) do
      {:ok, map}     -> {:ok, map}
      {:error, _}    -> {:error, :invalid_json}
    end
  end
  defp parse(_), do: {:error, :not_a_string}
  
  defp validate({:error, _} = error), do: error
  defp validate({:ok, %{"name" => name, "age" => age} = data}) 
    when is_binary(name) and is_integer(age) and age > 0 do
    {:ok, data}
  end
  defp validate({:ok, _}), do: {:error, :invalid_data}
  
  defp transform({:error, _} = error), do: error
  defp transform({:ok, data}) do
    transformed = %{
      name: data["name"],
      age: data["age"],
      created_at: DateTime.utc_now()
    }
    {:ok, transformed}
  end
  
  defp save({:error, _} = error), do: error
  defp save({:ok, data}) do
    # Database.insert(data)
    IO.puts("Saved: #{inspect(data)}")
    {:ok, :saved}
  end
end
```

### 30.3 Message Router (Actor Pattern)

```elixir
defmodule MessageRouter do
  def route({:chat, %{to: user, text: text}}) do
    IO.puts("Sending chat to #{user}: #{text}")
    {:ok, :chat_sent}
  end
  
  def route({:notification, %{type: :email, to: email} = opts}) do
    IO.puts("Sending email to #{email}")
    {:ok, :email_sent}
  end
  
  def route({:notification, %{type: :sms, to: phone} = opts}) do
    IO.puts("Sending SMS to #{phone}")
    {:ok, :sms_sent}
  end
  
  def route({:system, :shutdown}) do
    IO.puts("Shutting down system...")
    {:ok, :shutting_down}
  end
  
  def route({:system, :status}) do
    {:ok, %{status: :running, uptime: get_uptime()}}
  end
  
  def route(unknown) do
    {:error, {:unknown_message, unknown}}
  end
  
  defp get_uptime, do: :os.system_time(:second)
end

MessageRouter.route({:chat, %{to: "alice", text: "Hello!"}})
MessageRouter.route({:notification, %{type: :email, to: "bob@example.com"}})
MessageRouter.route({:system, :status})
```

---

## สรุป Part 03: Pattern Matching

### สิ่งที่ต้องจำ

1. **= คือ Match Operator** ไม่ใช่ Assignment
2. **_ คือ wildcard** ที่ match ทุกอย่างโดยไม่ bind
3. **^ คือ Pin Operator** ใช้ match กับค่าที่ bind แล้ว
4. **case** ใช้ match หลายรูปแบบ
5. **cond** ใช้กับ conditions หลายอัน
6. **when** ใช้เพิ่มเงื่อนไขใน pattern
7. **Multiple function clauses** ใช้ pattern ในการ define function

### Pattern Matching ทำให้โค้ด Elixir:
- อ่านง่ายขึ้นมาก
- ลด if/else ซ้อนกัน
- Handle หลาย cases ได้อย่างชัดเจน
- ทนทานต่อ unexpected data

---

## แบบฝึกหัด Part 03

1. เขียน function `describe/1` ที่ pattern match กับ:
   - Integer บวก/ลบ/ศูนย์
   - String ว่าง/ไม่ว่าง
   - List ว่าง/มีหนึ่ง element/หลาย elements
   - Tuple {:ok, _} / {:error, _} / อื่นๆ

2. เขียน function `fibonacci/1` ด้วย pattern matching (ไม่ใช้ if)

3. สร้าง `parse_command/1` ที่ parse string เป็น commands:
   - "quit" → :quit
   - "help" → :help
   - "echo <text>" → {:echo, text}
   - "calculate <a> + <b>" → {:add, a, b}
   - อื่นๆ → {:unknown, command}

---

## ไปต่อ

➡️ [Part 04: Functions และ Modules](./part-04-functions.md)
