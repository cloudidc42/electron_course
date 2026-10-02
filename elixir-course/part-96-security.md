# Part 96: Security Hardening (Steps 1021-1030)

## Step 1021: OWASP Top 10 Prevention

```elixir
# A01: Broken Access Control
defmodule MyAppWeb.Plugs.ResourceOwnership do
  import Plug.Conn
  import Phoenix.Controller

  def init(opts), do: opts

  def call(conn, resource_module: module) do
    resource_id = conn.params["id"]
    user        = conn.assigns.current_user

    resource = MyApp.Repo.get(module, resource_id)

    cond do
      is_nil(resource) ->
        conn |> put_status(404) |> json(%{error: "not_found"}) |> halt()
      resource.user_id != user.id and not admin?(user) ->
        conn |> put_status(403) |> json(%{error: "forbidden"}) |> halt()
      true ->
        assign(conn, :resource, resource)
    end
  end

  defp admin?(user), do: user.role == "admin"
end

# A02: Cryptographic Failures
defmodule MyApp.Crypto do
  @aes_key :crypto.strong_rand_bytes(32)  # Store in vault, not code

  def encrypt(plaintext) do
    iv = :crypto.strong_rand_bytes(12)
    {ciphertext, tag} = :crypto.crypto_one_time_aead(
      :aes_256_gcm, @aes_key, iv, plaintext, "", true
    )
    # iv + tag + ciphertext, base64 encoded
    Base.encode64(iv <> tag <> ciphertext)
  end

  def decrypt(encoded) do
    <<iv::binary-12, tag::binary-16, ciphertext::binary>> = Base.decode64!(encoded)
    :crypto.crypto_one_time_aead(:aes_256_gcm, @aes_key, iv, ciphertext, "", tag, false)
  end
end

# A03: Injection Prevention
defmodule MyApp.InputSanitizer do
  def sanitize_html(input) do
    # Use HtmlSanitize library
    HtmlSanitizeEx.strip_tags(input)
  end

  def validate_identifier(id) when is_integer(id) and id > 0, do: {:ok, id}
  def validate_identifier(_), do: {:error, "invalid_id"}

  def prevent_path_traversal(path) do
    safe_path = Path.expand(path)
    base_dir  = Path.expand("/allowed/base/dir")

    if String.starts_with?(safe_path, base_dir) do
      {:ok, safe_path}
    else
      {:error, "path_traversal_detected"}
    end
  end
end
```

---

## Step 1022: SQL Injection Prevention

```elixir
defmodule MyApp.Database.SafeQuery do
  # Always use parameterized queries with Ecto

  # BAD - never do this
  def bad_example(name) do
    MyApp.Repo.query!("SELECT * FROM users WHERE name = '#{name}'")
  end

  # GOOD - parameterized query
  def safe_example(name) do
    MyApp.Repo.query!("SELECT * FROM users WHERE name = $1", [name])
  end

  # GOOD - Ecto query
  def ecto_example(name) do
    import Ecto.Query
    from(u in User, where: u.name == ^name) |> MyApp.Repo.all()
  end

  # Dynamic queries - still safe with Ecto
  def dynamic_filters(filters) do
    import Ecto.Query

    Enum.reduce(filters, from(p in Product), fn
      {"name", name}, query ->
        where(query, [p], ilike(p.name, ^"%#{sanitize_like(name)}%"))
      {"min_price", price}, query ->
        where(query, [p], p.price >= ^price)
      _, query ->
        query
    end)
    |> MyApp.Repo.all()
  end

  # Escape LIKE special characters
  defp sanitize_like(str) do
    str
    |> String.replace("\\", "\\\\")
    |> String.replace("%", "\\%")
    |> String.replace("_", "\\_")
  end
end
```

---

## Step 1023: XSS Prevention

```elixir
defmodule MyApp.XSSPrevention do
  # Phoenix auto-escapes HEEx templates
  # Manual escaping when needed

  def safe_json(data) do
    # Use Phoenix.HTML.json/1 or Jason with escape option
    data
    |> Jason.encode!()
    |> String.replace("<", "\\u003c")
    |> String.replace(">", "\\u003e")
    |> String.replace("&", "\\u0026")
    |> String.replace("'", "\\u0027")
  end

  # Content Security Policy nonce
  def generate_nonce do
    Base.encode64(:crypto.strong_rand_bytes(16))
  end

  # Trusted Types for DOM manipulation
  def csp_with_nonce(nonce) do
    "default-src 'self'; " <>
    "script-src 'self' 'nonce-#{nonce}'; " <>
    "style-src 'self' 'nonce-#{nonce}'; " <>
    "object-src 'none'; " <>
    "base-uri 'self'; " <>
    "require-trusted-types-for 'script'"
  end
end

# Phoenix template auto-escaping
defmodule MyAppWeb.SafeHTML do
  use Phoenix.Component
  import Phoenix.HTML

  # This is automatically escaped by HEEx:
  # <p><%= @user_input %></p>

  # For raw HTML (use only for trusted content):
  def trusted_html(assigns) do
    ~H"""
    <div>{Phoenix.HTML.raw(@sanitized_content)}</div>
    """
  end
end
```

---

## Step 1024: CSRF Protection

```elixir
defmodule MyAppWeb.CSRFProtection do
  # Phoenix includes built-in CSRF protection

  # In router.ex - auto-enabled for :browser pipeline
  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
    plug :protect_from_forgery   # CSRF protection
    plug :put_secure_browser_headers
  end

  # For API endpoints, use token-based auth instead of sessions
  pipeline :api do
    plug :accepts, ["json"]
    plug MyAppWeb.Plugs.APIAuth
    # No :protect_from_forgery for APIs
  end

  # SameSite cookie attribute
  config :my_app, MyAppWeb.Endpoint,
    secret_key_base: System.get_env("SECRET_KEY_BASE"),
    session_options: [
      store:    :cookie,
      key:      "_my_app_key",
      same_site: "Strict",  # Or "Lax" for OAuth flows
      secure:   true,       # HTTPS only
      http_only: true       # No JS access
    ]
end
```

---

## Step 1025: Penetration Testing with ExUnit

```elixir
defmodule MyApp.SecurityTest do
  use ExUnit.Case, async: false
  use MyAppWeb.ConnCase

  describe "SQL injection prevention" do
    test "user search is safe against injection" do
      payloads = [
        "'; DROP TABLE users; --",
        "1' OR '1'='1",
        "admin'--",
        "1; SELECT pg_sleep(10)--"
      ]
      
      Enum.each(payloads, fn payload ->
        # Should return empty results, not error or data leak
        result = MyApp.Accounts.search_users(payload)
        assert is_list(result)
        assert length(result) == 0
      end)
    end
  end

  describe "path traversal prevention" do
    test "rejects path traversal attempts" do
      attacks = [
        "../../../etc/passwd",
        "..%2F..%2Fetc%2Fpasswd",
        "....//....//etc/passwd"
      ]
      
      Enum.each(attacks, fn path ->
        assert {:error, _} = MyApp.InputSanitizer.prevent_path_traversal(path)
      end)
    end
  end

  describe "XSS prevention" do
    test "user content is escaped in HTML" do
      xss = ~s(<script>alert('xss')</script>)
      safe = MyApp.XSSPrevention.safe_json(%{content: xss})
      refute String.contains?(safe, "<script>")
    end
  end

  describe "authentication" do
    test "brute force is rate limited" do
      for _ <- 1..10 do
        post(build_conn(), "/api/auth/login", %{
          email: "attacker@evil.com",
          password: "wrong"
        })
      end
      
      last_response = post(build_conn(), "/api/auth/login", %{
        email: "attacker@evil.com",
        password: "wrong"
      })
      
      assert last_response.status == 429
    end
  end
end
```

---

## Step 1026: Secrets Scanning

```elixir
defmodule MyApp.SecretsScanner do
  # Detect accidentally committed secrets

  @patterns [
    {~r/AKIA[0-9A-Z]{16}/,         "AWS Access Key"},
    {~r/(?i)api[_-]?key.{0,20}"[a-z0-9]{20,}/,  "API Key"},
    {~r/(?i)password\s*=\s*"[^"]+"/, "Hardcoded Password"},
    {~r/(?i)secret\s*=\s*"[^"]+"/, "Hardcoded Secret"},
    {~r/-----BEGIN (?:RSA|EC|DSA) PRIVATE KEY-----/, "Private Key"}
  ]

  def scan_file(path) do
    case File.read(path) do
      {:ok, content} ->
        findings = Enum.flat_map(@patterns, fn {pattern, name} ->
          case Regex.scan(pattern, content) do
            []      -> []
            matches -> Enum.map(matches, fn [match | _] ->
              %{file: path, type: name, match: redact(match)}
            end)
          end
        end)
        {:ok, findings}
      {:error, reason} ->
        {:error, reason}
    end
  end

  def scan_directory(dir) do
    dir
    |> File.ls!()
    |> Enum.flat_map(fn file ->
      path = Path.join(dir, file)
      if File.dir?(path) and not excluded?(file) do
        scan_directory(path)
      else
        case scan_file(path) do
          {:ok, findings} -> findings
          {:error, _}     -> []
        end
      end
    end)
  end

  defp excluded?(name), do: name in ["_build", "deps", ".git", "node_modules"]
  defp redact(match),   do: String.slice(match, 0, 4) <> "***"
end
```

---

## Step 1027: Rate Limiting & DDoS Protection

```elixir
defmodule MyApp.Protection.RateLimiter do
  # Sliding window rate limiter using Redis

  def check(key, limit, window_seconds) do
    now     = System.os_time(:millisecond)
    min_time = now - window_seconds * 1000
    
    pipeline = [
      ["ZREMRANGEBYSCORE", key, "-inf", min_time],
      ["ZADD", key, now, "#{now}"],
      ["ZCARD", key],
      ["EXPIRE", key, window_seconds + 1]
    ]

    {:ok, results} = Redix.pipeline(:redix, pipeline)
    count = Enum.at(results, 2)

    if count > limit do
      {:error, :rate_limited, count}
    else
      {:ok, limit - count}
    end
  end
end

# IP allowlist/blocklist
defmodule MyApp.Protection.IPFilter do
  def call(conn, _opts) do
    ip = conn.remote_ip |> :inet.ntoa() |> to_string()
    
    cond do
      blocked?(ip) ->
        conn |> Plug.Conn.send_resp(403, "Forbidden") |> Plug.Conn.halt()
      allowed?(ip) ->
        conn
      true ->
        conn
    end
  end

  def block(ip) do
    Redix.command(:redix, ["SADD", "blocked_ips", ip])
    Logger.warning("Blocked IP: #{ip}")
  end

  def block_range(cidr) do
    # Block entire CIDR range
    Redix.command(:redix, ["SADD", "blocked_cidrs", cidr])
  end

  defp blocked?(ip) do
    {:ok, 1} == Redix.command(:redix, ["SISMEMBER", "blocked_ips", ip])
  end

  defp allowed?(ip) do
    {:ok, 1} == Redix.command(:redix, ["SISMEMBER", "allowed_ips", ip])
  end
end
```

---

## Step 1028: Dependency Vulnerability Scanning

```elixir
defmodule MyApp.Security.DependencyAudit do
  # Check mix.lock for known vulnerabilities
  # Use: mix hex.audit

  def run_audit do
    case System.cmd("mix", ["hex.audit"], cd: File.cwd!()) do
      {output, 0}    ->
        Logger.info("Dependency audit passed")
        {:ok, output}
      {output, code} ->
        Logger.warning("Dependency audit found issues (exit #{code}):\n#{output}")
        {:error, output}
    end
  end

  # Sobelow for static analysis
  def run_sobelow do
    case System.cmd("mix", ["sobelow", "--format", "json"], cd: File.cwd!()) do
      {json, _} ->
        findings = Jason.decode!(json)
        critical = Enum.filter(findings, & &1["severity"] == "high")
        
        if length(critical) > 0 do
          Logger.error("Critical security issues found: #{inspect(critical)}")
          {:error, :critical_vulnerabilities, critical}
        else
          {:ok, findings}
        end
    end
  end

  # Check for outdated dependencies
  def check_outdated do
    System.cmd("mix", ["hex.outdated"])
  end
end
```

---

## Step 1029: Encryption at Rest

```elixir
defmodule MyApp.Encryption do
  # Transparent field-level encryption using Ecto.Type

  defmodule EncryptedString do
    use Ecto.Type

    @key Application.compile_env(:my_app, :encryption_key) |>
         Base.decode64!()

    def type,  do: :binary
    def equal?(a, b), do: a == b

    def cast(value) when is_binary(value), do: {:ok, value}
    def cast(_), do: :error

    def dump(value) when is_binary(value) do
      iv = :crypto.strong_rand_bytes(12)
      {ciphertext, tag} = :crypto.crypto_one_time_aead(
        :aes_256_gcm, @key, iv, value, "", true
      )
      {:ok, iv <> tag <> ciphertext}
    end

    def load(value) when is_binary(value) do
      <<iv::binary-12, tag::binary-16, ciphertext::binary>> = value
      case :crypto.crypto_one_time_aead(:aes_256_gcm, @key, iv, ciphertext, "", tag, false) do
        plaintext when is_binary(plaintext) -> {:ok, plaintext}
        :error -> :error
      end
    end
  end
end

# Usage in schema
defmodule MyApp.User do
  use Ecto.Schema

  schema "users" do
    field :email,        :string
    field :ssn,          MyApp.Encryption.EncryptedString
    field :credit_card,  MyApp.Encryption.EncryptedString
    field :phone,        MyApp.Encryption.EncryptedString

    timestamps()
  end
end
```

---

## Step 1030: Security Monitoring

```elixir
defmodule MyApp.SecurityMonitor do
  use GenServer
  require Logger

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def report_event(event_type, details) do
    GenServer.cast(__MODULE__, {:event, event_type, details})
  end

  def init([]) do
    :timer.send_interval(60_000, :analyze)
    {:ok, %{events: [], alerts_sent: MapSet.new()}}
  end

  def handle_cast({:event, type, details}, state) do
    event = %{type: type, details: details, at: DateTime.utc_now()}
    {:noreply, %{state | events: [event | state.events]}}
  end

  def handle_info(:analyze, state) do
    analyze_patterns(state.events)
    # Keep only last hour of events
    cutoff = DateTime.add(DateTime.utc_now(), -3600)
    recent = Enum.filter(state.events, fn e -> DateTime.compare(e.at, cutoff) == :gt end)
    {:noreply, %{state | events: recent}}
  end

  defp analyze_patterns(events) do
    # Failed logins from same IP
    failed_logins = events
      |> Enum.filter(& &1.type == :failed_login)
      |> Enum.group_by(& &1.details.ip)
      |> Enum.filter(fn {_, v} -> length(v) > 10 end)

    Enum.each(failed_logins, fn {ip, attempts} ->
      Logger.warning("Brute force detected from #{ip}: #{length(attempts)} attempts")
      :telemetry.execute([:security, :brute_force], %{count: length(attempts)}, %{ip: ip})
      MyApp.Protection.IPFilter.block(ip)
    end)

    # Privilege escalation attempts
    priv_attempts = Enum.count(events, & &1.type == :privilege_escalation_attempt)
    if priv_attempts > 0 do
      Logger.error("#{priv_attempts} privilege escalation attempts detected!")
      send_security_alert("Privilege escalation attempt", %{count: priv_attempts})
    end
  end

  defp send_security_alert(subject, data) do
    Task.start(fn ->
      MyApp.Emails.security_alert(subject, data) |> MyApp.Mailer.deliver()
    end)
  end
end
```

---

## สรุป Part 96

✅ **Step 1021** - OWASP Top 10 prevention  
✅ **Step 1022** - SQL injection prevention  
✅ **Step 1023** - XSS prevention  
✅ **Step 1024** - CSRF protection  
✅ **Step 1025** - Security testing  
✅ **Step 1026** - Secrets scanning  
✅ **Step 1027** - Rate limiting & DDoS protection  
✅ **Step 1028** - Dependency vulnerability scanning  
✅ **Step 1029** - Encryption at rest  
✅ **Step 1030** - Security monitoring  

➡️ [Part 97: CI/CD & Deployment](./part-97-cicd.md)
