# Part 06: Recursion และ Loops (Steps 51-60)

## Step 51: ทำไม Elixir ไม่มี For Loop แบบ Imperative?

### 51.1 Immutability ทำให้ Loop แบบ Imperative เป็นไปไม่ได้

```python
# Python - ใช้ mutation
total = 0
for x in [1, 2, 3, 4, 5]:
    total += total  # เปลี่ยนค่า total ตลอด
```

```elixir
# Elixir - ไม่สามารถ mutate ตัวแปรได้
# นี่ไม่ทำงานแบบที่คิด:
total = 0
# ไม่มีทางเปลี่ยนค่า total ใน "loop"

# วิธีที่ถูกต้องใน Elixir:
# 1. ใช้ Enum functions
total = Enum.sum([1, 2, 3, 4, 5])  # 15

# 2. ใช้ recursion
def sum([]), do: 0
def sum([h | t]), do: h + sum(t)

# 3. ใช้ reduce
Enum.reduce([1, 2, 3, 4, 5], 0, fn x, acc -> x + acc end)
```

### 51.2 Recursion คือ Loop ของ Elixir

```
loop (imperative):          recursion (functional):
┌──────────────────┐         ┌─────────────────────────┐
│ i = 0            │         │ sum([])    = 0            │
│ total = 0        │         │ sum([h|t]) = h + sum(t)   │
│ while i < n:     │         │                           │
│   total += a[i]  │    ↔    │ BEAM optimizes tail       │
│   i++            │         │ recursion = no stack      │
│ return total     │         │ overflow!                  │
└──────────────────┘         └─────────────────────────┘
```

---

## Step 52: Tail Recursion และ Tail Call Optimization

### 52.1 Non-tail Recursion

```elixir
# Non-tail recursive - stack frame ค้างอยู่
def sum([]), do: 0
def sum([head | tail]) do
  head + sum(tail)  # ← ต้องรอ sum(tail) ก่อนจึงจะบวกได้
  # Call stack เมื่อ sum([1,2,3])
  # sum([1,2,3]) = 1 + sum([2,3])
  #                    = 2 + sum([3])
  #                         = 3 + sum([])
  #                              = 0
  # Stack size = O(n)
end
```

### 52.2 Tail-recursive Version

```elixir
# Tail recursive - BEAM optimize เป็น loop
def sum_tr(list, acc \\ 0)
def sum_tr([], acc), do: acc
def sum_tr([head | tail], acc) do
  sum_tr(tail, acc + head)  # ← recursive call คือ LAST operation
  # Call stack เมื่อ sum_tr([1,2,3])
  # sum_tr([1,2,3], 0)
  # sum_tr([2,3], 1)
  # sum_tr([3], 3)
  # sum_tr([], 6)
  # 6
  # Stack size = O(1)! BEAM reuses stack frame
end
```

### 52.3 เปรียบเทียบ Performance

```elixir
# ทดสอบกับ list ขนาดใหญ่
large_list = Enum.to_list(1..1_000_000)

# Non-tail recursive - อาจ stack overflow
# sum(large_list)  # DANGEROUS!

# Tail recursive - ปลอดภัย
sum_tr(large_list)  # ทำงานได้ปกติ

# Enum.sum ก็ใช้ tail recursion
Enum.sum(large_list)  # เร็วที่สุดเพราะ built-in
```

---

## Step 53: Accumulator Pattern

Pattern ที่ใช้บ่อยที่สุดใน recursion

### 53.1 พื้นฐาน Accumulator

```elixir
defmodule MyList do
  # สร้าง list reversed ด้วย accumulator
  def reverse(list, acc \\ [])
  def reverse([], acc), do: acc
  def reverse([head | tail], acc) do
    reverse(tail, [head | acc])  # เพิ่ม head ไปหน้า acc
  end
end

MyList.reverse([1, 2, 3, 4, 5])
# reverse([1,2,3,4,5], [])
# reverse([2,3,4,5], [1])
# reverse([3,4,5], [2,1])
# reverse([4,5], [3,2,1])
# reverse([5], [4,3,2,1])
# reverse([], [5,4,3,2,1])
# [5,4,3,2,1]
```

### 53.2 Map ด้วย Accumulator

```elixir
defmodule MyEnum do
  def map(list, func, acc \\ [])
  def map([], _, acc), do: Enum.reverse(acc)  # ต้อง reverse เพราะใส่หน้า
  def map([head | tail], func, acc) do
    map(tail, func, [func.(head) | acc])
  end
end

MyEnum.map([1, 2, 3], fn x -> x * 2 end)
# [2, 4, 6]
```

### 53.3 Complex Accumulator

```elixir
defmodule Stats do
  def calculate(numbers) do
    calculate(numbers, %{sum: 0, count: 0, min: nil, max: nil})
  end
  
  defp calculate([], acc) do
    avg = if acc.count > 0, do: acc.sum / acc.count, else: 0
    Map.put(acc, :average, avg)
  end
  
  defp calculate([head | tail], acc) do
    new_acc = %{
      sum: acc.sum + head,
      count: acc.count + 1,
      min: if(acc.min == nil or head < acc.min, do: head, else: acc.min),
      max: if(acc.max == nil or head > acc.max, do: head, else: acc.max)
    }
    calculate(tail, new_acc)
  end
end

Stats.calculate([3, 1, 4, 1, 5, 9, 2, 6])
# %{sum: 31, count: 8, min: 1, max: 9, average: 3.875}
```

---

## Step 54: Mutual Recursion

### 54.1 Functions ที่เรียกกัน

```elixir
defmodule Mutual do
  # is_even และ is_odd เรียกกันไปมา
  def is_even(0), do: true
  def is_even(n) when n > 0, do: is_odd(n - 1)
  
  def is_odd(0), do: false
  def is_odd(n) when n > 0, do: is_even(n - 1)
end

Mutual.is_even(4)  # true
Mutual.is_odd(3)   # true

# Note: นี่ไม่ใช่ tail recursive (มีค่าค้างอยู่)
# สำหรับ production ใช้ rem/2 ดีกว่า:
# is_even(n) = rem(n, 2) == 0
```

### 54.2 Tree Traversal

```elixir
defmodule Tree do
  # Tree structure: {:node, value, left, right} หรือ :leaf
  
  def sum(:leaf), do: 0
  def sum({:node, value, left, right}) do
    value + sum(left) + sum(right)
  end
  
  def depth(:leaf), do: 0
  def depth({:node, _, left, right}) do
    1 + max(depth(left), depth(right))
  end
  
  def to_list(:leaf), do: []
  def to_list({:node, value, left, right}) do
    to_list(left) ++ [value] ++ to_list(right)
  end
end

tree = {:node, 1,
  {:node, 2,
    {:node, 4, :leaf, :leaf},
    {:node, 5, :leaf, :leaf}
  },
  {:node, 3,
    :leaf,
    {:node, 6, :leaf, :leaf}
  }
}

Tree.sum(tree)     # 21
Tree.depth(tree)   # 3
Tree.to_list(tree) # [4, 2, 5, 1, 3, 6] (in-order)
```

---

## Step 55: Enum Module ลึกขึ้น

Enum Module มี functions สำหรับ collections ครบถ้วน

### 55.1 การสร้างและแปลง

```elixir
# to_list - แปลงเป็น list
Enum.to_list(1..5)        # [1, 2, 3, 4, 5]
Enum.to_list(%{a: 1})     # [{:a, 1}]

# chunk_by - แบ่งกลุ่มตาม condition
[3, 1, 2, 1, 5, 5]
|> Enum.chunk_by(&(&1))
# [[3], [1], [2], [1], [5, 5]]

# chunk_every - แบ่งเป็น chunks ขนาดเท่ากัน
Enum.chunk_every([1, 2, 3, 4, 5], 2)
# [[1, 2], [3, 4], [5]]

Enum.chunk_every([1, 2, 3, 4, 5], 2, 1)
# [[1, 2], [2, 3], [3, 4], [4, 5], [5]]

# flat_map - map แล้ว flatten
Enum.flat_map([1, 2, 3], fn x -> [x, x * 2] end)
# [1, 2, 2, 4, 3, 6]

# zip - รวม lists
Enum.zip([1, 2, 3], ["a", "b", "c"])
# [{1, "a"}, {2, "b"}, {3, "c"}]

Enum.zip_with([1, 2, 3], [4, 5, 6], fn a, b -> a + b end)
# [5, 7, 9]
```

### 55.2 การค้นหา

```elixir
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# find - หา element แรกที่ match
Enum.find(numbers, fn x -> x > 5 end)  # 6
Enum.find(numbers, 0, fn x -> x > 100 end)  # 0 (default)

# find_index
Enum.find_index(numbers, fn x -> x > 5 end)  # 5 (index)

# find_value - หาแล้ว transform
Enum.find_value(numbers, fn x ->
  if x > 5, do: x * 10, else: nil
end)
# 60

# any? - มีอย่างน้อยหนึ่งที่ match
Enum.any?(numbers, fn x -> x > 5 end)   # true
Enum.any?(numbers, fn x -> x > 100 end) # false

# all? - ทุกตัว match
Enum.all?(numbers, fn x -> x > 0 end)   # true
Enum.all?(numbers, fn x -> x > 5 end)   # false

# none? - ไม่มีตัวไหน match
Enum.none?(numbers, fn x -> x > 100 end)  # true

# count กับ condition
Enum.count(numbers, fn x -> rem(x, 2) == 0 end)  # 5

# member?
Enum.member?(numbers, 5)    # true
Enum.member?(numbers, 15)   # false
5 in numbers                # true (shorthand)
```

### 55.3 การเรียงลำดับ

```elixir
# sort
Enum.sort([3, 1, 4, 1, 5, 9])  # [1, 1, 3, 4, 5, 9]
Enum.sort([3, 1, 4], :desc)    # [4, 3, 1]

# sort_by - sort ด้วย key function
users = [
  %{name: "Charlie", age: 35},
  %{name: "Alice", age: 30},
  %{name: "Bob", age: 25}
]

Enum.sort_by(users, & &1.age)
# [Bob(25), Alice(30), Charlie(35)]

Enum.sort_by(users, & &1.name)
# [Alice, Bob, Charlie]

Enum.sort_by(users, & &1.age, :desc)
# [Charlie(35), Alice(30), Bob(25)]

# min, max, min_max
Enum.min([3, 1, 4, 1, 5])    # 1
Enum.max([3, 1, 4, 1, 5])    # 5
Enum.min_max([3, 1, 4, 1, 5]) # {1, 5}

Enum.min_by(users, & &1.age)  # %{name: "Bob", age: 25}
Enum.max_by(users, & &1.age)  # %{name: "Charlie", age: 35}
```

### 55.4 Grouping

```elixir
# group_by
Enum.group_by([1, 2, 3, 4, 5, 6], fn x -> rem(x, 2) end)
# %{0 => [2, 4, 6], 1 => [1, 3, 5]}

users = [
  %{name: "Alice", role: :admin},
  %{name: "Bob", role: :user},
  %{name: "Charlie", role: :admin},
  %{name: "Dave", role: :user}
]

Enum.group_by(users, & &1.role)
# %{
#   admin: [%{name: "Alice"...}, %{name: "Charlie"...}],
#   user:  [%{name: "Bob"...}, %{name: "Dave"...}]
# }

# frequencies
Enum.frequencies(["apple", "banana", "apple", "cherry", "banana", "apple"])
# %{"apple" => 3, "banana" => 2, "cherry" => 1}

Enum.frequencies_by([1, 2, 3, 4, 5, 6], fn x -> rem(x, 2) == 0 end)
# %{false => 3, true => 3}
```

---

## Step 56: Stream Module

Stream ใช้สำหรับ lazy evaluation - ไม่คำนวณจนกว่าจะต้องการผล

### 56.1 Stream vs Enum

```elixir
# Enum - eager evaluation (คำนวณทันที)
[1, 2, 3, 4, 5]
|> Enum.map(fn x -> 
     IO.puts("mapping #{x}")
     x * 2 
   end)
|> Enum.filter(fn x -> x > 4 end)
# Output: "mapping 1", "mapping 2", "mapping 3", "mapping 4", "mapping 5"
# แล้วค่อย filter → [6, 8, 10]
# สร้าง intermediate lists ทุกขั้นตอน

# Stream - lazy evaluation  
[1, 2, 3, 4, 5]
|> Stream.map(fn x -> 
     IO.puts("mapping #{x}")
     x * 2 
   end)
|> Stream.filter(fn x -> x > 4 end)
|> Enum.to_list()
# Output: "mapping 1", "mapping 2", "mapping 3", "mapping 4", "mapping 5"
# คำนวณทีละ element, ไม่สร้าง intermediate lists
```

### 56.2 Stream สำหรับ Large Data

```elixir
# อ่านไฟล์ขนาดใหญ่โดยไม่โหลดทั้งหมดเข้า memory
File.stream!("large_file.txt")
|> Stream.map(&String.trim/1)
|> Stream.filter(&(String.length(&1) > 0))
|> Stream.take(100)    # เอาแค่ 100 บรรทัดแรก
|> Enum.to_list()

# Process CSV ขนาดใหญ่
File.stream!("data.csv")
|> Stream.drop(1)      # skip header
|> Stream.map(&String.split(&1, ","))
|> Stream.filter(fn [id | _] -> String.to_integer(id) > 1000 end)
|> Enum.each(fn row ->
     process_row(row)
   end)
```

### 56.3 Infinite Streams

```elixir
# Stream.iterate - สร้าง sequence อนันต์
Stream.iterate(0, fn x -> x + 1 end)
|> Enum.take(10)
# [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# Fibonacci sequence
Stream.iterate({0, 1}, fn {a, b} -> {b, a + b} end)
|> Stream.map(fn {a, _} -> a end)
|> Enum.take(10)
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Stream.unfold
Stream.unfold(0, fn n -> {n, n + 1} end)
|> Enum.take(5)
# [0, 1, 2, 3, 4]

# Powers of 2
Stream.unfold(1, fn n -> {n, n * 2} end)
|> Enum.take(10)
# [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]
```

### 56.4 Stream.resource

```elixir
# สำหรับ resources ที่ต้องเปิด/ปิด
def read_database_rows(query) do
  Stream.resource(
    fn -> Database.open_cursor(query) end,  # เปิด resource
    fn cursor ->
      case Database.fetch_row(cursor) do
        {:ok, row} -> {[row], cursor}  # emit row
        :done      -> {:halt, cursor}  # หยุด
      end
    end,
    fn cursor -> Database.close_cursor(cursor) end  # cleanup
  )
end

read_database_rows("SELECT * FROM users")
|> Stream.filter(& &1.active)
|> Stream.map(& &1.name)
|> Enum.to_list()
```

---

## Step 57: Reduce Deep Dive

`reduce` เป็น function ที่ทรงพลังที่สุดใน Enum

### 57.1 Reduce พื้นฐาน

```elixir
# ทุก Enum function สามารถเขียนด้วย reduce ได้

# sum
Enum.reduce([1, 2, 3, 4, 5], 0, fn x, acc -> x + acc end)  # 15

# map
Enum.reduce([1, 2, 3], [], fn x, acc -> 
  acc ++ [x * 2]  # ไม่ efficient แต่แสดง concept
end)
# [2, 4, 6]

# filter
Enum.reduce([1, 2, 3, 4, 5], [], fn x, acc ->
  if x > 3, do: acc ++ [x], else: acc
end)
# [4, 5]
```

### 57.2 Reduce กับ Complex State

```elixir
# Word frequency counter
words = String.split("the quick brown fox jumps over the lazy dog the")

word_count = Enum.reduce(words, %{}, fn word, acc ->
  Map.update(acc, word, 1, fn count -> count + 1 end)
end)

# %{"brown" => 1, "dog" => 1, "fox" => 1, "jumps" => 1,
#   "lazy" => 1, "over" => 1, "quick" => 1, "the" => 3}

# สร้าง index
records = [
  %{id: 1, name: "Alice"},
  %{id: 2, name: "Bob"},
  %{id: 3, name: "Charlie"}
]

index = Enum.reduce(records, %{}, fn record, acc ->
  Map.put(acc, record.id, record)
end)

# %{1 => %{id: 1, name: "Alice"}, ...}
index[2]  # %{id: 2, name: "Bob"}
```

### 57.3 Enum.reduce_while

```elixir
# หยุด reduce เมื่อต้องการ
result = Enum.reduce_while(1..100, 0, fn x, acc ->
  if acc + x > 100 do
    {:halt, acc}   # หยุดแล้ว return acc ปัจจุบัน
  else
    {:cont, acc + x}  # ทำต่อ
  end
end)

# result = 91 (1+2+...+13=91, เพิ่ม 14 จะเกิน 100)
```

---

## Step 58: List Comprehensions ขั้นสูง

### 58.1 Nested Comprehensions

```elixir
# Matrix flattening
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

for row <- matrix, element <- row, do: element
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Matrix transpose
for j <- 0..2 do
  for i <- 0..2 do
    Enum.at(Enum.at(matrix, i), j)
  end
end
# [[1, 4, 7], [2, 5, 8], [3, 6, 9]]
```

### 58.2 Comprehension กับ Binary

```elixir
# ดึงข้อมูลจาก binary
pixels = <<213, 45, 132, 64, 76, 32, 76, 0, 0, 234, 32, 15>>

for <<r::8, g::8, b::8 <- pixels>> do
  {r, g, b}
end
# [{213, 45, 132}, {64, 76, 32}, {76, 0, 0}, {234, 32, 15}]
```

### 58.3 Comprehension เป็น Generator

```elixir
# สร้าง combination
suits  = [:spades, :hearts, :diamonds, :clubs]
values = [2, 3, 4, 5, 6, 7, 8, 9, 10, :jack, :queen, :king, :ace]

deck = for suit <- suits, value <- values, do: {value, suit}
# [{2, :spades}, {2, :hearts}, ..., {:ace, :clubs}]
length(deck)  # 52

# Pythagorean triples
for a <- 1..20, b <- a..20, c <- b..20, a*a + b*b == c*c, do: {a, b, c}
# [{3, 4, 5}, {5, 12, 13}, {6, 8, 10}, {8, 15, 17}, {9, 12, 15}, 
#  {12, 16, 20}, {15, 20, 25}]
```

---

## Step 59: Practical Recursion Patterns

### 59.1 Flatten Nested Structure

```elixir
defmodule Flatten do
  def flatten([]), do: []
  def flatten([head | tail]) when is_list(head) do
    flatten(head) ++ flatten(tail)
  end
  def flatten([head | tail]) do
    [head | flatten(tail)]
  end
end

Flatten.flatten([1, [2, [3, 4], 5], [6, 7]])
# [1, 2, 3, 4, 5, 6, 7]
```

### 59.2 Tree Operations

```elixir
defmodule FileSystem do
  # จำลอง File System ด้วย nested structure
  def list_files({:dir, _, children}) do
    Enum.flat_map(children, &list_files/1)
  end
  
  def list_files({:file, name, _size}) do
    [name]
  end
  
  def total_size({:dir, _, children}) do
    Enum.sum(Enum.map(children, &total_size/1))
  end
  
  def total_size({:file, _, size}) do
    size
  end
  
  def find_file(tree, name) do
    case tree do
      {:file, ^name, _} ->
        {:found, tree}
      
      {:dir, _, children} ->
        Enum.find_value(children, :not_found, fn child ->
          case find_file(child, name) do
            :not_found -> nil
            result     -> result
          end
        end)
      
      _ ->
        :not_found
    end
  end
end

fs = {:dir, "/",
  [
    {:dir, "home",
      [
        {:dir, "alice",
          [
            {:file, "resume.pdf", 1024},
            {:file, "photo.jpg", 2048}
          ]
        }
      ]
    },
    {:dir, "var",
      [
        {:file, "log.txt", 512}
      ]
    }
  ]
}

FileSystem.list_files(fs)
# ["resume.pdf", "photo.jpg", "log.txt"]

FileSystem.total_size(fs)
# 3584

FileSystem.find_file(fs, "resume.pdf")
# {:found, {:file, "resume.pdf", 1024}}
```

### 59.3 String Parsing ด้วย Recursion

```elixir
defmodule CSVParser do
  def parse(csv_string) do
    csv_string
    |> String.split("\n")
    |> Enum.filter(&(String.trim(&1) != ""))
    |> parse_rows()
  end
  
  defp parse_rows(rows, acc \\ [])
  defp parse_rows([], acc), do: Enum.reverse(acc)
  defp parse_rows([row | rest], acc) do
    parsed = parse_row(row)
    parse_rows(rest, [parsed | acc])
  end
  
  defp parse_row(row) do
    row
    |> String.split(",")
    |> Enum.map(&String.trim/1)
  end
end

csv = """
Alice,30,Bangkok
Bob,25,Chiang Mai
Charlie,35,Phuket
"""

CSVParser.parse(csv)
# [["Alice", "30", "Bangkok"],
#  ["Bob", "25", "Chiang Mai"],
#  ["Charlie", "35", "Phuket"]]
```

---

## Step 60: Performance Considerations

### 60.1 List Operations Complexity

```elixir
# O(1) - เร็ว
[head | tail] = list           # Pattern match
[new_element | existing_list]  # Prepend

# O(n) - ช้ากว่า (ต้อง traverse ทั้ง list)
list ++ [new_element]          # Append
length(list)                   # Count
Enum.at(list, n)               # Index access
```

### 60.2 เมื่อไหร่ใช้อะไร

```elixir
# ใช้ List เมื่อ:
# - ต้องการ prepend บ่อยๆ
# - Sequential access
# - Pattern matching ด้วย [head | tail]

# ใช้ Tuple เมื่อ:
# - ขนาดคงที่
# - Fast element access ด้วย index
# - Return values {:ok, value}

# ใช้ Map เมื่อ:
# - Key-value access
# - O(log n) access time

# ใช้ ETS (ในอนาคต) เมื่อ:
# - ต้องการ shared mutable state
# - Large data ที่ต้องการ fast access

# ใช้ Stream เมื่อ:
# - Large/infinite data
# - Pipeline ที่ไม่ต้องการ intermediate results
```

### 60.3 Optimization Tips

```elixir
# ไม่ดี - build list แล้ว reverse
def build_list(n, acc \\ [])
def build_list(0, acc), do: acc  # ลืม reverse!
def build_list(n, acc) do
  build_list(n - 1, [n | acc])
end
# [1, 2, 3, ...n] - ลำดับถูก แต่ไม่ใส่ reverse

# ดี - prepend แล้ว reverse ตอนสุดท้าย
def build_list(n, acc \\ [])
def build_list(0, acc), do: Enum.reverse(acc)
def build_list(n, acc) do
  build_list(n - 1, [n | acc])
end

# ดีกว่า - ใช้ Enum ที่ optimize แล้ว
def build_list(n) do
  Enum.to_list(1..n)
end
```

---

## สรุป Part 06: Recursion และ Loops

✅ **Step 51** - ทำไม Elixir ไม่มี for loop  
✅ **Step 52** - Tail recursion และ TCO  
✅ **Step 53** - Accumulator pattern  
✅ **Step 54** - Mutual recursion  
✅ **Step 55** - Enum module ลึกขึ้น  
✅ **Step 56** - Stream module  
✅ **Step 57** - Reduce ขั้นสูง  
✅ **Step 58** - List comprehensions  
✅ **Step 59** - Real-world recursion  
✅ **Step 60** - Performance considerations  

---

## แบบฝึกหัด Part 06

1. เขียน `flatten/1` ที่ทำงานได้กับ arbitrarily nested lists
2. สร้าง Word Count จาก paragraph ด้วย `reduce`
3. สร้าง infinite Stream ของ prime numbers
4. Implement `zip/2` ด้วย recursion (ไม่ใช้ Enum.zip)
5. สร้าง function `deep_sum/1` ที่รวมตัวเลขใน nested list

---

## ไปต่อ

➡️ [Part 07: Enumerables และ Streams](./part-07-enumerables.md)
