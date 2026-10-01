# Part 01: แนะนำ Elixir และการติดตั้ง (Steps 1-10)

## Step 1: Elixir คืออะไร?

Elixir คือภาษาโปรแกรมมิ่งแบบ **Functional** ที่ทำงานบน **Erlang VM (BEAM)** สร้างโดย **José Valim** ในปี 2012

```
"Elixir เป็นภาษาที่นำความสวยงามของ Ruby 
มารวมกับความทรงพลังของ Erlang"
                        — José Valim
```

### ทำไม Erlang VM (BEAM) ถึงสำคัญ?

BEAM ถูกออกแบบโดย Ericsson สำหรับระบบโทรคมนาคม ซึ่งต้องการ:
- **ทำงาน 99.9999999% (nine nines)** = downtime เพียง 31 มิลลิวินาทีต่อปี
- รองรับการเชื่อมต่อพร้อมกันนับล้าน
- สามารถอัปเดตโค้ดได้โดยไม่ต้องหยุดระบบ

```
ระบบโทรคมนาคม Ericsson ที่ใช้ Erlang → Uptime 99.9999999%
Discord ที่ใช้ Elixir               → รองรับ 5 ล้าน concurrent users
WhatsApp ที่ใช้ Erlang               → 50 พันล้านข้อความต่อวัน
```

---

## Step 2: เปรียบเทียบ Elixir กับภาษาอื่น

### Elixir vs Python/Ruby (Web Development)

| คุณสมบัติ | Elixir/Phoenix | Python/Django | Ruby/Rails |
|-----------|---------------|---------------|------------|
| Concurrent Users | ล้าน+ | หลักพัน-หมื่น | หลักพัน |
| Latency | < 1ms | 10-100ms | 10-100ms |
| Memory per Connection | ~2KB | ~1MB | ~1MB |
| Fault Tolerance | Built-in | Manual | Manual |

### ตัวอย่างเปรียบเทียบโค้ด: Hello World

**Python:**
```python
def hello(name):
    return f"Hello, {name}!"

print(hello("World"))
```

**Elixir:**
```elixir
defmodule Greeter do
  def hello(name) do
    "Hello, #{name}!"
  end
end

IO.puts Greeter.hello("World")
```

### Elixir vs JavaScript/Node.js

| คุณสมบัติ | Elixir | Node.js |
|-----------|--------|---------|
| Concurrency Model | Actor Model (Processes) | Event Loop (Single Thread) |
| Error Isolation | แต่ละ Process แยกกัน | Error หนึ่งอาจล้มทั้งระบบ |
| Hot Code Reload | Built-in | ต้องใช้ Library เพิ่ม |
| Distribution | Built-in หลาย nodes | ต้องตั้งค่าเพิ่ม |

---

## Step 3: แนวคิดหลักของ Elixir

### 3.1 Functional Programming

Elixir เป็นภาษา Functional ล้วนๆ หมายความว่า:

```elixir
# ข้อมูลไม่สามารถเปลี่ยนแปลงได้ (Immutable)
x = 5
x = 10  # ไม่ใช่การเปลี่ยนค่า x แต่เป็นการ bind ค่าใหม่

# Functions คือ First-class citizens
double = fn x -> x * 2 end
double.(5)  # => 10

# สามารถส่ง function เป็น argument ได้
Enum.map([1, 2, 3], fn x -> x * 2 end)  # => [2, 4, 6]
```

### 3.2 Actor Model - หัวใจของ Concurrency

```
[Process 1] ←→ [Mailbox] ←→ Messages
[Process 2] ←→ [Mailbox] ←→ Messages  
[Process 3] ←→ [Mailbox] ←→ Messages
     ↕              ↕
[Supervisor] ← Monitor → [Restart if failed]
```

แต่ละ Process:
- มี Memory เป็นของตัวเอง (ไม่แชร์ Memory กัน)
- สื่อสารผ่าน Messages เท่านั้น
- ถ้า Process หนึ่งพัง Process อื่นยังทำงานต่อได้

### 3.3 "Let it Crash" Philosophy

```elixir
# แทนที่จะป้องกันทุกกรณี...
def divide(a, b) do
  if b == 0 do
    {:error, "Cannot divide by zero"}
  else
    {:ok, a / b}
  end
end

# Elixir ส่งเสริมให้ทำแบบนี้
def divide(a, b) do
  a / b  # ถ้า b == 0 → Process crash → Supervisor restart
end
```

---

## Step 4: การติดตั้ง Elixir

### 4.1 ติดตั้งบน macOS

```bash
# วิธีที่ 1: ใช้ Homebrew (แนะนำ)
brew install elixir

# วิธีที่ 2: ใช้ asdf version manager (แนะนำสำหรับมืออาชีพ)
brew install asdf

# เพิ่ม plugins
asdf plugin add erlang
asdf plugin add elixir

# ติดตั้ง Erlang ก่อน (Elixir ต้องการ Erlang)
asdf install erlang 26.0
asdf install elixir 1.15.0

# ตั้งเป็น global version
asdf global erlang 26.0
asdf global elixir 1.15.0
```

### 4.2 ติดตั้งบน Ubuntu/Debian

```bash
# เพิ่ม repository ของ Erlang Solutions
wget https://packages.erlang-solutions.com/erlang-solutions_2.0_all.deb
sudo dpkg -i erlang-solutions_2.0_all.deb
sudo apt-get update

# ติดตั้ง Erlang และ Elixir
sudo apt-get install esl-erlang
sudo apt-get install elixir

# หรือใช้ asdf (แนะนำ)
git clone https://github.com/asdf-vm/asdf.git ~/.asdf
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
source ~/.bashrc

asdf plugin add erlang
asdf plugin add elixir
asdf install erlang 26.0
asdf install elixir 1.15.0
asdf global erlang 26.0
asdf global elixir 1.15.0
```

### 4.3 ติดตั้งบน Windows

```powershell
# วิธีที่ 1: ใช้ Chocolatey
choco install elixir

# วิธีที่ 2: ดาวน์โหลด Installer
# ไปที่ https://elixir-lang.org/install.html
# ดาวน์โหลด Windows installer
# รันไฟล์ .exe และทำตาม wizard

# วิธีที่ 3: ใช้ WSL2 (Windows Subsystem for Linux) - แนะนำ
# เปิด PowerShell as Administrator
wsl --install
# จากนั้นติดตั้งใน Ubuntu WSL เหมือน Linux
```

### 4.4 ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Elixir version
elixir --version

# Output ที่ควรเห็น:
# Erlang/OTP 26 [erts-14.0] [source] [64-bit] [smp:8:8] 
# [ds:8:8:10] [async-threads:1] [jit:ns]
# Elixir 1.15.0 (compiled with Erlang/OTP 26)

# ตรวจสอบ Mix (build tool)
mix --version
# Mix 1.15.0 (compiled with Erlang/OTP 26)

# ตรวจสอบ IEx (interactive shell)
iex --version
# IEx 1.15.0 (compiled with Erlang/OTP 26)
```

---

## Step 5: IEx - Interactive Elixir Shell

IEx เป็นเครื่องมือที่จะเป็นเพื่อนคู่กายตลอดการเรียนรู้

### 5.1 เริ่มใช้ IEx

```bash
# เปิด IEx
iex

# จะเห็น prompt:
# Erlang/OTP 26 [erts-14.0] ...
# Interactive Elixir (1.15.0) - press Ctrl+C to exit
# iex(1)>
```

### 5.2 คำสั่งพื้นฐานใน IEx

```elixir
# คำนวณเลข
iex(1)> 1 + 1
2

iex(2)> 10 / 3
3.3333333333333335

iex(3)> div(10, 3)    # หารเอาเลขจำนวนเต็ม
3

iex(4)> rem(10, 3)    # เศษจากการหาร (modulo)
1

# String
iex(5)> "Hello, World!"
"Hello, World!"

iex(6)> String.upcase("hello")
"HELLO"

# Boolean
iex(7)> true and false
false

iex(8)> true or false
true
```

### 5.3 คำสั่ง IEx ที่ต้องรู้

```elixir
# h/1 - ดู documentation
iex> h String.upcase
# แสดง documentation ของ String.upcase

# h/0 - ดู help
iex> h

# i/1 - ดูข้อมูลของค่า
iex> i "hello"
# Term: "hello"
# Data type: BitString
# Byte size: 5
# Description: This is a string: a UTF-8 encoded binary...

# v/0 - ดูค่าล่าสุด
iex> 1 + 1
2
iex> v
2

# v/1 - ดูค่าจาก line ที่ระบุ
iex> v(1)

# c/1 - compile ไฟล์
iex> c "my_file.ex"

# r/1 - recompile module
iex> r MyModule

# l/1 - load module
iex> l MyModule

# Ctrl+C แล้ว a - ออกจาก IEx
# หรือ
iex> System.halt()
```

### 5.4 ใช้ IEx กับ Project

```bash
# เปิด IEx พร้อมโหลด project
iex -S mix

# ใน IEx จะสามารถใช้ module ของ project ได้ทันที
```

---

## Step 6: โปรแกรมแรก - Hello, World!

### 6.1 รันใน IEx (วิธีที่เร็วที่สุด)

```bash
iex
```

```elixir
iex(1)> IO.puts("Hello, World!")
Hello, World!
:ok
```

### 6.2 สร้างไฟล์ .ex

สร้างไฟล์ `hello.ex`:

```elixir
# hello.ex
defmodule Hello do
  def greet(name \\ "World") do
    IO.puts("Hello, #{name}!")
  end
  
  def greet_multiple(names) do
    Enum.each(names, fn name ->
      greet(name)
    end)
  end
end

# เรียกใช้
Hello.greet()
Hello.greet("Elixir")
Hello.greet_multiple(["Alice", "Bob", "Charlie"])
```

```bash
# รันไฟล์
elixir hello.ex

# Output:
# Hello, World!
# Hello, Elixir!
# Hello, Alice!
# Hello, Bob!
# Hello, Charlie!
```

### 6.3 สร้างไฟล์ .exs (Script)

Elixir มีนามสกุลไฟล์ 2 แบบ:
- `.ex` - ไฟล์ที่ต้อง compile ก่อน (ใช้ใน project)
- `.exs` - Script ไฟล์ ไม่ต้อง compile (ใช้ทดสอบ)

```elixir
# hello.exs
IO.puts("Hello, World!")
IO.puts("นี่คือโปรแกรมแรกของผม")

name = "Elixir Developer"
IO.puts("สวัสดี, #{name}!")

# รันด้วย
# elixir hello.exs
```

---

## Step 7: สร้าง Mix Project แรก

Mix คือ Build Tool ของ Elixir ที่ใช้จัดการ project

### 7.1 สร้าง Project ใหม่

```bash
# สร้าง project ใหม่
mix new my_first_app

# Output:
# * creating README.md
# * creating .formatter.exs
# * creating .gitignore
# * creating mix.exs
# * creating lib/
# * creating lib/my_first_app.ex
# * creating test/
# * creating test/test_helper.exs
# * creating test/my_first_app_test.exs
# 
# Your Mix project was created successfully.
# You can use "mix" to compile it, test it, and more:
#
#     cd my_first_app
#     mix test
#
# Run "mix help" for more commands.
```

### 7.2 โครงสร้าง Project

```
my_first_app/
├── .formatter.exs     # กำหนดรูปแบบ code formatting
├── .gitignore         # ไฟล์ที่ไม่ต้อง commit
├── README.md          # เอกสาร project
├── mix.exs            # ไฟล์ configuration หลัก
├── lib/
│   └── my_first_app.ex  # โค้ดหลัก
└── test/
    ├── test_helper.exs    # ตั้งค่า test
    └── my_first_app_test.exs  # ไฟล์ test
```

### 7.3 ดูเนื้อหาใน mix.exs

```elixir
# mix.exs
defmodule MyFirstApp.MixProject do
  use Mix.Project

  def project do
    [
      app: :my_first_app,       # ชื่อ application
      version: "0.1.0",          # version
      elixir: "~> 1.15",         # Elixir version ที่ต้องการ
      start_permanent: Mix.env() == :prod,
      deps: deps()               # dependencies
    ]
  end

  # Application configuration
  def application do
    [
      extra_applications: [:logger]
    ]
  end

  # Dependencies (libraries)
  defp deps do
    [
      # เพิ่ม dependencies ที่นี่
      # {:dep_from_hexpm, "~> 0.3.0"},
    ]
  end
end
```

### 7.4 แก้ไขโค้ดหลัก

เปิดและแก้ไข `lib/my_first_app.ex`:

```elixir
# lib/my_first_app.ex
defmodule MyFirstApp do
  @moduledoc """
  Documentation for `MyFirstApp`.
  นี่คือ module แรกของเรา!
  """

  @doc """
  แสดงคำทักทาย
  
  ## Examples
  
      iex> MyFirstApp.hello()
      :world
      
      iex> MyFirstApp.greet("Alice")
      "Hello, Alice!"
  """
  def hello do
    :world
  end
  
  def greet(name) do
    "Hello, #{name}!"
  end
  
  def greet_with_time(name) do
    hour = :erlang.element(2, :erlang.time())
    
    greeting = cond do
      hour < 12 -> "Good morning"
      hour < 17 -> "Good afternoon"
      true      -> "Good evening"
    end
    
    "#{greeting}, #{name}!"
  end
end
```

### 7.5 รัน และ Test

```bash
cd my_first_app

# Compile project
mix compile

# รัน test
mix test

# เปิด IEx พร้อม project
iex -S mix
```

```elixir
# ใน IEx
iex(1)> MyFirstApp.hello()
:world

iex(2)> MyFirstApp.greet("World")
"Hello, World!"

iex(3)> MyFirstApp.greet_with_time("Alice")
"Good afternoon, Alice!"
```

---

## Step 8: Mix Commands ที่ต้องรู้

### 8.1 คำสั่งพื้นฐาน

```bash
# สร้าง project ใหม่
mix new project_name

# สร้าง Phoenix project
mix phx.new phoenix_project

# Compile
mix compile

# รัน test
mix test

# รัน test แบบ verbose
mix test --trace

# รัน test ไฟล์เดียว
mix test test/my_test.exs

# รัน test บรรทัดที่ระบุ
mix test test/my_test.exs:10

# เปิด IEx
iex -S mix

# รัน task พิเศษ
mix run -e "IO.puts('Hello')"
```

### 8.2 จัดการ Dependencies

```bash
# ดาวน์โหลด dependencies
mix deps.get

# Compile dependencies
mix deps.compile

# อัปเดต dependencies
mix deps.update --all

# ดูสถานะ dependencies
mix deps

# ลบ dependencies ที่ไม่ได้ใช้
mix deps.clean --unused
```

### 8.3 Format Code

```bash
# Format โค้ดทั้ง project
mix format

# ตรวจสอบว่า format แล้วหรือยัง
mix format --check-formatted
```

### 8.4 Documentation

```bash
# สร้าง documentation
mix docs

# เปิด documentation ใน browser
# ไฟล์จะอยู่ใน doc/ directory
```

---

## Step 9: เข้าใจ Elixir Ecosystem

### 9.1 Hex - Package Manager ของ Elixir

```
Hex.pm คือ package repository ของ Elixir
เหมือน npm ของ JavaScript หรือ PyPI ของ Python
```

```bash
# ดูข้อมูล package
mix hex.info package_name

# ค้นหา package
mix hex.search search_term

# ดู packages ที่ติดตั้งแล้ว
mix hex.packages
```

### 9.2 Popular Libraries

| Library | ใช้สำหรับ | Hex Package |
|---------|-----------|-------------|
| Phoenix | Web Framework | phoenix |
| Ecto | Database ORM | ecto |
| Plug | HTTP Middleware | plug |
| HTTPoison | HTTP Client | httpoison |
| Jason | JSON | jason |
| Poison | JSON | poison |
| Timex | Date/Time | timex |
| Credo | Code Analysis | credo |
| Dialyxir | Type Checking | dialyxir |
| ExDoc | Documentation | ex_doc |

### 9.3 เพิ่ม Dependency ลงใน Project

แก้ไข `mix.exs`:

```elixir
defp deps do
  [
    {:jason, "~> 1.4"},      # JSON library
    {:httpoison, "~> 2.0"},  # HTTP client
    {:timex, "~> 3.7"},      # Date/Time library
  ]
end
```

```bash
# ดาวน์โหลด dependencies
mix deps.get

# ลองใช้
iex -S mix
```

```elixir
# ใน IEx
iex(1)> Jason.encode!(%{name: "Alice", age: 30})
"{\"age\":30,\"name\":\"Alice\"}"

iex(2)> Jason.decode!("{\"name\":\"Alice\"}")
%{"name" => "Alice"}
```

---

## Step 10: Code Style และ Convention

### 10.1 Naming Conventions

```elixir
# Module names: PascalCase
defmodule MyModule do
defmodule UserAccount do
defmodule HTTPClient do

# Function names: snake_case
def get_user(id) do
def calculate_total_price(items) do

# Variables: snake_case
user_name = "Alice"
total_price = 100

# Constants/Atoms: snake_case or lowercase
:ok
:error
:not_found
MY_CONSTANT = 42  # สำหรับ module-level constants

# Boolean functions ลงท้ายด้วย ?
def valid?(email) do
def admin?(user) do

# Functions ที่ raise exception ลงท้ายด้วย !
def get_user!(id) do    # raise ถ้าไม่เจอ
def parse_json!(data) do  # raise ถ้า parse ไม่ได้
```

### 10.2 Code Formatting

```elixir
# ดี - ใช้ 2 spaces สำหรับ indentation
defmodule Good do
  def hello do
    IO.puts("Hello")
  end
end

# ไม่ดี - ใช้ tabs หรือ 4 spaces
defmodule Bad do
    def hello do
        IO.puts("Hello")
    end
end
```

### 10.3 Line Length

```elixir
# ดี - ไม่เกิน 98 ตัวอักษรต่อบรรทัด
def very_long_function_name(param_one, param_two, param_three) do
  # code here
end

# ดี - แยกบรรทัดเมื่อยาวเกิน
def very_long_function_name(
  param_one,
  param_two,
  param_three
) do
  # code here
end
```

### 10.4 Comments

```elixir
# Single line comment

# Module documentation (ใช้ @moduledoc)
defmodule MyModule do
  @moduledoc """
  This module handles user authentication.
  
  It provides functions for login, logout, and
  session management.
  """
  
  # Function documentation (ใช้ @doc)
  @doc """
  Authenticates a user with email and password.
  
  Returns `{:ok, user}` on success, `{:error, reason}` on failure.
  
  ## Examples
  
      iex> authenticate("alice@example.com", "password123")
      {:ok, %User{name: "Alice"}}
      
      iex> authenticate("alice@example.com", "wrong")
      {:error, :invalid_credentials}
  """
  def authenticate(email, password) do
    # implementation
  end
end
```

---

## สรุป Part 01

ใน Part นี้คุณได้เรียนรู้:

✅ **Step 1** - Elixir คืออะไรและทำไมถึงใช้  
✅ **Step 2** - เปรียบเทียบ Elixir กับภาษาอื่น  
✅ **Step 3** - แนวคิดหลัก: Functional, Actor Model, Let it Crash  
✅ **Step 4** - การติดตั้ง Elixir บน macOS, Linux, Windows  
✅ **Step 5** - IEx Interactive Shell  
✅ **Step 6** - โปรแกรมแรก Hello World  
✅ **Step 7** - สร้าง Mix Project แรก  
✅ **Step 8** - Mix Commands ที่จำเป็น  
✅ **Step 9** - Elixir Ecosystem (Hex, Libraries)  
✅ **Step 10** - Code Style และ Convention  

---

## แบบฝึกหัด Part 01

1. ติดตั้ง Elixir บนเครื่องของคุณ
2. ทดลองใช้ IEx ทำการคำนวณ 10 อย่าง
3. สร้าง Mix project ชื่อ `my_calculator`
4. เพิ่ม function ใน module:
   - `add(a, b)` - บวก
   - `subtract(a, b)` - ลบ
   - `multiply(a, b)` - คูณ
   - `divide(a, b)` - หาร
5. รัน `iex -S mix` และทดสอบทุก function

---

## ไปต่อ

➡️ [Part 02: ประเภทข้อมูลพื้นฐาน](./part-02-basic-types.md)
