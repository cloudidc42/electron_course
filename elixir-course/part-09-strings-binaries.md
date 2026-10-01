# Part 09: Strings และ Binaries (Steps 81-90)

## Step 81: String Internal Representation

### 81.1 String = UTF-8 Binary

```elixir
# String ใน Elixir คือ UTF-8 encoded binary
iex> is_binary("Hello")
true

iex> byte_size("Hello")
5

iex> byte_size("สวัสดี")
21  # UTF-8 ใช้ 3-4 bytes ต่อ Thai character

iex> String.length("Hello")
5

iex> String.length("สวัสดี")
6  # 6 grapheme clusters

# ดูว่า String valid UTF-8 หรือไม่
iex> String.valid?("Hello")
true

iex> String.valid?(<<0xFF>>)  # invalid UTF-8 byte
false
```

### 81.2 Grapheme Clusters

```elixir
# Grapheme cluster = "ตัวอักษรที่มองเห็น" แม้ว่าจะประกอบจากหลาย codepoints
iex> String.graphemes("Hello")
["H", "e", "l", "l", "o"]

# อักษรที่มี combining characters
iex> String.graphemes("é")  # é (e + combining accent)
["é"]  # นับเป็น 1 grapheme

iex> String.length("é")
1

iex> byte_size("é")
3  # e = 1 byte, combining accent = 2 bytes
```

### 81.3 Codepoints

```elixir
# Unicode codepoints
iex> String.codepoints("Hello")
["H", "e", "l", "l", "o"]

iex> String.to_charlist("Hello")
[72, 101, 108, 108, 111]

# ดู codepoint ของ character ด้วย ?
iex> ?A
65

iex> ?a
97

iex> ?ก
3585

# แปลง codepoint เป็น character
iex> <<65::utf8>>
"A"

iex> <<3585::utf8>>
"ก"
```

---

## Step 82: String Operations ครบวงจร

### 82.1 String Inspection

```elixir
str = "Hello, World! สวัสดี"

String.length(str)          # 20 (characters)
byte_size(str)               # (bytes)
String.valid?(str)           # true

# ตรวจสอบ content
String.contains?(str, "World")    # true
String.contains?(str, "Elixir")  # false
String.contains?(str, ["World", "Elixir"])  # true (any match)

String.starts_with?(str, "Hello")  # true
String.ends_with?(str, "สวัสดี")   # true

String.match?(str, ~r/\d+/)  # false - ไม่มีตัวเลข
```

### 82.2 String Manipulation

```elixir
# Case conversion
String.upcase("hello world")      # "HELLO WORLD"
String.downcase("HELLO WORLD")    # "hello world"
String.capitalize("hello world")  # "Hello world" (capitalize แค่ตัวแรก)

# สำหรับ title case (capitalize แต่ละคำ)
"hello world elixir"
|> String.split()
|> Enum.map(&String.capitalize/1)
|> Enum.join(" ")
# "Hello World Elixir"

# Trimming
String.trim("  hello  ")         # "hello"
String.trim_leading("  hello  ") # "hello  "
String.trim_trailing("  hello  ") # "  hello"
String.trim("xxhelloxx", "x")    # "hello" (trim specific chars)

# Padding
String.pad_leading("42", 5)       # "   42"
String.pad_leading("42", 5, "0")  # "00042"
String.pad_trailing("hi", 5)      # "hi   "
String.pad_trailing("hi", 5, "!")  # "hi!!!"
```

### 82.3 String Splitting

```elixir
# Split by whitespace (default)
String.split("hello world elixir")
# ["hello", "world", "elixir"]

# Split by delimiter
String.split("a,b,c,d", ",")
# ["a", "b", "c", "d"]

# Split with limit
String.split("a,b,c,d", ",", parts: 2)
# ["a", "b,c,d"]

# Split กับ regex
String.split("one1two2three", ~r/\d/)
# ["one", "two", "three"]

# Split and trim
String.split("  a , b , c  ", ",", trim: true)
# ["  a ", " b ", " c  "]

"  a , b , c  "
|> String.split(",")
|> Enum.map(&String.trim/1)
# ["a", "b", "c"]
```

### 82.4 String Replacement

```elixir
str = "The quick brown fox"

# Replace all
String.replace(str, "o", "0")
# "The quick br0wn f0x"

# Replace first only
String.replace(str, "o", "0", global: false)
# "The quick br0wn fox"

# Replace with regex
String.replace(str, ~r/[aeiou]/, "*")
# "Th* q**ck br*wn f*x"

# Replace with function
String.replace(str, ~r/\b\w+\b/, fn word ->
  String.capitalize(word)
end)
# "The Quick Brown Fox"
```

---

## Step 83: Regular Expressions

### 83.1 Regex พื้นฐาน

```elixir
# สร้าง regex ด้วย ~r sigil
pattern = ~r/hello/
pattern_i = ~r/hello/i  # case insensitive

# Match
Regex.match?(~r/\d+/, "abc123")    # true
Regex.match?(~r/^\d+$/, "abc123")  # false

# String.match?
String.match?("hello world", ~r/hello/)   # true
String.match?("HELLO", ~r/hello/i)        # true

# Run - ดู match groups
Regex.run(~r/(\d{4})-(\d{2})-(\d{2})/, "2024-01-15")
# ["2024-01-15", "2024", "01", "15"]

# Named captures
Regex.named_captures(~r/(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})/, "2024-01-15")
# %{"day" => "15", "month" => "01", "year" => "2024"}
```

### 83.2 Regex ขั้นสูง

```elixir
# scan - หาทุก matches
Regex.scan(~r/\d+/, "abc 123 def 456 ghi 789")
# [["123"], ["456"], ["789"]]

# กับ groups
Regex.scan(~r/(\w+)=(\w+)/, "a=1 b=2 c=3")
# [["a=1", "a", "1"], ["b=2", "b", "2"], ["c=3", "c", "3"]]

# split ด้วย regex
Regex.split(~r/\s+/, "  hello   world  ")
# ["", "hello", "world", ""]

Regex.split(~r/\s+/, "  hello   world  ", trim: true)
# ["hello", "world"]
```

### 83.3 Practical Regex Examples

```elixir
defmodule Validator do
  @email_regex ~r/^[a-zA-Z0-9.!#$%&'*+\/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$/
  @phone_regex ~r/^(\+66|0)[0-9]{9}$/
  @url_regex ~r/^https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&\/=]*)$/
  
  def valid_email?(email), do: String.match?(email, @email_regex)
  def valid_phone?(phone), do: String.match?(phone, @phone_regex)
  def valid_url?(url), do: String.match?(url, @url_regex)
  
  def validate_user(attrs) do
    errors = []
    
    errors = if valid_email?(attrs[:email] || ""), 
      do: errors, 
      else: ["Invalid email" | errors]
    
    errors = if attrs[:phone] == nil or valid_phone?(attrs[:phone]),
      do: errors,
      else: ["Invalid phone" | errors]
    
    case errors do
      [] -> {:ok, attrs}
      errors -> {:error, Enum.reverse(errors)}
    end
  end
end

Validator.valid_email?("alice@example.com")   # true
Validator.valid_email?("not-an-email")        # false
Validator.valid_phone?("0812345678")           # true
Validator.valid_phone?("+66812345678")         # true
```

---

## Step 84: Sigils

Sigils คือ shorthand สำหรับสร้าง data types

### 84.1 Built-in Sigils

```elixir
# ~r - Regular Expression
~r/hello/

# ~s - String (รองรับ escape sequences เหมือน double-quoted string)
~s(Hello\nWorld)
# "Hello\nWorld"

# ~S - String (ไม่ process escape sequences)
~S(Hello\nWorld)
# "Hello\\nWorld"

# ~c - Charlist
~c(hello)
# 'hello' = [104, 101, 108, 108, 111]

# ~w - Word list
~w(one two three)
# ["one", "two", "three"]

~w(one two three)a  # atom list
# [:one, :two, :three]

~w(one two three)c  # charlist
# ['one', 'two', 'three']

# ~D - Date
~D[2024-01-15]
# ~D[2024-01-15]

# ~T - Time
~T[13:30:00]
# ~T[13:30:00]

# ~N - NaiveDateTime
~N[2024-01-15 13:30:00]
# ~N[2024-01-15 13:30:00]

# ~U - UTC DateTime
~U[2024-01-15 13:30:00Z]
# ~U[2024-01-15 13:30:00Z]
```

### 84.2 Heredoc Sigils

```elixir
# Multi-line strings
html = ~s"""
<div class="container">
  <h1>Hello, World!</h1>
  <p>This is a paragraph</p>
</div>
"""

sql = ~s"""
SELECT users.*, profiles.*
FROM users
JOIN profiles ON profiles.user_id = users.id
WHERE users.active = true
  AND users.role = 'admin'
ORDER BY users.created_at DESC
LIMIT 10
"""
```

### 84.3 Custom Sigils

```elixir
defmodule MySigils do
  # Custom sigil สำหรับ Thai text normalization
  def sigil_t(string, []) do
    string
    |> String.trim()
    |> String.normalize(:nfc)  # Unicode normalization
  end
  
  # Custom sigil สำหรับ CSV
  def sigil_CSV(string, _opts) do
    string
    |> String.split("\n")
    |> Enum.filter(&(String.trim(&1) != ""))
    |> Enum.map(&String.split(&1, ","))
  end
end

import MySigils

text = ~t(  สวัสดีครับ  )
# "สวัสดีครับ"

data = ~CSV"""
Alice,30,Bangkok
Bob,25,Chiang Mai
"""
# [["Alice", "30", "Bangkok"], ["Bob", "25", "Chiang Mai"]]
```

---

## Step 85: Binaries ขั้นสูง

### 85.1 Binary Syntax

```elixir
# สร้าง Binary
<<1, 2, 3>>           # binary จาก bytes
<<65, 66, 67>>        # "ABC"
<<"hello">>           # binary จาก string
<<0xFF, 0x00>>        # hex notation

# ระบุ bit size
<<1::8>>    # 1 byte
<<255::8>>  # maximum byte value
<<1::16>>   # 2 bytes (big-endian by default)
<<1::32>>   # 4 bytes
<<1::64>>   # 8 bytes

# Float
<<3.14::float-32>>   # 4-byte float
<<3.14::float-64>>   # 8-byte float

# String ใน binary
<<104, 101, 108, 108, 111>> == "hello"  # true
```

### 85.2 Binary Pattern Matching

```elixir
# Parse binary data
<<version::8, type::8, length::16, data::binary>> = packet

# Parse IP header
<<version::4, ihl::4, dscp::6, ecn::2, 
  total_length::16, id::16, flags::3, offset::13,
  ttl::8, protocol::8, checksum::16,
  src_ip::32, dst_ip::32,
  rest::binary>> = ip_packet

# ดึง IP address
<<a, b, c, d>> = <<src_ip::32>>
"#{a}.#{b}.#{c}.#{d}"  # "192.168.1.1"

# Parse PNG header
<<137, 80, 78, 71, 13, 10, 26, 10, rest::binary>> = png_data
# ตรวจสอบว่า valid PNG
```

### 85.3 Binary Comprehension

```elixir
# map บน binary
for <<byte <- "hello">>, into: <<>>, do: <<byte + 1>>
# <<105, 102, 109, 109, 112>>  = "ifmmp"

# filter binary bytes
for <<byte <- data>>, byte != 0, into: <<>>, do: <<byte>>
# ลบ null bytes ออก

# Parse pixel data
pixels = <<255, 0, 0, 0, 255, 0, 0, 0, 255>>  # RGB values

for <<r::8, g::8, b::8 <- pixels>> do
  %{r: r, g: g, b: b}
end
# [%{b: 0, g: 0, r: 255}, %{b: 0, g: 255, r: 0}, %{b: 255, g: 0, r: 0}]
```

---

## Step 86: String Encoding และ Decoding

### 86.1 Base64

```elixir
# Encode
Base.encode64("Hello, World!")
# "SGVsbG8sIFdvcmxkIQ=="

# Decode
Base.decode64!("SGVsbG8sIFdvcmxkIQ==")
# "Hello, World!"

Base.decode64("invalid!")
# {:error, :malformed_padding}

# URL-safe Base64
Base.url_encode64("Hello, World!")
Base.url_decode64!("SGVsbG8sIFdvcmxkIQ==")

# Hex encoding
Base.encode16(<<255, 0, 128>>)
# "FF0080"

Base.decode16!("FF0080")
# <<255, 0, 128>>
```

### 86.2 JSON กับ Jason

```elixir
# เพิ่ม dependency ใน mix.exs:
# {:jason, "~> 1.4"}

# Encode
Jason.encode!(%{name: "Alice", age: 30})
# "{\"age\":30,\"name\":\"Alice\"}"

Jason.encode!([1, 2, 3])
# "[1,2,3]"

# Pretty print
Jason.encode!(%{name: "Alice"}, pretty: true)
# "{\n  \"name\": \"Alice\"\n}"

# Decode
Jason.decode!("{\"name\":\"Alice\",\"age\":30}")
# %{"age" => 30, "name" => "Alice"}

# Keys เป็น atoms (ระวัง! atoms ไม่ถูก GC)
Jason.decode!("{\"name\":\"Alice\"}", keys: :atoms)
# %{age: 30, name: "Alice"}

# Handle error
case Jason.decode(invalid_json) do
  {:ok, data}    -> process(data)
  {:error, error} -> {:error, "Invalid JSON: #{inspect(error)}"}
end
```

### 86.3 URI Encoding

```elixir
URI.encode("Hello, World!")
# "Hello%2C%20World%21"

URI.decode("Hello%2C%20World%21")
# "Hello, World!"

URI.encode_query([name: "Alice Wonderland", age: 30])
# "age=30&name=Alice+Wonderland"

URI.decode_query("name=Alice+Wonderland&age=30")
# %{"age" => "30", "name" => "Alice Wonderland"}

# Parse URL
URI.parse("https://user:pass@example.com:8080/path?key=value#section")
# %URI{
#   scheme: "https",
#   userinfo: "user:pass",
#   host: "example.com",
#   port: 8080,
#   path: "/path",
#   query: "key=value",
#   fragment: "section"
# }
```

---

## Step 87: String Formatting

### 87.1 IO.puts, IO.write, IO.inspect

```elixir
# IO.puts - print with newline
IO.puts("Hello, World!")

# IO.write - print without newline
IO.write("Hello")
IO.write(", World!\n")

# IO.inspect - debug print (return ค่าเดิม)
[1, 2, 3] |> IO.inspect(label: "list") |> Enum.sum()
# list: [1, 2, 3]
# 6

# Format options
IO.inspect(%{a: 1}, pretty: true, width: 40)
IO.inspect([1,2,3,4,5], limit: 3)  # [1, 2, 3, ...]
```

### 87.2 :io_lib.format (Erlang)

```elixir
# Erlang-style formatting
:io_lib.format("~p~n", ["hello"])
# ใช้ ~p สำหรับ inspect, ~s สำหรับ string, ~d สำหรับ integer

# แปลงเป็น String
IO.iodata_to_binary(:io_lib.format("Value: ~p", [42]))
# "Value: 42"
```

### 87.3 String.Chars Protocol

```elixir
# ทุก type ที่ implement String.Chars สามารถใช้ใน interpolation ได้
defmodule Temperature do
  defstruct [:value, :unit]
  
  defimpl String.Chars do
    def to_string(%{value: v, unit: :celsius}) do
      "#{v}°C"
    end
    
    def to_string(%{value: v, unit: :fahrenheit}) do
      "#{v}°F"
    end
  end
end

temp = %Temperature{value: 25, unit: :celsius}
"Current temperature: #{temp}"
# "Current temperature: 25°C"
```

---

## Step 88: IOData และ CharData

### 88.1 IOData

```elixir
# IOData คือ binary, charlist, หรือ nested list ของทั้งคู่
# ใช้สร้าง strings อย่างมีประสิทธิภาพ

iodata = ["Hello", ", ", "World", "!"]
IO.iodata_to_binary(iodata)
# "Hello, World!"

# Building large strings efficiently
defmodule HTMLBuilder do
  def render(title, items) do
    [
      "<html><head><title>", title, "</title></head>",
      "<body><ul>",
      Enum.map(items, fn item -> ["<li>", item, "</li>"] end),
      "</ul></body></html>"
    ]
    |> IO.iodata_to_binary()
  end
end

HTMLBuilder.render("My List", ["Item 1", "Item 2", "Item 3"])
```

### 88.2 EEx Templates

```elixir
# EEx (Embedded Elixir) ใช้สำหรับ templates
# ต้องเพิ่ม {:eex, "~> 1.0"} ใน mix.exs

template = "Hello, <%= name %>! You are <%= age %> years old."
EEx.eval_string(template, name: "Alice", age: 30)
# "Hello, Alice! You are 30 years old."

# Template files
EEx.eval_file("template.html.eex", assigns: %{title: "My Page"})
```

---

## Step 89: String Performance

### 89.1 IOData vs String Concatenation

```elixir
# ช้า - String concatenation สร้าง binary ใหม่ทุกครั้ง
def build_bad(n) do
  Enum.reduce(1..n, "", fn i, acc ->
    acc <> "item#{i},"
  end)
end

# เร็ว - สร้าง IOData แล้ว convert ทีเดียว
def build_good(n) do
  1..n
  |> Enum.map(fn i -> "item#{i}," end)
  |> IO.iodata_to_binary()
end

# เร็วที่สุด - เฉพาะ case ง่ายๆ
def build_best(n) do
  Enum.join(Enum.map(1..n, fn i -> "item#{i}" end), ",")
end
```

### 89.2 String.bag_distance สำหรับ Fuzzy Match

```elixir
# bag_distance ใช้วัดความคล้ายของ strings
String.bag_distance("alice", "alice")   # 1.0 (identical)
String.bag_distance("alice", "alicE")   # 0.8
String.bag_distance("alice", "bob")     # 0.0

# jaro_distance
String.jaro_distance("alice", "alice")  # 1.0
String.jaro_distance("alice", "alcie")  # 0.933...
String.jaro_distance("abc", "xyz")      # 0.0

# ใช้สำหรับ autocomplete
defmodule Autocomplete do
  def suggest(query, candidates, threshold \\ 0.7) do
    candidates
    |> Enum.map(fn candidate ->
         {candidate, String.jaro_distance(query, candidate)}
       end)
    |> Enum.filter(fn {_, score} -> score >= threshold end)
    |> Enum.sort_by(fn {_, score} -> score end, :desc)
    |> Enum.map(fn {candidate, _} -> candidate end)
  end
end

words = ["apple", "application", "apply", "apartment", "banana"]
Autocomplete.suggest("app", words)
# ["apple", "apply", "application"]
```

---

## Step 90: Template Strings และ Text Processing

### 90.1 String Processing Pipeline

```elixir
defmodule TextProcessor do
  def process(text, opts \\ []) do
    text
    |> maybe_trim(opts)
    |> maybe_normalize_whitespace(opts)
    |> maybe_remove_html(opts)
    |> maybe_truncate(opts)
  end
  
  defp maybe_trim(text, opts) do
    if Keyword.get(opts, :trim, true), do: String.trim(text), else: text
  end
  
  defp maybe_normalize_whitespace(text, opts) do
    if Keyword.get(opts, :normalize_whitespace, false) do
      Regex.replace(~r/\s+/, text, " ")
    else
      text
    end
  end
  
  defp maybe_remove_html(text, opts) do
    if Keyword.get(opts, :remove_html, false) do
      Regex.replace(~r/<[^>]*>/, text, "")
    else
      text
    end
  end
  
  defp maybe_truncate(text, opts) do
    case Keyword.get(opts, :max_length) do
      nil -> text
      max ->
        if String.length(text) > max do
          String.slice(text, 0, max - 3) <> "..."
        else
          text
        end
    end
  end
end

TextProcessor.process("  Hello   World  ",
  trim: true,
  normalize_whitespace: true,
  max_length: 8
)
# "Hello..."
```

### 90.2 Template Engine

```elixir
defmodule SimpleTemplate do
  def render(template, vars) when is_binary(template) and is_map(vars) do
    Regex.replace(~r/\{\{(\w+)\}\}/, template, fn _, key ->
      key_atom = String.to_existing_atom(key)
      Map.get(vars, key_atom, "") |> to_string()
    end)
  end
end

template = """
Dear {{name}},

Thank you for your order #{{order_id}}.
Your total is {{total}} THB.

Best regards,
{{company}}
"""

SimpleTemplate.render(template, %{
  name: "Alice",
  order_id: "ORD-12345",
  total: "1,500",
  company: "My Shop"
})
```

### 90.3 Levenshtein Distance

```elixir
defmodule StringDistance do
  def levenshtein(s, t) do
    s_chars = String.graphemes(s)
    t_chars = String.graphemes(t)
    
    s_len = length(s_chars)
    t_len = length(t_chars)
    
    # สร้าง matrix
    matrix = for i <- 0..s_len, into: %{} do
      {i, %{0 => i}}
    end
    
    matrix = for j <- 1..t_len, reduce: matrix do
      m -> put_in(m, [0, j], j)
    end
    
    Enum.reduce(Enum.with_index(s_chars, 1), matrix, fn {sc, i}, matrix ->
      Enum.reduce(Enum.with_index(t_chars, 1), matrix, fn {tc, j}, matrix ->
        cost = if sc == tc, do: 0, else: 1
        
        val = Enum.min([
          matrix[i-1][j] + 1,     # deletion
          matrix[i][j-1] + 1,     # insertion
          matrix[i-1][j-1] + cost # substitution
        ])
        
        put_in(matrix, [i, j], val)
      end)
    end)
    |> get_in([s_len, t_len])
  end
end

StringDistance.levenshtein("kitten", "sitting")   # 3
StringDistance.levenshtein("hello", "hello")       # 0
StringDistance.levenshtein("hello", "world")       # 4
```

---

## สรุป Part 09: Strings และ Binaries

✅ **Step 81** - String internal representation (UTF-8 binary)  
✅ **Step 82** - String operations ครบวงจร  
✅ **Step 83** - Regular Expressions  
✅ **Step 84** - Sigils  
✅ **Step 85** - Binaries ขั้นสูง  
✅ **Step 86** - Encoding และ Decoding  
✅ **Step 87** - String Formatting  
✅ **Step 88** - IOData  
✅ **Step 89** - String Performance  
✅ **Step 90** - Template Strings  

---

## แบบฝึกหัด Part 09

1. สร้าง `MarkdownParser` ที่แปลง Markdown เป็น HTML
2. สร้าง `QueryBuilder` ที่สร้าง SQL WHERE clause จาก Map
3. Implement `Tokenizer` ที่แยก source code เป็น tokens
4. สร้าง `Diff` ที่เปรียบเทียบสอง string ด้วย Levenshtein
5. เขียน Binary parser สำหรับ BMP image format

---

➡️ [Part 10: Processes พื้นฐาน](./part-10-processes.md)
