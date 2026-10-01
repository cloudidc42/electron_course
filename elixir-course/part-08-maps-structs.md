# Part 08: Maps, Keyword Lists, Structs (Steps 71-80)

## Step 71: Maps ลึกขึ้น

### 71.1 Map Construction

```elixir
# สร้าง Map หลายวิธี
# 1. Literal
user = %{name: "Alice", age: 30}

# 2. Map.new จาก list ของ tuples
Map.new([{:name, "Alice"}, {:age, 30}])

# 3. Map.new กับ transform function
Map.new(1..5, fn x -> {x, x * x} end)
# %{1 => 1, 2 => 4, 3 => 9, 4 => 16, 5 => 25}

# 4. จาก Keyword List
Enum.into([name: "Alice", age: 30], %{})

# Map กับ complex keys
%{
  {1, 2} => "tuple key",
  [:a, :b] => "list key",  # ไม่แนะนำ
  %{} => "map key"          # ไม่แนะนำ
}
```

### 71.2 Map Access Patterns

```elixir
user = %{
  name: "Alice",
  address: %{
    street: "123 Main St",
    city: "Bangkok"
  },
  roles: [:admin, :user]
}

# Access nested
user.address.city         # "Bangkok"
user[:address][:city]     # "Bangkok"

# get_in - ลึกหลาย levels
get_in(user, [:address, :city])   # "Bangkok"
get_in(user, [:address, :country]) # nil

# Access list element ใน map
get_in(user, [:roles, Access.at(0)])   # :admin
get_in(user, [:roles, Access.at(-1)])  # :user

# put_in - แก้ไข nested
updated = put_in(user.address.city, "Chiang Mai")
# หรือ
updated = put_in(user, [:address, :city], "Chiang Mai")

# update_in - แก้ไขด้วย function
update_in(user.address.city, &String.upcase/1)

# get_and_update_in - ดึงและอัปเดตพร้อมกัน
{old_city, new_user} = get_and_update_in(
  user, 
  [:address, :city], 
  fn city -> {city, String.upcase(city)} end
)
```

### 71.3 Map Merging และ Updating

```elixir
defaults = %{
  role: :user,
  active: true,
  timezone: "UTC"
}

user_input = %{
  name: "Alice",
  email: "alice@example.com",
  timezone: "Asia/Bangkok"
}

# merge - ค่าใน second map ชนะ
result = Map.merge(defaults, user_input)
# %{active: true, email: "alice@example.com", 
#   name: "Alice", role: :user, timezone: "Asia/Bangkok"}

# merge กับ resolver function
Map.merge(%{a: 1, b: 2}, %{b: 3, c: 4}, fn key, v1, v2 ->
  IO.puts("Conflict on #{key}: #{v1} vs #{v2}")
  v1 + v2  # combine values
end)
# Conflict on b: 2 vs 3
# %{a: 1, b: 5, c: 4}
```

---

## Step 72: Advanced Map Patterns

### 72.1 Map.reduce

```elixir
# ประมวลผล Map
scores = %{alice: 85, bob: 92, charlie: 78, dave: 88}

# หา average
total = Map.values(scores) |> Enum.sum()
count = map_size(scores)
average = total / count  # 85.75

# Transform ทุก values
Map.new(scores, fn {name, score} ->
  grade = cond do
    score >= 90 -> "A"
    score >= 80 -> "B"
    score >= 70 -> "C"
    true        -> "F"
  end
  {name, %{score: score, grade: grade}}
end)
```

### 72.2 Dynamic Map Operations

```elixir
defmodule DynamicConfig do
  def build(options) do
    base = %{
      debug: false,
      log_level: :info,
      timeout: 5000
    }
    
    Enum.reduce(options, base, fn {key, value}, config ->
      if Map.has_key?(config, key) do
        Map.put(config, key, value)
      else
        config  # ignore unknown keys
      end
    end)
  end
  
  def get(config, key, default \\ nil) do
    Map.get(config, key, default)
  end
end

config = DynamicConfig.build([
  debug: true,
  timeout: 10000,
  unknown_key: "ignored"
])
# %{debug: true, log_level: :info, timeout: 10000}
```

### 72.3 Map as Accumulator

```elixir
# Build Index
defmodule Indexer do
  def build_word_index(documents) do
    Enum.reduce(documents, %{}, fn {doc_id, text}, index ->
      words = text |> String.downcase() |> String.split()
      
      Enum.reduce(words, index, fn word, acc ->
        Map.update(acc, word, [doc_id], fn doc_ids ->
          [doc_id | doc_ids]
        end)
      end)
    end)
  end
  
  def search(index, word) do
    Map.get(index, String.downcase(word), [])
  end
end

documents = [
  {1, "Elixir is a functional language"},
  {2, "Elixir runs on the BEAM VM"},
  {3, "Functional programming is powerful"}
]

index = Indexer.build_word_index(documents)
Indexer.search(index, "Elixir")       # [2, 1]
Indexer.search(index, "functional")   # [3, 1]
```

---

## Step 73: Struct พื้นฐาน

Struct คือ Map ที่มี schema กำหนดไว้ล่วงหน้า

### 73.1 defstruct

```elixir
defmodule User do
  defstruct [:name, :email, age: 0, role: :user, active: true]
  # :name, :email ไม่มี default (nil)
  # age: 0, role: :user, active: true มี default
end

# สร้าง Struct
alice = %User{name: "Alice", email: "alice@example.com", age: 30}
# %User{active: true, age: 30, email: "alice@example.com", name: "Alice", role: :user}

# ไม่ต้องใส่ทุก field
bob = %User{name: "Bob"}
# %User{active: true, age: 0, email: nil, name: "Bob", role: :user}

# Access ด้วย dot notation
alice.name   # "Alice"
alice.age    # 30
alice.role   # :user
```

### 73.2 Update Struct

```elixir
# Update ด้วย | syntax
updated = %User{alice | age: 31, role: :admin}
# %User{active: true, age: 31, email: "alice@example.com", name: "Alice", role: :admin}

# ต้นฉบับไม่เปลี่ยน (immutable)
alice.age    # ยังเป็น 30

# Map.put ก็ใช้ได้
Map.put(alice, :age, 31)

# แต่ไม่สามารถเพิ่ม key ที่ไม่ได้ define ใน struct
Map.put(alice, :phone, "0812345678")
# %{...} จะกลายเป็น Map ธรรมดา ไม่ใช่ Struct อีกต่อไป!
```

### 73.3 Struct กับ Pattern Matching

```elixir
def greet(%User{name: name, role: :admin}) do
  "Welcome, Admin #{name}!"
end

def greet(%User{name: name}) do
  "Hello, #{name}!"
end

def greet(%User{} = user) do
  # ดึง struct ทั้งหมด
  "User: #{inspect(user)}"
end

greet(alice)  # "Welcome, Admin Alice!"
greet(bob)    # "Hello, Bob!"
```

---

## Step 74: Struct กับ Module Functions

### 74.1 Struct + Functions = Object-like Pattern

```elixir
defmodule BankAccount do
  defstruct [
    :id,
    :owner,
    balance: 0,
    currency: :thb,
    transactions: []
  ]
  
  # Constructor
  def new(id, owner, initial_balance \\ 0) do
    %BankAccount{
      id: id,
      owner: owner,
      balance: initial_balance
    }
  end
  
  # Operations
  def deposit(account, amount) when amount > 0 do
    transaction = {DateTime.utc_now(), :deposit, amount}
    
    {:ok, %BankAccount{account |
      balance: account.balance + amount,
      transactions: [transaction | account.transactions]
    }}
  end
  
  def deposit(_, amount), do: {:error, "Amount must be positive, got: #{amount}"}
  
  def withdraw(%BankAccount{balance: balance}, amount) when amount > balance do
    {:error, :insufficient_funds}
  end
  
  def withdraw(account, amount) when amount > 0 do
    transaction = {DateTime.utc_now(), :withdrawal, amount}
    
    {:ok, %BankAccount{account |
      balance: account.balance - amount,
      transactions: [transaction | account.transactions]
    }}
  end
  
  def transfer(from_account, to_account, amount) do
    with {:ok, from} <- withdraw(from_account, amount),
         {:ok, to}   <- deposit(to_account, amount) do
      {:ok, from, to}
    end
  end
  
  def statement(%BankAccount{owner: owner, balance: balance, transactions: txns}) do
    header = "Account Statement for: #{owner}"
    divider = String.duplicate("-", String.length(header))
    
    entries = txns
    |> Enum.reverse()
    |> Enum.map(fn {date, type, amount} ->
         "#{DateTime.to_string(date)} | #{type} | #{amount}"
       end)
    
    [header, divider | entries] ++ [divider, "Balance: #{balance}"]
    |> Enum.join("\n")
  end
end

# การใช้งาน
account = BankAccount.new("ACC001", "Alice", 1000)

{:ok, account} = BankAccount.deposit(account, 500)
{:ok, account} = BankAccount.withdraw(account, 200)

IO.puts(BankAccount.statement(account))
```

### 74.2 Struct Validation

```elixir
defmodule Product do
  defstruct [:name, :price, :sku, category: :general, active: true]
  
  @valid_categories [:electronics, :clothing, :food, :general]
  
  def new(attrs) do
    with {:ok, name}     <- validate_name(attrs[:name]),
         {:ok, price}    <- validate_price(attrs[:price]),
         {:ok, sku}      <- validate_sku(attrs[:sku]),
         {:ok, category} <- validate_category(attrs[:category] || :general) do
      
      {:ok, struct!(__MODULE__, [
        name: name,
        price: price,
        sku: sku,
        category: category
      ])}
    end
  end
  
  defp validate_name(nil), do: {:error, "Name is required"}
  defp validate_name(n) when is_binary(n) and byte_size(n) > 0, do: {:ok, n}
  defp validate_name(_), do: {:error, "Name must be a non-empty string"}
  
  defp validate_price(nil), do: {:error, "Price is required"}
  defp validate_price(p) when is_number(p) and p >= 0, do: {:ok, p}
  defp validate_price(_), do: {:error, "Price must be a non-negative number"}
  
  defp validate_sku(nil), do: {:error, "SKU is required"}
  defp validate_sku(sku) when is_binary(sku) do
    if Regex.match?(~r/^[A-Z0-9-]+$/, sku) do
      {:ok, sku}
    else
      {:error, "SKU must contain only uppercase letters, numbers, and hyphens"}
    end
  end
  
  defp validate_category(cat) when cat in @valid_categories, do: {:ok, cat}
  defp validate_category(cat), do: {:error, "Invalid category: #{cat}"}
end

Product.new(%{name: "MacBook Pro", price: 45000, sku: "APPLE-MBP-2024"})
# {:ok, %Product{name: "MacBook Pro", price: 45000, ...}}

Product.new(%{name: "Laptop", price: -100, sku: "LAP-001"})
# {:error, "Price must be a non-negative number"}
```

---

## Step 75: Access Module

Access ใช้สำหรับ dynamic key access และ update

### 75.1 Access พื้นฐาน

```elixir
# Access.get - เหมือน Map.get แต่ polymorphic
Access.get(%{a: 1}, :a)       # 1
Access.get([1, 2, 3], 0)      # 1
Access.get([a: 1, b: 2], :a)  # 1

# get_in ใช้ Access functions
data = %{users: [%{name: "Alice"}, %{name: "Bob"}]}

get_in(data, [:users, Access.at(0), :name])  # "Alice"
get_in(data, [:users, Access.all(), :name])   # ["Alice", "Bob"]
```

### 75.2 Access.key, Access.at, Access.all

```elixir
data = %{
  teams: [
    %{name: "Backend", members: ["Alice", "Bob"]},
    %{name: "Frontend", members: ["Charlie", "Dave"]}
  ]
}

# ดึง member แรกของ team แรก
get_in(data, [:teams, Access.at(0), :members, Access.at(0)])
# "Alice"

# ดึงชื่อ team ทั้งหมด
get_in(data, [:teams, Access.all(), :name])
# ["Backend", "Frontend"]

# อัปเดต members ของ team แรก
update_in(data, [:teams, Access.at(0), :members], fn members ->
  members ++ ["Eve"]
end)
```

### 75.3 Custom Access Implementations

```elixir
defmodule ConfigTree do
  defstruct [:children, :value]
  
  def get_by_path(%__MODULE__{} = tree, []) do
    tree.value
  end
  
  def get_by_path(%__MODULE__{children: children}, [key | rest]) do
    case Map.get(children, key) do
      nil   -> nil
      child -> get_by_path(child, rest)
    end
  end
end
```

---

## Step 76: Keyword Lists ขั้นสูง

### 76.1 Keyword List Functions

```elixir
kw = [a: 1, b: 2, c: 3, a: 4]  # key ซ้ำได้!

# get - ดึงค่าแรกของ key
Keyword.get(kw, :a)    # 1 (ค่าแรก)
Keyword.get(kw, :d)    # nil
Keyword.get(kw, :d, "default")  # "default"

# get_values - ดึงทุกค่าของ key
Keyword.get_values(kw, :a)  # [1, 4]

# put - เพิ่ม key-value (เพิ่มด้านหน้า, อาจมี key ซ้ำ)
Keyword.put(kw, :e, 5)     # [e: 5, a: 1, b: 2, c: 3, a: 4]

# put_new - เพิ่มเฉพาะถ้า key ยังไม่มี
Keyword.put_new(kw, :a, 99)  # ไม่เปลี่ยนเพราะ :a มีแล้ว
Keyword.put_new(kw, :e, 5)   # เพิ่ม :e = 5

# merge
Keyword.merge([a: 1, b: 2], [b: 3, c: 4])
# [a: 1, b: 3, c: 4]  (key ซ้ำ ตัวหลังชนะ)

# delete
Keyword.delete(kw, :a)    # [b: 2, c: 3] (ลบทุก :a)
Keyword.delete_first(kw, :a)  # [b: 2, c: 3, a: 4] (ลบแค่ :a แรก)
```

### 76.2 Function Options Pattern

```elixir
defmodule HTTPClient do
  def get(url, opts \\ []) do
    timeout     = Keyword.get(opts, :timeout, 5000)
    retries     = Keyword.get(opts, :retries, 3)
    headers     = Keyword.get(opts, :headers, [])
    verify_ssl  = Keyword.get(opts, :verify_ssl, true)
    
    # ใช้ opts ในการ config request
    IO.puts("GET #{url}")
    IO.puts("Timeout: #{timeout}ms, Retries: #{retries}")
    IO.puts("SSL verification: #{verify_ssl}")
  end
end

# เรียกใช้โดยส่ง options
HTTPClient.get("https://api.example.com",
  timeout: 10_000,
  retries: 5,
  headers: [{"Authorization", "Bearer token123"}]
)
```

---

## Step 77: MapSet

MapSet เป็น Set data structure

### 77.1 MapSet พื้นฐาน

```elixir
# สร้าง MapSet
set = MapSet.new([1, 2, 3, 2, 1])
# MapSet.new([1, 2, 3]) - ไม่มีซ้ำ

# เพิ่ม element
MapSet.put(set, 4)    # MapSet.new([1, 2, 3, 4])
MapSet.put(set, 2)    # MapSet.new([1, 2, 3]) ไม่เปลี่ยน

# ลบ element
MapSet.delete(set, 2)  # MapSet.new([1, 3])

# ตรวจสอบ
MapSet.member?(set, 2)     # true
MapSet.member?(set, 5)     # false
2 in set                   # true (shorthand)

# ขนาด
MapSet.size(set)  # 3
```

### 77.2 Set Operations

```elixir
a = MapSet.new([1, 2, 3, 4])
b = MapSet.new([3, 4, 5, 6])

# Union (รวม)
MapSet.union(a, b)
# MapSet.new([1, 2, 3, 4, 5, 6])

# Intersection (ตัดกัน)
MapSet.intersection(a, b)
# MapSet.new([3, 4])

# Difference (ความแตกต่าง)
MapSet.difference(a, b)
# MapSet.new([1, 2]) - มีใน a แต่ไม่มีใน b

MapSet.difference(b, a)
# MapSet.new([5, 6]) - มีใน b แต่ไม่มีใน a

# Subset?
MapSet.subset?(MapSet.new([1, 2]), a)  # true
MapSet.subset?(a, b)                   # false
```

### 77.3 MapSet Use Cases

```elixir
defmodule TagSystem do
  # ระบบ Tags ด้วย MapSet
  def add_tags(item, new_tags) do
    existing = MapSet.new(item.tags)
    updated  = MapSet.union(existing, MapSet.new(new_tags))
    %{item | tags: MapSet.to_list(updated)}
  end
  
  def remove_tags(item, tags_to_remove) do
    existing = MapSet.new(item.tags)
    updated  = MapSet.difference(existing, MapSet.new(tags_to_remove))
    %{item | tags: MapSet.to_list(updated)}
  end
  
  def has_all_tags?(item, required_tags) do
    existing = MapSet.new(item.tags)
    required = MapSet.new(required_tags)
    MapSet.subset?(required, existing)
  end
  
  def find_by_tags(items, search_tags) do
    search = MapSet.new(search_tags)
    Enum.filter(items, fn item ->
      item_tags = MapSet.new(item.tags)
      not MapSet.disjoint?(item_tags, search)  # มี tag ร่วมกันอย่างน้อย 1 อัน
    end)
  end
end
```

---

## Step 78: Nested Data Structures

### 78.1 Update Nested Structures

```elixir
# deep update ด้วย put_in/update_in
config = %{
  database: %{
    host: "localhost",
    port: 5432,
    pool: %{
      size: 10,
      timeout: 5000
    }
  },
  cache: %{
    host: "localhost",
    port: 6379
  }
}

# เปลี่ยน pool size
updated = put_in(config, [:database, :pool, :size], 20)

# เพิ่ม timeout
updated = update_in(config, [:database, :pool, :timeout], fn t -> t * 2 end)

# เพิ่ม key ใหม่ใน nested map
updated = put_in(config, [:database, :name], "myapp_db")
```

### 78.2 Deep Merge

```elixir
defmodule DeepMerge do
  def merge(left, right) when is_map(left) and is_map(right) do
    Map.merge(left, right, fn _key, left_val, right_val ->
      merge(left_val, right_val)
    end)
  end
  
  def merge(_left, right), do: right
end

base_config = %{
  server: %{host: "localhost", port: 8080},
  database: %{host: "db.local", port: 5432}
}

override_config = %{
  server: %{port: 9090},
  logging: %{level: :debug}
}

DeepMerge.merge(base_config, override_config)
# %{
#   server: %{host: "localhost", port: 9090},  # host preserved, port updated
#   database: %{host: "db.local", port: 5432},
#   logging: %{level: :debug}                  # new key added
# }
```

---

## Step 79: Struct Inheritance Pattern

Elixir ไม่มี inheritance แต่ใช้ composition pattern แทน

### 79.1 Composition ด้วย Struct

```elixir
defmodule Address do
  defstruct [:street, :city, :country, zipcode: ""]
  
  def validate(%Address{city: nil}), do: {:error, "City required"}
  def validate(%Address{country: nil}), do: {:error, "Country required"}
  def validate(%Address{} = addr), do: {:ok, addr}
end

defmodule Person do
  defstruct [:name, :email, :address]
  
  def new(name, email, address_attrs) do
    with {:ok, address} <- build_address(address_attrs) do
      {:ok, %Person{name: name, email: email, address: address}}
    end
  end
  
  defp build_address(attrs) do
    addr = struct(Address, attrs)
    Address.validate(addr)
  end
end

defmodule Employee do
  defstruct [:person, :department, :salary, :hire_date]
  
  def new(person_attrs, employee_attrs) do
    with {:ok, person} <- Person.new(
           person_attrs[:name],
           person_attrs[:email],
           person_attrs[:address] || %{}
         ) do
      employee = struct(__MODULE__, Map.put(employee_attrs, :person, person))
      {:ok, employee}
    end
  end
  
  def name(%Employee{person: %Person{name: name}}), do: name
  def email(%Employee{person: %Person{email: email}}), do: email
end
```

### 79.2 Protocol กับ Struct

```elixir
defprotocol Serializable do
  def to_json(data)
  def to_string(data)
end

defmodule Point do
  defstruct [:x, :y]
end

defimpl Serializable, for: Point do
  def to_json(%Point{x: x, y: y}) do
    ~s({"x": #{x}, "y": #{y}})
  end
  
  def to_string(%Point{x: x, y: y}) do
    "(#{x}, #{y})"
  end
end

defmodule Circle do
  defstruct [:center, :radius]
end

defimpl Serializable, for: Circle do
  def to_json(%Circle{center: center, radius: r}) do
    ~s({"center": #{Serializable.to_json(center)}, "radius": #{r}})
  end
  
  def to_string(%Circle{center: center, radius: r}) do
    "Circle at #{Serializable.to_string(center)} with radius #{r}"
  end
end

point  = %Point{x: 3, y: 4}
circle = %Circle{center: point, radius: 5}

Serializable.to_json(circle)
# {"center": {"x": 3, "y": 4}, "radius": 5}
```

---

## Step 80: Real-world Data Modeling

### 80.1 E-commerce Data Model

```elixir
defmodule Ecommerce.Product do
  defstruct [
    :id, :name, :description, :sku,
    price: 0,
    stock: 0,
    categories: [],
    images: [],
    active: true,
    metadata: %{}
  ]
  
  def available?(%__MODULE__{active: true, stock: stock}) when stock > 0, do: true
  def available?(_), do: false
  
  def discounted_price(%__MODULE__{price: price}, discount_percent) 
    when discount_percent > 0 and discount_percent < 100 do
    price * (1 - discount_percent / 100)
  end
end

defmodule Ecommerce.CartItem do
  defstruct [:product_id, :quantity, :unit_price]
  
  def total(%__MODULE__{quantity: qty, unit_price: price}) do
    qty * price
  end
end

defmodule Ecommerce.Cart do
  defstruct [
    :user_id,
    items: [],
    coupon: nil,
    created_at: nil
  ]
  
  def new(user_id) do
    %__MODULE__{
      user_id: user_id,
      created_at: DateTime.utc_now()
    }
  end
  
  def add_item(cart, product, quantity) do
    item = %CartItem{
      product_id: product.id,
      quantity: quantity,
      unit_price: product.price
    }
    
    %{cart | items: [item | cart.items]}
  end
  
  def subtotal(%__MODULE__{items: items}) do
    items
    |> Enum.map(&CartItem.total/1)
    |> Enum.sum()
  end
  
  def total(%__MODULE__{} = cart) do
    subtotal = subtotal(cart)
    discount = case cart.coupon do
      nil -> 0
      %{type: :percentage, value: pct} -> subtotal * pct / 100
      %{type: :fixed, value: amount}   -> min(amount, subtotal)
    end
    
    max(0, subtotal - discount)
  end
  
  def item_count(%__MODULE__{items: items}) do
    Enum.sum(Enum.map(items, & &1.quantity))
  end
end

# การใช้งาน
alias Ecommerce.{Product, Cart}

product1 = %Product{id: 1, name: "MacBook Pro", price: 45000, stock: 5}
product2 = %Product{id: 2, name: "iPhone 15", price: 28000, stock: 10}

cart = Cart.new("user_001")
|> Cart.add_item(product1, 1)
|> Cart.add_item(product2, 2)

IO.puts("Subtotal: #{Cart.subtotal(cart)}")     # 101000
IO.puts("Items: #{Cart.item_count(cart)}")       # 3

cart_with_coupon = %{cart | coupon: %{type: :percentage, value: 10}}
IO.puts("Total with 10% off: #{Cart.total(cart_with_coupon)}")  # 90900
```

---

## สรุป Part 08

✅ **Step 71** - Maps ลึกขึ้น (construction, access, merge)  
✅ **Step 72** - Advanced Map patterns  
✅ **Step 73** - Struct พื้นฐาน  
✅ **Step 74** - Struct กับ Module functions  
✅ **Step 75** - Access Module  
✅ **Step 76** - Keyword Lists ขั้นสูง  
✅ **Step 77** - MapSet  
✅ **Step 78** - Nested Data Structures  
✅ **Step 79** - Struct Composition  
✅ **Step 80** - Real-world Data Modeling  

---

## แบบฝึกหัด Part 08

1. สร้าง `Config` struct ที่มี nested config
2. Implement `merge/2` ที่ deep merge Maps และ Structs
3. สร้าง simple ORM ที่แปลง Struct เป็น SQL
4. สร้าง Permission system ด้วย MapSet
5. Model ระบบ Library (Book, Member, Loan) ด้วย Structs

---

➡️ [Part 09: Strings และ Binaries](./part-09-strings-binaries.md)
