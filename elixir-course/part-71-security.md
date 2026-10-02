# Part 71: Advanced Security (Steps 771-790)

## Step 771: OWASP Top 10 in Elixir

```
OWASP Top 10 and Elixir mitigations:

1. Broken Access Control
   → Plug-based authorization, resource ownership checks
   
2. Cryptographic Failures
   → Comeonin/Bcrypt, :crypto, secure random, HTTPS only
   
3. Injection (SQL, Command, etc.)
   → Ecto parameterized queries, avoid raw SQL, no shell exec
   
4. Insecure Design
   → Immutable data, explicit data flows, avoid shared mutable state
   
5. Security Misconfiguration
   → Runtime config, secrets in env vars, secure headers
   
6. Vulnerable Components
   → mix audit, mix hex.audit, dependency pinning
   
7. Auth & Session Failures
   → Guardian/Joken JWT, secure cookie flags, session fixation
   
8. Software Integrity Failures
   → mix.lock, checksums, signed releases
   
9. Logging & Monitoring Failures
   → Logger with PII scrubbing, telemetry, alerts
   
10. SSRF (Server-Side Request Forgery)
    → URL allowlist, block internal IPs, timeout limits
```

---

## Step 772: SQL Injection Prevention

```elixir
# WRONG: raw string interpolation (SQL injection)
def search_users_bad(name) do
  MyApp.Repo.query!("SELECT * FROM users WHERE name = '#{name}'")
end

# CORRECT: parameterized query
def search_users_good(name) do
  MyApp.Repo.query!("SELECT * FROM users WHERE name = $1", [name])
end

# CORRECT: Ecto (parameterized automatically)
def search_users_ecto(name) do
  import Ecto.Query
  from(u in MyApp.User, where: u.name == ^name)
  |> MyApp.Repo.all()
end

# CORRECT: Dynamic queries with Ecto
def filter_users(filters) do
  import Ecto.Query
  
  Enum.reduce(filters, from(u in MyApp.User), fn
    {:name, name}, query -> where(query, [u], u.name == ^name)
    {:email, email}, query -> where(query, [u], u.email == ^email)
    {:role, role}, query -> where(query, [u], u.role == ^role)
    _, query -> query
  end)
  |> MyApp.Repo.all()
end

# DANGER: dynamic ORDER BY (cannot parameterize column names)
def sort_users_safe(column, direction) do
  allowed_columns = ["name", "email", "inserted_at"]
  allowed_directions = ["asc", "desc"]

  unless column in allowed_columns and direction in allowed_directions do
    raise ArgumentError, "Invalid sort parameters"
  end

  MyApp.Repo.query!("SELECT * FROM users ORDER BY #{column} #{direction}")
end
```

---

## Step 773: Secure Authentication

```elixir
defmodule MyApp.Auth do
  alias MyApp.{Repo, Accounts.User}
  import Ecto.Query

  # Constant-time comparison to prevent timing attacks
  def authenticate(email, password) do
    user = Repo.get_by(User, email: String.downcase(email))

    # Always run verify even if user not found (prevent timing oracle)
    case Bcrypt.check_pass(user, password, hash_key: :password_hash) do
      {:ok, user} ->
        if user.confirmed_at do
          {:ok, user}
        else
          {:error, :email_not_confirmed}
        end
      {:error, _}  -> {:error, :invalid_credentials}
    end
  end

  def hash_password(password) do
    Bcrypt.hash_pwd_salt(password, log_rounds: 12)
  end

  # Secure token generation
  def generate_token(length \\ 32) do
    :crypto.strong_rand_bytes(length)
    |> Base.url_encode64(padding: false)
  end

  # HMAC-based token (for email links)
  def generate_signed_token(data, secret \\ token_secret()) do
    payload  = Jason.encode!(data)
    mac      = :crypto.mac(:hmac, :sha256, secret, payload)
    signature = Base.url_encode64(mac, padding: false)
    Base.url_encode64(payload, padding: false) <> "." <> signature
  end

  def verify_signed_token(token, secret \\ token_secret()) do
    with [payload_b64, sig_b64] <- String.split(token, "."),
         {:ok, payload_raw}     <- Base.url_decode64(payload_b64, padding: false),
         {:ok, sig_raw}         <- Base.url_decode64(sig_b64, padding: false) do
      expected = :crypto.mac(:hmac, :sha256, secret, payload_raw)
      if Plug.Crypto.secure_compare(sig_raw, expected) do
        {:ok, Jason.decode!(payload_raw)}
      else
        {:error, :invalid_signature}
      end
    else
      _ -> {:error, :invalid_token}
    end
  end

  defp token_secret, do: Application.fetch_env!(:my_app, :token_secret)
end
```

---

## Step 774: CSRF Protection

```elixir
# Phoenix has built-in CSRF via Plug.CSRFProtection
# Automatically added when using :browser pipeline

# config.exs
config :my_app, MyAppWeb.Endpoint,
  live_view: [signing_salt: "your_salt"]

# In templates, CSRF token is automatically included
# In custom forms:
<form action="/submit" method="post">
  <%= tag :input, type: "hidden",
    name: "_csrf_token",
    value: Phoenix.Controller.get_csrf_token() %>
  # ...
end

# For API endpoints requiring CSRF (e.g., JSON APIs)
defmodule MyAppWeb.Plugs.CheckCSRF do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    if safe_method?(conn.method) or valid_origin?(conn) do
      conn
    else
      conn
      |> send_resp(403, "CSRF protection")
      |> halt()
    end
  end

  defp safe_method?(method), do: method in ~w(GET HEAD OPTIONS)

  defp valid_origin?(conn) do
    origin = get_req_header(conn, "origin") |> List.first()
    host   = conn.host

    origin == nil or String.contains?(origin, host)
  end
end
```

---

## Step 775: Security Headers

```elixir
defmodule MyAppWeb.Plugs.SecurityHeaders do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    conn
    # Prevent XSS
    |> put_resp_header("x-content-type-options",    "nosniff")
    |> put_resp_header("x-xss-protection",          "1; mode=block")
    # Prevent clickjacking
    |> put_resp_header("x-frame-options",            "SAMEORIGIN")
    # HTTPS only
    |> put_resp_header("strict-transport-security",  "max-age=31536000; includeSubDomains")
    # Referrer policy
    |> put_resp_header("referrer-policy",            "strict-origin-when-cross-origin")
    # Permissions policy
    |> put_resp_header("permissions-policy",         "camera=(), microphone=(), geolocation=()")
    # Content Security Policy
    |> put_resp_header("content-security-policy",    csp())
  end

  defp csp do
    """
    default-src 'self';
    script-src 'self' 'nonce-#{generate_nonce()}';
    style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
    font-src 'self' https://fonts.gstatic.com;
    img-src 'self' data: https:;
    connect-src 'self' wss:;
    frame-ancestors 'none';
    base-uri 'self';
    form-action 'self'
    """
    |> String.replace("\n", " ")
    |> String.trim()
  end

  defp generate_nonce do
    :crypto.strong_rand_bytes(16)
    |> Base.encode64(padding: false)
  end
end

# Add to endpoint.ex
plug MyAppWeb.Plugs.SecurityHeaders
```

---

## Step 776: Input Validation & Sanitization

```elixir
defmodule MyApp.Sanitizer do
  # HTML sanitization (prevent XSS in stored content)
  # mix.exs: {:html_sanitize_ex, "~> 1.4"}

  @allowed_tags ~w(p br b i u em strong a img ul ol li blockquote code pre)
  @allowed_attrs %{
    "a"   => ["href", "title"],
    "img" => ["src", "alt", "width", "height"]
  }

  def sanitize_html(html) do
    HtmlSanitizeEx.basic_html(html)
  end

  def sanitize_markdown(md) do
    md
    |> Earmark.as_html!()
    |> sanitize_html()
  end

  def strip_html(html) do
    HtmlSanitizeEx.strip_tags(html)
  end

  # Safe URL validation
  def safe_url?(url) do
    case URI.parse(url) do
      %URI{scheme: scheme} when scheme in ["http", "https"] ->
        not internal_ip?(url)
      _ -> false
    end
  end

  defp internal_ip?(url) do
    host = URI.parse(url).host
    # Block RFC1918 addresses and localhost
    host in ["localhost", "127.0.0.1", "0.0.0.0"] or
    String.starts_with?(host, "192.168.") or
    String.starts_with?(host, "10.") or
    String.starts_with?(host, "172.16.")
  end
end

# Ecto changeset validation
def safe_changeset(post, attrs) do
  post
  |> cast(attrs, [:title, :content, :url])
  |> validate_required([:title, :content])
  |> validate_length(:title, max: 200)
  |> sanitize_content()
  |> validate_safe_url(:url)
end

defp sanitize_content(changeset) do
  case get_change(changeset, :content) do
    nil     -> changeset
    content -> put_change(changeset, :content, MyApp.Sanitizer.sanitize_html(content))
  end
end
```

---

## Step 777: Rate Limiting Brute Force

```elixir
defmodule MyApp.BruteForceProtection do
  @max_attempts 5
  @lockout_minutes 30

  def check_and_record(identifier) do
    key       = "brute_force:#{identifier}"
    attempts  = increment_attempts(key)
    
    cond do
      locked_out?(key)         -> {:error, :locked_out, lockout_remaining(key)}
      attempts >= @max_attempts ->
        lock_out(key)
        {:error, :locked_out, @lockout_minutes * 60}
      true                     ->
        {:ok, @max_attempts - attempts}
    end
  end

  def reset(identifier) do
    key = "brute_force:#{identifier}"
    Redix.command(:redix, ["DEL", key, "#{key}:locked"])
  end

  defp increment_attempts(key) do
    {:ok, count} = Redix.command(:redix, ["INCR", key])
    if count == 1 do
      Redix.command(:redix, ["EXPIRE", key, @lockout_minutes * 60])
    end
    count
  end

  defp locked_out?(key) do
    case Redix.command(:redix, ["GET", "#{key}:locked"]) do
      {:ok, nil} -> false
      {:ok, _}   -> true
    end
  end

  defp lock_out(key) do
    lock_key = "#{key}:locked"
    Redix.command(:redix, ["SET", lock_key, "1", "EX", @lockout_minutes * 60])
    Logger.warning("Account locked after #{@max_attempts} failed attempts: #{key}")
  end

  defp lockout_remaining(key) do
    case Redix.command(:redix, ["TTL", "#{key}:locked"]) do
      {:ok, ttl} when ttl > 0 -> ttl
      _                       -> 0
    end
  end
end
```

---

## Step 778: Audit Logging

```elixir
defmodule MyApp.AuditLog do
  use Ecto.Schema

  schema "audit_logs" do
    field :user_id,   :integer
    field :action,    :string
    field :resource,  :string
    field :resource_id, :string
    field :changes,   :map
    field :ip_address, :string
    field :user_agent, :string
    field :status,    :string  # success, failure
    timestamps(updated_at: false)
  end

  def log(user_id, action, resource, resource_id, opts \\ []) do
    attrs = %{
      user_id:    user_id,
      action:     to_string(action),
      resource:   to_string(resource),
      resource_id: to_string(resource_id),
      changes:    Keyword.get(opts, :changes, %{}),
      ip_address: Keyword.get(opts, :ip),
      user_agent: Keyword.get(opts, :user_agent),
      status:     Keyword.get(opts, :status, "success")
    }

    # Async logging so it doesn't block main request
    Task.start(fn ->
      MyApp.Repo.insert!(%__MODULE__{} |> Ecto.Changeset.change(attrs))
    end)
  end

  def for_user(user_id, opts \\ []) do
    import Ecto.Query

    from(a in __MODULE__,
      where: a.user_id == ^user_id,
      order_by: [desc: a.inserted_at],
      limit: ^Keyword.get(opts, :limit, 100)
    )
    |> MyApp.Repo.all()
  end
end

# Usage in controllers
def delete(conn, %{"id" => id}) do
  user = conn.assigns.current_user
  resource = get_resource!(id)

  case MyApp.Resources.delete(resource) do
    {:ok, _} ->
      MyApp.AuditLog.log(user.id, :delete, :resource, id,
        ip: conn.remote_ip |> Tuple.to_list() |> Enum.join("."),
        user_agent: get_req_header(conn, "user-agent") |> List.first()
      )
      json(conn, %{status: "deleted"})
    {:error, _} ->
      MyApp.AuditLog.log(user.id, :delete, :resource, id, status: "failure")
      send_resp(conn, 500, "Error")
  end
end
```

---

## Step 779: Secrets Management

```elixir
# Use runtime config for all secrets
# config/runtime.exs

import Config

# Database
config :my_app, MyApp.Repo,
  url: System.fetch_env!("DATABASE_URL")

# JWT signing key
config :my_app, :jwt_secret, System.fetch_env!("JWT_SECRET")

# External APIs
config :my_app, :stripe_key,      System.fetch_env!("STRIPE_SECRET_KEY")
config :my_app, :sendgrid_api_key, System.fetch_env!("SENDGRID_API_KEY")

# Encryption key (must be exactly 32 bytes for AES-256)
config :my_app, :encryption_key,
  System.fetch_env!("ENCRYPTION_KEY")
  |> Base.decode64!()

# For Vault integration
defmodule MyApp.Secrets do
  def fetch(path) do
    case vault_enabled?() do
      true  -> fetch_from_vault(path)
      false -> fetch_from_env(path)
    end
  end

  defp vault_enabled?, do: System.get_env("VAULT_ADDR") != nil

  defp fetch_from_vault(path) do
    vault_token = System.fetch_env!("VAULT_TOKEN")
    vault_addr  = System.fetch_env!("VAULT_ADDR")
    
    Req.get!("#{vault_addr}/v1/#{path}",
      headers: [{"X-Vault-Token", vault_token}]
    ).body["data"]["value"]
  end

  defp fetch_from_env(path) do
    env_key = path |> String.upcase() |> String.replace("/", "_")
    System.get_env(env_key)
  end
end
```

---

## Step 780: Encryption at Rest

```elixir
defmodule MyApp.Encryption do
  # Field-level encryption using AES-256-GCM

  @aad "MyApp.Encryption.v1"

  def encrypt(plaintext) when is_binary(plaintext) do
    key   = get_key()
    iv    = :crypto.strong_rand_bytes(12)  # 96-bit IV for GCM
    
    {ciphertext, tag} = :crypto.crypto_one_time_aead(
      :aes_256_gcm, key, iv, plaintext, @aad, true
    )

    # Store IV + tag + ciphertext together
    (iv <> tag <> ciphertext) |> Base.encode64()
  end

  def decrypt(encoded) do
    key     = get_key()
    data    = Base.decode64!(encoded)
    
    <<iv::binary-12, tag::binary-16, ciphertext::binary>> = data

    :crypto.crypto_one_time_aead(
      :aes_256_gcm, key, iv, ciphertext, @aad, tag, false
    )
  end

  defp get_key do
    Application.fetch_env!(:my_app, :encryption_key)
  end
end

# Ecto type for transparent encryption
defmodule MyApp.EncryptedField do
  use Ecto.Type

  def type, do: :string

  def cast(value) when is_binary(value), do: {:ok, value}
  def cast(_), do: :error

  def dump(value) when is_binary(value) do
    {:ok, MyApp.Encryption.encrypt(value)}
  end
  def dump(_), do: :error

  def load(encrypted) when is_binary(encrypted) do
    {:ok, MyApp.Encryption.decrypt(encrypted)}
  end
  def load(_), do: :error
end

# Usage in schema
schema "users" do
  field :ssn,   MyApp.EncryptedField
  field :phone, MyApp.EncryptedField
end
```

---

## สรุป Part 71

✅ **Step 771** - OWASP Top 10  
✅ **Step 772** - SQL injection prevention  
✅ **Step 773** - Secure authentication  
✅ **Step 774** - CSRF protection  
✅ **Step 775** - Security headers  
✅ **Step 776** - Input validation & sanitization  
✅ **Step 777** - Brute force protection  
✅ **Step 778** - Audit logging  
✅ **Step 779** - Secrets management  
✅ **Step 780** - Encryption at rest  

➡️ [Part 72: Observability & Monitoring](./part-72-observability.md)
