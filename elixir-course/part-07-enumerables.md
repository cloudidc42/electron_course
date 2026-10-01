# Part 07: Enumerables และ Streams (Steps 61-70)

## Step 61: Enumerable Protocol

Enumerable คือ Protocol ที่ทำให้ data type สามารถใช้กับ Enum ได้

### 61.1 ทุกอย่างที่ implements Enumerable

```elixir
# Types ที่ implement Enumerable:
# List
Enum.count([1, 2, 3])     # 3

# Map (iterate เป็น {key, value} tuples)
Enum.count(%{a: 1, b: 2}) # 2
Enum.map(%{a: 1, b: 2}, fn {k, v} -> {k, v * 2} end)
# [a: 2, b: 4]

# Range
Enum.sum(1..100)  # 5050

# MapSet
Enum.member?(MapSet.new([1, 2, 3]), 2)  # true

# Keyword List
Enum.map([a: 1, b: 2], fn {k, v} -> {k, v + 10} end)
# [a: 11, b: 12]
```

### 61.2 Custom Enumerable

```elixir
defmodule NumberRange do
  defstruct [:from, :to, :step]
  
  # สร้าง struct
  def new(from, to, step \\ 1) do
    %NumberRange{from: from, to: to, step: step}
  end
end

# Implement Enumerable Protocol
defimpl Enumerable, for: NumberRange do
  def count(%{from: from, to: to, step: step}) do
    count = div(to - from, step) + 1
    {:ok, max(0, count)}
  end
  
  def member?(%{from: from, to: to, step: step}, value) do
    is_member = value >= from and value <= to and 
                rem(value - from, step) == 0
    {:ok, is_member}
  end
  
  def slice(_enum) do
    {:error, __MODULE__}
  end
  
  def reduce(%{from: from, to: to, step: step}, acc, fun) do
    reduce_range(from, to, step, acc, fun)
  end
  
  defp reduce_range(_current, _to, _step, {:halt, acc}, _fun) do
    {:halted, acc}
  end
  
  defp reduce_range(current, to, step, {:suspend, acc}, fun) do
    {:suspended, acc, fn new_acc ->
      reduce_range(current, to, step, new_acc, fun)
    end}
  end
  
  defp reduce_range(current, to, _step, {:cont, acc}, _fun) 
    when current > to do
    {:done, acc}
  end
  
  defp reduce_range(current, to, step, {:cont, acc}, fun) do
    reduce_range(current + step, to, step, fun.(current, acc), fun)
  end
end

# ใช้งาน
range = NumberRange.new(1, 10, 2)
Enum.to_list(range)   # [1, 3, 5, 7, 9]
Enum.sum(range)       # 25
Enum.member?(range, 5) # true
Enum.map(range, &(&1 * 2))  # [2, 6, 10, 14, 18]
```

---

## Step 62: Enum.map, filter, reduce Deep Dive

### 62.1 map ขั้นสูง

```elixir
# map กับ index
Enum.with_index([10, 20, 30])
|> Enum.map(fn {value, index} -> "#{index}: #{value}" end)
# ["0: 10", "1: 20", "2: 30"]

# หรือใช้ map_with_index
Enum.map_with_index([10, 20, 30], fn value, index ->
  "#{index}: #{value}"
end)
# ["0: 10", "1: 20", "2: 30"]

# map กับ multiple lists
Enum.zip_with([1, 2, 3], [4, 5, 6], fn a, b -> a + b end)
# [5, 7, 9]

# Nested map
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
Enum.map(matrix, fn row -> Enum.map(row, &(&1 * 2)) end)
# [[2, 4, 6], [8, 10, 12], [14, 16, 18]]
```

### 62.2 filter ขั้นสูง

```elixir
# filter กับ Pattern Matching
responses = [
  {:ok, 200, "OK"},
  {:error, 404, "Not Found"},
  {:ok, 201, "Created"},
  {:error, 500, "Server Error"}
]

# เฉพาะ success responses
Enum.filter(responses, fn {status, _, _} -> status == :ok end)
# [{:ok, 200, "OK"}, {:ok, 201, "Created"}]

# filter map กับ complex conditions
users = [
  %{name: "Alice", age: 30, role: :admin, active: true},
  %{name: "Bob", age: 17, role: :user, active: true},
  %{name: "Charlie", age: 25, role: :user, active: false}
]

active_adults = Enum.filter(users, fn user ->
  user.active and user.age >= 18
end)
# [Alice, Charlie ที่ไม่ active ก็ออกไป... wait, Charlie inactive]
# ได้แค่ Alice เพราะ Bob อายุ < 18 และ Charlie inactive
```

### 62.3 reduce กับ Multiple Accumulators

```elixir
# แยก list เป็นคู่/คี่พร้อมกัน
{evens, odds} = Enum.reduce([1, 2, 3, 4, 5, 6], {[], []}, fn x, {evens, odds} ->
  if rem(x, 2) == 0 do
    {[x | evens], odds}
  else
    {evens, [x | odds]}
  end
end)

# evens = [6, 4, 2] (reversed)
# odds  = [5, 3, 1] (reversed)

# หรือใช้ Enum.split_with
{evens, odds} = Enum.split_with([1, 2, 3, 4, 5, 6], fn x -> rem(x, 2) == 0 end)
# evens = [2, 4, 6], odds = [1, 3, 5]
```

---

## Step 63: Enum Functions คลังอาวุธ

### 63.1 Slicing และ Taking

```elixir
list = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# take - เอา n ตัวแรก
Enum.take(list, 3)      # [1, 2, 3]
Enum.take(list, -3)     # [8, 9, 10] (จากท้าย)

# drop - ทิ้ง n ตัวแรก
Enum.drop(list, 3)      # [4, 5, 6, 7, 8, 9, 10]
Enum.drop(list, -3)     # [1, 2, 3, 4, 5, 6, 7]

# split - แบ่งเป็น 2 ส่วน
Enum.split(list, 3)     # {[1, 2, 3], [4, 5, 6, 7, 8, 9, 10]}
Enum.split(list, -3)    # {[1, 2, 3, 4, 5, 6, 7], [8, 9, 10]}

# take_while - เอาตราบเท่าที่ condition เป็น true
Enum.take_while(list, fn x -> x < 5 end)  # [1, 2, 3, 4]

# drop_while
Enum.drop_while(list, fn x -> x < 5 end)  # [5, 6, 7, 8, 9, 10]

# split_while
Enum.split_while(list, fn x -> x < 5 end)  # {[1, 2, 3, 4], [5, 6, 7, 8, 9, 10]}

# slice
Enum.slice(list, 2, 4)   # [3, 4, 5, 6] (start_index, count)
Enum.slice(list, 2..5)   # [3, 4, 5, 6] (range)
```

### 63.2 Aggregation

```elixir
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

Enum.sum(numbers)      # 39
Enum.min(numbers)      # 1
Enum.max(numbers)      # 9
Enum.min_max(numbers)  # {1, 9}
Enum.count(numbers)    # 10
Enum.product(numbers)  # 3*1*4*1*5*9*2*6*5*3 = 97200

# ด้วย function
Enum.sum_by(users, & &1.age)  # Elixir 1.15+
Enum.min_by(users, & &1.age)  # return element ที่มี min value
```

### 63.3 Deduplication

```elixir
# uniq
Enum.uniq([1, 2, 3, 2, 1, 4])  # [1, 2, 3, 4]

# uniq_by
users = [
  %{id: 1, name: "Alice"},
  %{id: 2, name: "Bob"},
  %{id: 1, name: "Alice (duplicate)"}
]
Enum.uniq_by(users, & &1.id)
# [%{id: 1, name: "Alice"}, %{id: 2, name: "Bob"}]

# dedup - ลบที่อยู่ติดกัน
Enum.dedup([1, 1, 2, 3, 3, 3, 4])  # [1, 2, 3, 4]
Enum.dedup_by([{1, "a"}, {1, "b"}, {2, "c"}], fn {n, _} -> n end)
# [{1, "a"}, {2, "c"}]
```

---

## Step 64: Enum Combinations

### 64.1 Combinations และ Permutations

```elixir
# ไม่มี built-in combination แต่สามารถสร้างได้
defmodule Combination do
  def combinations(_, 0), do: [[]]
  def combinations([], _), do: []
  def combinations([head | tail], n) do
    for combo <- combinations(tail, n - 1) do
      [head | combo]
    end ++ combinations(tail, n)
  end
  
  def permutations([]), do: [[]]
  def permutations(list) do
    for element <- list,
        rest <- permutations(list -- [element]) do
      [element | rest]
    end
  end
end

Combination.combinations([1, 2, 3, 4], 2)
# [[1, 2], [1, 3], [1, 4], [2, 3], [2, 4], [3, 4]]

Combination.permutations([1, 2, 3])
# [[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]
```

### 64.2 Interleave และ Intersperse

```elixir
# intersperse - แทรก separator
Enum.intersperse([1, 2, 3, 4], 0)
# [1, 0, 2, 0, 3, 0, 4]

# เหมือนกับ join สำหรับ String
Enum.join(["a", "b", "c"], ", ")  # "a, b, c"
Enum.intersperse(["a", "b", "c"], ", ")
# ["a", ", ", "b", ", ", "c"]
```

---

## Step 65: Stream ขั้นสูง

### 65.1 Stream Transformations

```elixir
# scan - เหมือน reduce แต่ emit ทุก intermediate value
Stream.scan(1..5, 0, fn x, acc -> x + acc end)
|> Enum.to_list()
# [1, 3, 6, 10, 15] (running sum)

# transform - แปลงพร้อม emit หลาย values
Stream.transform(1..5, 0, fn x, acc ->
  new_acc = acc + x
  {[x, new_acc], new_acc}  # emit [original, running_sum], new_state
end)
|> Enum.to_list()
# [1, 1, 2, 3, 3, 6, 4, 10, 5, 15]
```

### 65.2 Stream Concurrent Processing

```elixir
# ประมวลผลแบบ parallel ด้วย Task
list = 1..100

list
|> Task.async_stream(fn x ->
     # จำลอง heavy computation
     Process.sleep(10)
     x * x
   end,
   max_concurrency: 10,  # รัน 10 tasks พร้อมกัน
   timeout: 5000
)
|> Stream.map(fn {:ok, result} -> result end)
|> Enum.to_list()
```

### 65.3 Stream.cycle

```elixir
# cycle - วนซ้ำ list อนันต์
Stream.cycle([:red, :green, :blue])
|> Enum.take(7)
# [:red, :green, :blue, :red, :green, :blue, :red]

# ใช้สำหรับ round-robin
servers = ["server1", "server2", "server3"]
requests = 1..10

Stream.cycle(servers)
|> Enum.zip(requests)
|> Enum.map(fn {server, request} ->
     "Request #{request} → #{server}"
   end)
```

### 65.4 Stream.repeatedly

```elixir
# repeatedly - เรียก function ซ้ำๆ
Stream.repeatedly(fn -> :rand.uniform(100) end)
|> Enum.take(5)
# [42, 17, 83, 31, 56] (random numbers)

# สร้าง random passwords
Stream.repeatedly(fn -> 
  :rand.uniform(26) + 64  # ASCII uppercase
  |> List.wrap()
  |> IO.chardata_to_string()
end)
|> Enum.take(10)
|> Enum.join()
# "XKQMBZPJLA"
```

---

## Step 66: Collectable Protocol

Collectable คือ inverse ของ Enumerable - รับ elements เข้ามาสร้าง collection

### 66.1 into: กับ for comprehension

```elixir
# สร้าง Map
for {k, v} <- [a: 1, b: 2, c: 3], into: %{}, do: {k, v * 10}
# %{a: 10, b: 20, c: 30}

# สร้าง MapSet
for x <- 1..10, into: MapSet.new(), do: rem(x, 3)
# MapSet.new([0, 1, 2])

# สร้าง String
for char <- ?a..?z, into: "", do: <<char>>
# "abcdefghijklmnopqrstuvwxyz"
```

### 66.2 Enum.into

```elixir
# Enum.into เป็น explicit version
Enum.into([a: 1, b: 2], %{})
# %{a: 1, b: 2}

Enum.into([1, 2, 3], MapSet.new())
# MapSet.new([1, 2, 3])

Enum.into(["hello", " ", "world"], "")
# "hello world"
```

### 66.3 Custom Collectable

```elixir
defmodule MyQueue do
  defstruct items: []
  
  def new, do: %MyQueue{}
  def push(%MyQueue{items: items}, item), do: %MyQueue{items: items ++ [item]}
end

defimpl Collectable, for: MyQueue do
  def into(queue) do
    collector = fn
      queue, {:cont, item} -> MyQueue.push(queue, item)
      queue, :done          -> queue
      _queue, :halt         -> :ok
    end
    
    {queue, collector}
  end
end

# ตอนนี้สามารถใช้ into: %MyQueue{} ได้
result = for x <- 1..5, into: MyQueue.new(), do: x * 2
# %MyQueue{items: [2, 4, 6, 8, 10]}
```

---

## Step 67: Enum กับ Maps ขั้นสูง

### 67.1 Map Operations

```elixir
users = %{
  1 => %{name: "Alice", age: 30, role: :admin},
  2 => %{name: "Bob", age: 25, role: :user},
  3 => %{name: "Charlie", age: 35, role: :admin}
}

# map สำหรับ Map
Map.new(users, fn {id, user} -> {id, user.name} end)
# %{1 => "Alice", 2 => "Bob", 3 => "Charlie"}

# filter สำหรับ Map
Map.filter(users, fn {_, user} -> user.role == :admin end)
# %{1 => %{name: "Alice"...}, 3 => %{name: "Charlie"...}}

# Enum.filter บน Map return Keyword list
Enum.filter(users, fn {_, user} -> user.role == :admin end)
# [{1, %{name: "Alice"...}}, {3, %{name: "Charlie"...}}]

# แปลงกลับเป็น Map
Enum.into(
  Enum.filter(users, fn {_, user} -> user.role == :admin end),
  %{}
)
# %{1 => %{...}, 3 => %{...}}
```

### 67.2 Map.update กับ Complex Operations

```elixir
# เพิ่ม/อัปเดต values ทุก key พร้อมกัน
inventory = %{apple: 10, banana: 5, cherry: 20}

# เพิ่ม 5 ให้ทุก item
Map.new(inventory, fn {item, count} -> {item, count + 5} end)
# %{apple: 15, banana: 10, cherry: 25}

# Map.put_new - เพิ่มเฉพาะ key ที่ยังไม่มี
Map.put_new(inventory, :date, 0)
# %{apple: 10, banana: 5, cherry: 20, date: 0}
Map.put_new(inventory, :apple, 999)  # apple มีแล้ว ไม่เปลี่ยน
# %{apple: 10, banana: 5, cherry: 20}

# Map.update - อัปเดตด้วย function
Map.update(inventory, :apple, 0, fn count -> count + 10 end)
# %{apple: 20, banana: 5, cherry: 20}
Map.update(inventory, :date, 0, fn count -> count + 10 end)
# %{apple: 10, banana: 5, cherry: 20, date: 0} (date ไม่มี ใช้ default 0)

# Map.update! - raise ถ้า key ไม่มี
Map.update!(inventory, :apple, &(&1 + 10))
# %{apple: 20, banana: 5, cherry: 20}
```

---

## Step 68: Sorting ขั้นสูง

### 68.1 Custom Comparators

```elixir
# sort_by กับ multiple criteria
users = [
  %{name: "Alice", age: 30, score: 85},
  %{name: "Bob", age: 25, score: 90},
  %{name: "Charlie", age: 30, score: 75},
  %{name: "Dave", age: 25, score: 90}
]

# เรียงตาม age แล้วตาม score (descending)
Enum.sort_by(users, fn u -> {u.age, -u.score} end)
# Dave(25,90) → Bob(25,90) → Alice(30,85) → Charlie(30,75)
# Wait: -90 < -85, so score 90 comes first
# = [Dave, Bob, Charlie, Alice] ... ลองคิดดู

# ใช้ Enum.sort กับ comparator function
Enum.sort(users, fn a, b ->
  cond do
    a.age != b.age -> a.age < b.age
    a.score != b.score -> a.score > b.score  # descending score
    true -> a.name < b.name
  end
end)
```

### 68.2 Stable Sort

```elixir
# Elixir's sort เป็น stable sort (ลำดับ element ที่เท่ากันไม่เปลี่ยน)
data = [{:a, 1}, {:b, 1}, {:c, 2}, {:d, 1}]

Enum.sort_by(data, fn {_, n} -> n end)
# [{:a, 1}, {:b, 1}, {:d, 1}, {:c, 2}]
# element ที่มี n=1 ยังคงลำดับ a, b, d
```

---

## Step 69: Enum กับ Concurrency

### 69.1 Task.async_stream

```elixir
# ประมวลผล I/O intensive tasks แบบ parallel
urls = [
  "https://api.example.com/users",
  "https://api.example.com/products",
  "https://api.example.com/orders"
]

# แบบ sequential (ช้า)
Enum.map(urls, &fetch_url/1)

# แบบ parallel (เร็วกว่ามาก)
urls
|> Task.async_stream(&fetch_url/1,
     max_concurrency: 5,
     timeout: 10_000,
     on_timeout: :kill_task
   )
|> Enum.map(fn
     {:ok, result} -> result
     {:exit, reason} -> {:error, reason}
   end)
```

### 69.2 Parallel Map

```elixir
defmodule ParallelMap do
  def map(list, func, opts \\ []) do
    list
    |> Task.async_stream(func, opts)
    |> Enum.map(fn {:ok, result} -> result end)
  end
end

# ใช้งาน
result = ParallelMap.map(1..10, fn x ->
  Process.sleep(100)  # simulate slow computation
  x * x
end, max_concurrency: 4)
```

---

## Step 70: Real-world Enumerable Examples

### 70.1 Data Pipeline

```elixir
defmodule DataPipeline do
  def process(raw_data) do
    raw_data
    |> parse_records()
    |> validate_records()
    |> transform_records()
    |> group_and_aggregate()
    |> format_output()
  end
  
  defp parse_records(raw) do
    raw
    |> String.split("\n")
    |> Enum.filter(&(String.trim(&1) != ""))
    |> Enum.map(&parse_line/1)
  end
  
  defp parse_line(line) do
    [date, name, amount] = String.split(line, ",")
    %{
      date: Date.from_iso8601!(String.trim(date)),
      name: String.trim(name),
      amount: String.to_float(String.trim(amount))
    }
  end
  
  defp validate_records(records) do
    Enum.filter(records, fn record ->
      record.amount > 0 and record.name != ""
    end)
  end
  
  defp transform_records(records) do
    Enum.map(records, fn record ->
      %{record | amount: Float.round(record.amount, 2)}
    end)
  end
  
  defp group_and_aggregate(records) do
    records
    |> Enum.group_by(& &1.name)
    |> Enum.map(fn {name, transactions} ->
         %{
           name: name,
           total: Enum.sum(Enum.map(transactions, & &1.amount)),
           count: length(transactions),
           avg: Enum.sum(Enum.map(transactions, & &1.amount)) / length(transactions)
         }
       end)
    |> Enum.sort_by(& &1.total, :desc)
  end
  
  defp format_output(aggregated) do
    Enum.map(aggregated, fn row ->
      "#{row.name}: total=#{row.total}, count=#{row.count}, avg=#{Float.round(row.avg, 2)}"
    end)
  end
end
```

### 70.2 Event Stream Processing

```elixir
defmodule EventProcessor do
  def process_events(events) do
    events
    |> Stream.filter(&valid_event?/1)
    |> Stream.map(&enrich_event/1)
    |> Stream.chunk_by(& &1.user_id)
    |> Stream.map(&process_user_events/1)
    |> Enum.to_list()
  end
  
  defp valid_event?(%{type: type, user_id: id}) 
    when is_atom(type) and is_integer(id), do: true
  defp valid_event?(_), do: false
  
  defp enrich_event(event) do
    Map.put(event, :processed_at, DateTime.utc_now())
  end
  
  defp process_user_events(user_events) do
    user_id = hd(user_events).user_id
    
    %{
      user_id: user_id,
      event_count: length(user_events),
      event_types: user_events |> Enum.map(& &1.type) |> Enum.uniq(),
      first_event: hd(user_events).processed_at,
      last_event: List.last(user_events).processed_at
    }
  end
end
```

### 70.3 Analytics Engine

```elixir
defmodule Analytics do
  def compute_stats(numbers) when is_list(numbers) and length(numbers) > 0 do
    sorted = Enum.sort(numbers)
    count  = length(numbers)
    sum    = Enum.sum(numbers)
    mean   = sum / count
    
    variance = numbers
    |> Enum.map(fn x -> :math.pow(x - mean, 2) end)
    |> Enum.sum()
    |> Kernel./(count)
    
    %{
      count:    count,
      sum:      sum,
      mean:     mean,
      median:   median(sorted, count),
      mode:     mode(numbers),
      variance: variance,
      std_dev:  :math.sqrt(variance),
      min:      hd(sorted),
      max:      List.last(sorted),
      range:    List.last(sorted) - hd(sorted),
      p25:      percentile(sorted, 0.25),
      p75:      percentile(sorted, 0.75),
      p95:      percentile(sorted, 0.95),
      p99:      percentile(sorted, 0.99)
    }
  end
  
  defp median(sorted, count) when rem(count, 2) == 1 do
    Enum.at(sorted, div(count, 2))
  end
  defp median(sorted, count) do
    mid = div(count, 2)
    (Enum.at(sorted, mid - 1) + Enum.at(sorted, mid)) / 2
  end
  
  defp mode(numbers) do
    frequencies = Enum.frequencies(numbers)
    max_count = frequencies |> Map.values() |> Enum.max()
    frequencies
    |> Enum.filter(fn {_, count} -> count == max_count end)
    |> Enum.map(fn {value, _} -> value end)
    |> Enum.sort()
  end
  
  defp percentile(sorted, p) do
    index = p * (length(sorted) - 1)
    lower = floor(index)
    upper = ceil(index)
    
    if lower == upper do
      Enum.at(sorted, round(index))
    else
      lower_val = Enum.at(sorted, lower)
      upper_val = Enum.at(sorted, upper)
      lower_val + (upper_val - lower_val) * (index - lower)
    end
  end
end

data = Enum.map(1..1000, fn _ -> :rand.uniform(100) end)
stats = Analytics.compute_stats(data)
IO.puts("Mean: #{stats.mean}, Std Dev: #{Float.round(stats.std_dev, 2)}")
```

---

## สรุป Part 07: Enumerables และ Streams

✅ **Step 61** - Enumerable Protocol  
✅ **Step 62** - map/filter/reduce ขั้นสูง  
✅ **Step 63** - Enum คลังอาวุธ  
✅ **Step 64** - Combinations  
✅ **Step 65** - Stream ขั้นสูง  
✅ **Step 66** - Collectable Protocol  
✅ **Step 67** - Enum กับ Maps  
✅ **Step 68** - Sorting ขั้นสูง  
✅ **Step 69** - Enum กับ Concurrency  
✅ **Step 70** - Real-world Examples  

---

## แบบฝึกหัด Part 07

1. Implement `Enumerable` สำหรับ Binary Tree
2. สร้าง lazy number generator ด้วย Stream.unfold
3. เขียน function ที่อ่าน large CSV ด้วย Stream
4. สร้าง Analytics module ที่คำนวณ rolling average
5. Implement parallel `flat_map` ด้วย Task.async_stream

---

➡️ [Part 08: Maps, Keyword Lists, Structs](./part-08-maps-structs.md)
