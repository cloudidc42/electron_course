# Part 02: ประเภทข้อมูลพื้นฐาน (Steps 11-20)

## Step 11: Integer และ Float

### 11.1 Integer (จำนวนเต็ม)

```elixir
# Integer ธรรมดา
iex> 42
42

iex> -17
-17

iex> 1_000_000    # ใช้ _ เพื่อความอ่านง่าย
1000000

# Binary (ฐาน 2) - ขึ้นต้นด้วย 0b
iex> 0b1010
10

iex> 0b11111111
255

# Octal (ฐาน 8) - ขึ้นต้นด้วย 0o
iex> 0o17
15

iex> 0o777
511

# Hexadecimal (ฐาน 16) - ขึ้นต้นด้วย 0x
iex> 0xFF
255

iex> 0x1A2B
6699
```

### 11.2 Integer Operations

```elixir
# การคำนวณพื้นฐาน
iex> 10 + 3       # บวก
13

iex> 10 - 3       # ลบ
7

iex> 10 * 3       # คูณ
30

iex> 10 / 3       # หาร (ผลลัพธ์เป็น Float เสมอ!)
3.3333333333333335

iex> div(10, 3)   # หารเอาเลขจำนวนเต็ม
3

iex> rem(10, 3)   # เศษจากการหาร
1

# การยกกำลัง (ไม่มี operator ** ใน Elixir)
iex> :math.pow(2, 10)    # 2^10
1024.0

iex> Integer.pow(2, 10)  # รุ่น Elixir 1.12+
1024

# Bitwise operations
iex> 5 &&& 3    # AND
1

iex> 5 ||| 3    # OR
7

iex> 5 ^^^ 3    # XOR
6

iex> ~~~5       # NOT (bitwise complement)
-6

iex> 1 <<< 4    # Left shift
16

iex> 16 >>> 2   # Right shift
4
```

### 11.3 Float (ทศนิยม)

```elixir
# Float ธรรมดา
iex> 3.14
3.14

iex> -2.5
-2.5

iex> 1.0e10     # Scientific notation
10000000000.0

iex> 1.0e-10
1.0e-10

# Float Operations
iex> 1.5 + 2.5
4.0

iex> 3.0 * 2.0
6.0

iex> 10.0 / 3.0
3.3333333333333335

# Rounding
iex> Float.round(3.14159, 2)
3.14

iex> Float.ceil(3.1)
4.0

iex> Float.floor(3.9)
3.0

iex> trunc(3.7)    # ตัดทศนิยมทิ้ง
3

iex> round(3.5)    # ปัดเศษ
4
```

### 11.4 แปลง Integer ↔ Float

```elixir
# Integer → Float
iex> 42 * 1.0
42.0

iex> 42 + 0.0
42.0

# Float → Integer
iex> trunc(3.7)
3

iex> round(3.5)
4

iex> floor(3.9)      # ต้องใช้ Float.floor แล้วแปลง
3.0

iex> Float.floor(3.9) |> trunc()
3
```

---

## Step 12: Atom

Atom คือค่าคงที่ที่มีชื่อของตัวเอง เป็น concept ที่สำคัญมากใน Elixir

### 12.1 Atom พื้นฐาน

```elixir
# Atom ขึ้นต้นด้วย :
iex> :ok
:ok

iex> :error
:error

iex> :hello
:hello

iex> :my_atom
:my_atom

# Atom ที่มี space หรืออักขระพิเศษ ต้องใส่ใน quotes
iex> :"hello world"
:"hello world"

iex> :"atom with spaces and symbols!@#"
:"atom with spaces and symbols!@#"
```

### 12.2 Boolean เป็น Atom!

```elixir
# true และ false คือ Atom!
iex> true == :true
true

iex> false == :false
true

iex> nil == :nil
true

iex> is_atom(true)
true

iex> is_atom(false)
true

iex> is_atom(nil)
true
```

### 12.3 Atom ใช้ทำอะไร?

```elixir
# 1. Return values - บอกผลลัพธ์
def find_user(id) do
  case Database.get(id) do
    nil  -> {:error, :not_found}
    user -> {:ok, user}
  end
end

# 2. Pattern matching keys
{:ok, user} = find_user(1)
{:error, reason} = find_user(999)

# 3. Map keys (ใช้บ่อยมาก)
user = %{name: "Alice", age: 30, role: :admin}
#       ↑ shorthand สำหรับ {:name, "Alice"} เป็น atom key

# 4. Flags/States
status = :active
color = :red
direction = :north
```

### 12.4 Module Names ก็เป็น Atom

```elixir
# Module name = Atom
iex> is_atom(String)
true

iex> String == :"Elixir.String"
true

# เรียก function ผ่าน atom
iex> apply(String, :upcase, ["hello"])
"HELLO"
```

---

## Step 13: Boolean

### 13.1 Boolean Operations

```elixir
# Logical AND
iex> true and true
true

iex> true and false
false

iex> false and true
false

# Logical OR
iex> true or false
true

iex> false or false
false

# Logical NOT
iex> not true
false

iex> not false
true

# Short-circuit operators (ใช้กับ non-boolean ได้)
iex> 1 && 2       # ถ้า truthy ทั้งคู่ return ค่าขวา
2

iex> nil && 2     # ถ้าซ้ายเป็น falsy return ค่าซ้าย
nil

iex> 1 || false   # ถ้าซ้ายเป็น truthy return ค่าซ้าย
1

iex> nil || "default"  # ถ้าซ้ายเป็น falsy return ค่าขวา
"default"

iex> !true
false

iex> !nil
true
```

### 13.2 Truthy และ Falsy

```elixir
# ใน Elixir มีเพียง 2 ค่าที่เป็น falsy:
# - false
# - nil

# ทุกอย่างอื่นเป็น truthy!
iex> if 0, do: "truthy", else: "falsy"
"truthy"      # 0 เป็น truthy ใน Elixir!

iex> if "", do: "truthy", else: "falsy"
"truthy"      # "" เป็น truthy ใน Elixir!

iex> if [], do: "truthy", else: "falsy"
"truthy"      # [] เป็น truthy ใน Elixir!

iex> if nil, do: "truthy", else: "falsy"
"falsy"       # nil เป็น falsy

iex> if false, do: "truthy", else: "falsy"
"falsy"       # false เป็น falsy
```

### 13.3 Comparison Operators

```elixir
# ความเท่ากัน
iex> 1 == 1
true

iex> 1 == 1.0   # == เปรียบ value (ไม่สน type)
true

iex> 1 === 1.0  # === เปรียบทั้ง value และ type
false

iex> 1 !== 1.0
true

# การเปรียบขนาด
iex> 1 < 2
true

iex> 1 > 2
false

iex> 1 <= 1
true

iex> 1 >= 2
false

# เปรียบข้าม Type ได้! (Elixir มี type ordering)
iex> 1 < :atom
true

# Order: number < atom < reference < function < port < pid < tuple < map < list < bitstring
```

---

## Step 14: String

String ใน Elixir คือ **UTF-8 encoded binary**

### 14.1 String พื้นฐาน

```elixir
# String ใช้ double quotes เสมอ
iex> "Hello, World!"
"Hello, World!"

# Single quotes คือ Charlist (ไม่ใช่ String!)
iex> 'Hello'
'Hello'    # นี่คือ [72, 101, 108, 108, 111]

# Multi-line string
iex> """
...> This is
...> a multi-line
...> string
...> """
"This is\na multi-line\nstring\n"

# String Interpolation
iex> name = "Alice"
iex> "Hello, #{name}!"
"Hello, Alice!"

iex> "2 + 2 = #{2 + 2}"
"2 + 2 = 4"

# Escape sequences
iex> "Hello\nWorld"    # newline
"Hello\nWorld"

iex> "Tab\there"       # tab
"Tab\there"

iex> "Quote: \""       # double quote
"Quote: \""

iex> "Backslash: \\"   # backslash
"Backslash: \\"
```

### 14.2 String Functions ที่ต้องรู้

```elixir
# ความยาว (จำนวน grapheme cluster ไม่ใช่ byte)
iex> String.length("Hello")
5

iex> String.length("สวัสดี")
6

# แปลง case
iex> String.upcase("hello")
"HELLO"

iex> String.downcase("HELLO")
"hello"

iex> String.capitalize("hello world")
"Hello world"

# ตัด whitespace
iex> String.trim("  hello  ")
"hello"

iex> String.trim_leading("  hello  ")
"hello  "

iex> String.trim_trailing("  hello  ")
"  hello"

# แทนที่
iex> String.replace("Hello World", "World", "Elixir")
"Hello Elixir"

iex> String.replace("aababc", "a", "x")
"xxbxbc"

iex> String.replace("aababc", "a", "x", global: false)
"xababc"   # แทนที่แค่ครั้งแรก

# แบ่ง
iex> String.split("Hello World Elixir")
["Hello", "World", "Elixir"]

iex> String.split("a,b,c", ",")
["a", "b", "c"]

iex> String.split("a,b,c", ",", parts: 2)
["a", "b,c"]

# รวม
iex> Enum.join(["Hello", "World"], " ")
"Hello World"

# ตรวจสอบ
iex> String.contains?("Hello World", "World")
true

iex> String.starts_with?("Hello", "He")
true

iex> String.ends_with?("Hello", "lo")
true

# ดึงส่วน
iex> String.slice("Hello World", 0, 5)
"Hello"

iex> String.slice("Hello World", 6..-1)
"World"

# ตรวจสอบ type
iex> is_binary("Hello")
true

iex> is_binary(:atom)
false
```

### 14.3 String Concatenation

```elixir
# ใช้ <> operator
iex> "Hello" <> ", " <> "World!"
"Hello, World!"

# หรือใช้ interpolation (แนะนำ)
iex> name = "World"
iex> "Hello, #{name}!"
"Hello, World!"
```

### 14.4 Unicode Support

```elixir
# Elixir รองรับ Unicode เต็มรูปแบบ
iex> "สวัสดี"
"สวัสดี"

iex> String.length("สวัสดี")
6

iex> String.upcase("héllo")
"HÉLLO"

# Unicode codepoints
iex> ?A
65

iex> ?ก
3585

# String.codepoints/1
iex> String.codepoints("abc")
["a", "b", "c"]

# String.graphemes/1
iex> String.graphemes("abc")
["a", "b", "c"]
```

---

## Step 15: List

List คือ Linked List ใน Elixir

### 15.1 List พื้นฐาน

```elixir
# สร้าง List
iex> [1, 2, 3]
[1, 2, 3]

iex> ["hello", :world, 42, true]  # Mixed types
["hello", :world, 42, true]

iex> []    # Empty list
[]

# Head และ Tail
iex> [head | tail] = [1, 2, 3, 4, 5]
iex> head
1
iex> tail
[2, 3, 4, 5]

# เพิ่มด้านหน้า (prepend) - O(1)
iex> [0 | [1, 2, 3]]
[0, 1, 2, 3]

# เพิ่มด้านหลัง (append) - O(n) ช้ากว่า
iex> [1, 2, 3] ++ [4, 5]
[1, 2, 3, 4, 5]

# ลบ element
iex> [1, 2, 3, 2, 1] -- [2, 1]
[3, 2, 1]    # ลบแค่ครั้งแรกที่เจอ
```

### 15.2 List Operations

```elixir
# ความยาว
iex> length([1, 2, 3])
3

# ดึง element ตาม index (เริ่มจาก 0)
iex> Enum.at([1, 2, 3], 0)
1

iex> Enum.at([1, 2, 3], -1)   # Index ลบ = นับจากท้าย
3

# ตรวจสอบ element
iex> 2 in [1, 2, 3]
true

iex> 5 in [1, 2, 3]
false

# เรียงลำดับ
iex> Enum.sort([3, 1, 4, 1, 5, 9, 2, 6])
[1, 1, 2, 3, 4, 5, 6, 9]

iex> Enum.sort([3, 1, 4], :desc)
[4, 3, 1]

# กรอง
iex> Enum.filter([1, 2, 3, 4, 5], fn x -> x > 3 end)
[4, 5]

# แปลง
iex> Enum.map([1, 2, 3], fn x -> x * 2 end)
[2, 4, 6]

# รวมค่า
iex> Enum.reduce([1, 2, 3, 4, 5], 0, fn x, acc -> x + acc end)
15

# First และ Last
iex> List.first([1, 2, 3])
1

iex> List.last([1, 2, 3])
3

# Reverse
iex> Enum.reverse([1, 2, 3])
[3, 2, 1]

# Flatten
iex> List.flatten([[1, 2], [3, [4, 5]]])
[1, 2, 3, 4, 5]
```

### 15.3 Pattern Matching กับ List

```elixir
# Destructuring
iex> [a, b, c] = [1, 2, 3]
iex> a
1
iex> b
2
iex> c
3

# Head | Tail pattern
iex> [head | tail] = [1, 2, 3, 4, 5]
iex> head
1
iex> tail
[2, 3, 4, 5]

# Skip elements ด้วย _
iex> [_, second, _] = [1, 2, 3]
iex> second
2

# Nested List
iex> [[1, 2], [3, 4]] = [[1, 2], [3, 4]]
[[1, 2], [3, 4]]
```

---

## Step 16: Tuple

Tuple คือ fixed-size collection ที่เข้าถึงได้เร็ว

### 16.1 Tuple พื้นฐาน

```elixir
# สร้าง Tuple
iex> {1, 2, 3}
{1, 2, 3}

iex> {:ok, "success"}
{:ok, "success"}

iex> {:error, :not_found}
{:error, :not_found}

iex> {"Alice", 30, :admin}
{"Alice", 30, :admin}

# ดึง element
iex> elem({:ok, "hello"}, 0)
:ok

iex> elem({:ok, "hello"}, 1)
"hello"

# ขนาด
iex> tuple_size({1, 2, 3})
3

# แก้ไข element
iex> put_elem({:ok, "hello"}, 1, "world")
{:ok, "world"}

# แปลง tuple ↔ list
iex> Tuple.to_list({1, 2, 3})
[1, 2, 3]

iex> List.to_tuple([1, 2, 3])
{1, 2, 3}
```

### 16.2 Convention: {:ok, value} และ {:error, reason}

```elixir
# Pattern ที่ใช้บ่อยมากใน Elixir
case File.read("file.txt") do
  {:ok, content}  -> IO.puts("File content: #{content}")
  {:error, reason} -> IO.puts("Error: #{reason}")
end

# ฟังก์ชันที่ return tuple
def divide(a, b) when b != 0 do
  {:ok, a / b}
end

def divide(_, 0) do
  {:error, :division_by_zero}
end

# การใช้งาน
case divide(10, 2) do
  {:ok, result}    -> IO.puts("Result: #{result}")
  {:error, reason} -> IO.puts("Error: #{reason}")
end
```

### 16.3 Tuple vs List

```elixir
# Tuple - ใช้เมื่อ:
# 1. ขนาดคงที่
# 2. หลาย types ที่มีความหมาย
# 3. Pattern matching return values

{:ok, user} = authenticate(email, password)
{name, age, role} = {"Alice", 30, :admin}

# List - ใช้เมื่อ:
# 1. จำนวน elements ไม่แน่นอน
# 2. ต้องการ iterate
# 3. elements มี type เดียวกัน

users = ["Alice", "Bob", "Charlie"]
numbers = [1, 2, 3, 4, 5]
```

---

## Step 17: Map

Map คือ Key-Value data structure

### 17.1 Map พื้นฐาน

```elixir
# สร้าง Map
iex> %{name: "Alice", age: 30}
%{age: 30, name: "Alice"}

# Atom keys (shorthand)
iex> %{name: "Alice", age: 30}
# เหมือนกับ
iex> %{:name => "Alice", :age => 30}

# String keys
iex> %{"name" => "Alice", "age" => 30}

# Mixed keys
iex> %{:name => "Alice", "email" => "alice@example.com", 1 => :one}

# Nested Map
iex> %{
...>   user: %{
...>     name: "Alice",
...>     address: %{
...>       city: "Bangkok",
...>       country: "Thailand"
...>     }
...>   }
...> }
```

### 17.2 เข้าถึงข้อมูล

```elixir
iex> user = %{name: "Alice", age: 30, role: :admin}

# Dot notation (เฉพาะ Atom keys)
iex> user.name
"Alice"

iex> user.age
30

# [] notation (ทุก type ของ key)
iex> user[:name]
"Alice"

iex> user["name"]    # ถ้า key เป็น string ต้องใช้ []
nil                  # ถ้าไม่เจอ key จะ return nil (ไม่ error!)

# Map.get/2 และ Map.get/3
iex> Map.get(user, :name)
"Alice"

iex> Map.get(user, :phone)
nil

iex> Map.get(user, :phone, "N/A")  # default value
"N/A"

# Map.fetch/2
iex> Map.fetch(user, :name)
{:ok, "Alice"}

iex> Map.fetch(user, :phone)
:error

# Map.fetch!/2 - raise ถ้าไม่เจอ key
iex> Map.fetch!(user, :name)
"Alice"

iex> Map.fetch!(user, :phone)
# ** (KeyError) key :phone not found in: %{...}
```

### 17.3 แก้ไข Map

```elixir
iex> user = %{name: "Alice", age: 30}

# อัปเดต existing key (ใช้ | syntax)
iex> %{user | age: 31}
%{age: 31, name: "Alice"}

# เพิ่ม key ใหม่ (ต้องใช้ Map.put)
iex> Map.put(user, :email, "alice@example.com")
%{age: 30, email: "alice@example.com", name: "Alice"}

# ลบ key
iex> Map.delete(user, :age)
%{name: "Alice"}

# รวม Maps
iex> Map.merge(%{a: 1}, %{b: 2})
%{a: 1, b: 2}

iex> Map.merge(%{a: 1, b: 2}, %{b: 3, c: 4})
%{a: 1, b: 3, c: 4}    # key ซ้ำ: Map ขวาชนะ
```

### 17.4 Pattern Matching กับ Map

```elixir
# ดึงค่าจาก Map
iex> %{name: name, age: age} = %{name: "Alice", age: 30}
iex> name
"Alice"
iex> age
30

# ตรวจสอบว่ามี key หรือไม่
iex> %{name: name} = %{name: "Alice", age: 30, role: :admin}
iex> name
"Alice"
# Map ที่มี keys มากกว่าก็ match ได้

# ใน function parameters
def greet(%{name: name, role: :admin}) do
  "Welcome back, Admin #{name}!"
end

def greet(%{name: name}) do
  "Hello, #{name}!"
end
```

---

## Step 18: Keyword List

Keyword List คือ List ของ tuples ที่ key เป็น Atom

### 18.1 Keyword List พื้นฐาน

```elixir
# Keyword List
iex> [name: "Alice", age: 30]
[name: "Alice", age: 30]

# เหมือนกับ
iex> [{:name, "Alice"}, {:age, 30}]
[name: "Alice", age: 30]

# ต่างจาก Map คือ:
# 1. เก็บลำดับ
# 2. key ซ้ำได้
iex> [a: 1, a: 2, b: 3]
[a: 1, a: 2, b: 3]    # มี :a สองตัว

# เข้าถึงข้อมูล
iex> kw = [name: "Alice", age: 30]
iex> kw[:name]
"Alice"

iex> Keyword.get(kw, :name)
"Alice"

iex> Keyword.get(kw, :phone, "N/A")
"N/A"
```

### 18.2 เมื่อไหร่ใช้ Keyword List vs Map

```elixir
# Keyword List ใช้สำหรับ:
# 1. Function options (แบบ named parameters)
String.split("hello world", " ", trim: true, parts: 2)
#                                  ↑↑↑↑↑↑↑↑ keyword list options

# 2. DSL (Domain Specific Language)
defmodule Router do
  use Plug.Router
  
  get "/", do: send_resp(conn, 200, "Hello")
  # do: ... คือ keyword list!
end

# Map ใช้สำหรับ:
# 1. ข้อมูลที่ต้องการ access ด้วย key
user = %{name: "Alice", email: "alice@example.com"}

# 2. เมื่อ key ซ้ำไม่ได้
# 3. เมื่อต้องการ performance ที่ดีกว่า
```

---

## Step 19: ประเภทข้อมูลพิเศษ

### 19.1 Nil

```elixir
# nil แทน "ไม่มีค่า"
iex> nil
nil

iex> is_nil(nil)
true

iex> is_nil(0)
false

iex> is_nil(false)
false

# nil ใช้เป็น default value
iex> user = %{name: "Alice"}
iex> user[:age]       # key ที่ไม่มีใน Map
nil

# Guard against nil
def process(nil), do: {:error, :no_value}
def process(value), do: {:ok, value}
```

### 19.2 Range

```elixir
# สร้าง Range
iex> 1..10
1..10

iex> 1..10//2    # step (Elixir 1.12+)
1..10//2

# ใช้งาน Range
iex> Enum.to_list(1..5)
[1, 2, 3, 4, 5]

iex> Enum.map(1..5, fn x -> x * 2 end)
[2, 4, 6, 8, 10]

iex> 3 in 1..10
true

iex> Enum.sum(1..100)
5050

# Reverse range
iex> Enum.to_list(10..1//-1)
[10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
```

### 19.3 Charlist

```elixir
# Charlist ใช้ single quotes
# คือ List ของ integer (Unicode codepoints)
iex> 'hello'
'hello'

iex> [72, 101, 108, 108, 111]
'Hello'

# แปลง Charlist ↔ String
iex> to_string('hello')
"hello"

iex> to_charlist("hello")
'hello'

# ใช้ Charlist เมื่อ:
# 1. ทำงานกับ Erlang libraries (บางอันต้องการ Charlist)
# 2. ทำงานกับ character-level operations
```

### 19.4 Binary และ Bitstring

```elixir
# Binary (String = Binary)
iex> is_binary("hello")
true

iex> byte_size("hello")
5

# Bitstring syntax
iex> <<1, 2, 3>>
<<1, 2, 3>>

iex> <<255>>
<<255>>

# Binary patterns
iex> <<r, g, b>> = <<255, 128, 0>>
iex> r
255
iex> g
128
iex> b
0

# String ก็คือ Binary
iex> <<104, 101, 108, 108, 111>>
"hello"
```

---

## Step 20: Type Checking Functions

### 20.1 is_* Functions

```elixir
# ตรวจสอบ type
iex> is_integer(42)
true

iex> is_float(3.14)
true

iex> is_number(42)       # integer หรือ float
true

iex> is_number(3.14)
true

iex> is_atom(:ok)
true

iex> is_boolean(true)
true

iex> is_binary("hello")
true

iex> is_bitstring(<<1, 2>>)
true

iex> is_list([1, 2, 3])
true

iex> is_tuple({1, 2})
true

iex> is_map(%{a: 1})
true

iex> is_nil(nil)
true

iex> is_function(fn x -> x end)
true

iex> is_pid(self())
true
```

### 20.2 typeof Equivalent

```elixir
# ไม่มี typeof ใน Elixir แต่ใช้ i/1 ใน IEx
iex> i "hello"
# Term: "hello"
# Data type: BitString
# ...

# หรือ inspect type ด้วย
defmodule TypeOf do
  def typeof(x) when is_integer(x), do: :integer
  def typeof(x) when is_float(x),   do: :float
  def typeof(x) when is_atom(x),    do: :atom
  def typeof(x) when is_binary(x),  do: :binary
  def typeof(x) when is_list(x),    do: :list
  def typeof(x) when is_tuple(x),   do: :tuple
  def typeof(x) when is_map(x),     do: :map
  def typeof(x) when is_nil(x),     do: :nil
  def typeof(_),                     do: :unknown
end

TypeOf.typeof("hello")  # :binary
TypeOf.typeof(42)       # :integer
TypeOf.typeof([1,2,3])  # :list
```

### 20.3 Inspect

```elixir
# inspect/1 แปลงค่าเป็น String representation
iex> inspect(42)
"42"

iex> inspect([1, 2, 3])
"[1, 2, 3]"

iex> inspect(%{name: "Alice"})
"%{name: \"Alice\"}"

# ใช้ใน IO.puts
iex> IO.puts(inspect([1, 2, 3]))
[1, 2, 3]
:ok

# IO.inspect - พิมพ์แล้ว return ค่าเดิม (ดีสำหรับ debugging)
iex> [1, 2, 3] |> IO.inspect(label: "list") |> Enum.sum()
list: [1, 2, 3]
6
```

---

## สรุป Part 02

ใน Part นี้คุณได้เรียนรู้ประเภทข้อมูลพื้นฐานทั้งหมดของ Elixir:

✅ **Step 11** - Integer และ Float  
✅ **Step 12** - Atom (รวมถึง true, false, nil เป็น Atom)  
✅ **Step 13** - Boolean และ Comparison Operators  
✅ **Step 14** - String (UTF-8 Binary)  
✅ **Step 15** - List (Linked List)  
✅ **Step 16** - Tuple  
✅ **Step 17** - Map (Key-Value)  
✅ **Step 18** - Keyword List  
✅ **Step 19** - Nil, Range, Charlist, Binary  
✅ **Step 20** - Type Checking Functions  

### ตารางสรุปประเภทข้อมูล

| Type | ตัวอย่าง | ใช้เมื่อ |
|------|---------|---------|
| Integer | `42`, `-1`, `0xFF` | ตัวเลขจำนวนเต็ม |
| Float | `3.14`, `1.0e10` | ตัวเลขทศนิยม |
| Atom | `:ok`, `:error`, `true` | ค่าคงที่ที่มีชื่อ |
| String | `"hello"` | ข้อความ |
| List | `[1, 2, 3]` | Collections ขนาดแปรผัน |
| Tuple | `{:ok, value}` | Collections ขนาดคงที่ |
| Map | `%{key: value}` | Key-value pairs |
| Keyword List | `[key: value]` | Options/Named params |

---

## แบบฝึกหัด Part 02

1. สร้าง Map ที่เก็บข้อมูลของคุณ: name, age, city, favorite_languages (list)
2. ลองใช้ String functions: upcase, split, trim, contains?
3. สร้าง function ที่รับ List ของตัวเลข แล้วคืน sum, min, max
4. ทดลองใช้ Pattern matching กับ Tuple {:ok, value}
5. สร้าง Range 1..100 แล้วหาผลรวมทุกเลขคี่

---

## ไปต่อ

➡️ [Part 03: Pattern Matching](./part-03-pattern-matching.md)
