# Part 38: Security และ Hardening (Steps 411-430)

## Step 411: Security Checklist

```
OWASP Top 10 for Elixir/Phoenix:
1. Injection: Ecto parameterized queries prevent SQL injection
2. Broken Auth: use phx.gen.auth, strong secrets, bcrypt
3. Sensitive Data Exposure: HTTPS, encrypted fields, no logging secrets
4. XXE: not applicable (no XML parsing typical)
5. Broken Access Control: RBAC, policy checks
6. Security Misconfiguration: check all configs
7. XSS: Phoenix auto-escapes HEEx templates
8. Insecure Deserialization: never eval/decode untrusted data
9. Using Vulnerable Components: mix audit
10. Insufficient Logging: audit logs, telemetry
```

---

## Step 412: SQL Injection Prevention

```elixir
defmodule MyApp.Secure.Queries do
  import Ecto.Query
  
  # ✅ SAFE: Parameterized query
  def find_user(email) do
    from(u in User, where: u.email == ^email)
    |> Repo.one()
  end
  
  # ✅ SAFE: LIKE with parameterization
  def search_users(term) do
    from(u in User, where: ilike(u.name, ^"%#{term}%"))
    |> Repo.all()
  end
  
  # ❌ UNSAFE: String interpolation in query (never do this)
  def bad_search(name) do
    Repo.query!("SELECT * FROM users WHERE name = '#{name}'")
    # SQL injection possible!
  end
  
  # ✅ SAFE: Dynamic column sorting
  @sortable_columns [:name, :email, :inserted_at]
  
  def sorted_users(column) when column in @sortable_columns do
    from(u in User, order_by: [{:asc, field(u, ^column)}])
    |> Repo.all()
  end
  def sorted_users(_), do: {:error, :invalid_column}
end
```

---

## Step 413: XSS Prevention

```elixir
# Phoenix HEEx auto-escapes by default
# <%= @user_content %>         ← SAFE: auto-escaped
# <%= raw(@html_content) %>    ← UNSAFE: raw HTML injection
# <.user_content text={@text}/> ← SAFE: escapes in component

defmodule MyAppWeb.SafeHelpers do
  use Phoenix.HTML
  
  # Sanitize HTML for rich text
  def sanitize_html(html) when is_binary(html) do
    # Use HtmlSanitizeEx
    HtmlSanitizeEx.strip_tags(html)
  end
  
  # Allow only safe tags
  def sanitize_rich_text(html) do
    HtmlSanitizeEx.Scrubber.scrub(html, MyApp.HTMLScrubber)
  end
end

defmodule MyApp.HTMLScrubber do
  require HtmlSanitizeEx.Scrubber.Meta
  alias HtmlSanitizeEx.Scrubber.Meta
  
  # Allow safe tags only
  Meta.allow_tag_with_uri_attributes("a", ["href"], ["http", "https"])
  Meta.allow_tag_with_these_attributes("strong", [])
  Meta.allow_tag_with_these_attributes("em", [])
  Meta.allow_tag_with_these_attributes("p", [])
  Meta.allow_tag_with_these_attributes("ul", [])
  Meta.allow_tag_with_these_attributes("li", [])
  Meta.strip_everything_not_covered()
end

# Content Security Policy
defmodule MyAppWeb.SecurityHeaders do
  def add_security_headers(conn) do
    conn
    |> Plug.Conn.put_resp_header("x-frame-options", "DENY")
    |> Plug.Conn.put_resp_header("x-content-type-options", "nosniff")
    |> Plug.Conn.put_resp_header("x-xss-protection", "1; mode=block")
    |> Plug.Conn.put_resp_header("referrer-policy", "strict-origin-when-cross-origin")
    |> Plug.Conn.put_resp_header("content-security-policy",
        "default-src 'self'; script-src 'self' 'nonce-#{nonce()}'; style-src 'self'")
  end
  
  defp nonce do
    :crypto.strong_rand_bytes(16) |> Base.encode64()
  end
end
```

---

## Step 414: Password Security

```elixir
defmodule MyApp.Accounts.Password do
  @min_length 12
  
  def hash(plaintext) when byte_size(plaintext) >= @min_length do
    Bcrypt.hash_pwd_salt(plaintext, log_rounds: 12)
  end
  def hash(_), do: {:error, :too_short}
  
  def verify(plaintext, hash) do
    Bcrypt.verify_pass(plaintext, hash)
  end
  
  # Dummy verify to prevent timing attacks on non-existent users
  def dummy_verify do
    Bcrypt.no_user_verify()
  end
  
  def strong?(password) do
    with true <- byte_size(password) >= @min_length,
         true <- String.match?(password, ~r/[A-Z]/),
         true <- String.match?(password, ~r/[a-z]/),
         true <- String.match?(password, ~r/[0-9]/),
         true <- String.match?(password, ~r/[^A-Za-z0-9]/) do
      true
    else
      _ -> false
    end
  end
  
  def check_not_breached(password) do
    # Check against HaveIBeenPwned API using k-anonymity
    hash = :crypto.hash(:sha1, password) |> Base.encode16()
    prefix = String.slice(hash, 0, 5)
    suffix = String.slice(hash, 5, 35)
    
    case Req.get("https://api.pwnedpasswords.com/range/#{prefix}") do
      {:ok, %{body: body}} ->
        if String.contains?(body, suffix) do
          {:error, :password_breached}
        else
          :ok
        end
      _ -> :ok  # fail open if API unavailable
    end
  end
end
```

---

## Step 415: Encryption

```elixir
defmodule MyApp.Encryption do
  @aad "MyApp v1"
  
  def encrypt(plaintext) when is_binary(plaintext) do
    key  = get_key()
    iv   = :crypto.strong_rand_bytes(12)
    
    {ciphertext, tag} = :crypto.crypto_one_time_aead(
      :aes_256_gcm, key, iv, plaintext, @aad, true
    )
    
    iv <> tag <> ciphertext
    |> Base.encode64()
  end
  
  def decrypt(ciphertext_b64) do
    with {:ok, bin} <- Base.decode64(ciphertext_b64) do
      <<iv::binary-12, tag::binary-16, ciphertext::binary>> = bin
      key = get_key()
      
      case :crypto.crypto_one_time_aead(
        :aes_256_gcm, key, iv, ciphertext, @aad, tag, false
      ) do
        :error     -> {:error, :decryption_failed}
        plaintext  -> {:ok, plaintext}
      end
    end
  end
  
  # Field-level encryption for PII
  def encrypt_pii(value) when is_binary(value), do: encrypt(value)
  def encrypt_pii(nil), do: nil
  
  def decrypt_pii(nil), do: nil
  def decrypt_pii(value) do
    case decrypt(value) do
      {:ok, plain} -> plain
      _            -> nil
    end
  end
  
  defp get_key do
    System.fetch_env!("ENCRYPTION_KEY")
    |> Base.decode64!()
  end
end

# Custom Ecto type for encrypted fields
defmodule MyApp.EncryptedString do
  use Ecto.Type
  
  def type, do: :string
  
  def cast(value) when is_binary(value), do: {:ok, value}
  def cast(_), do: :error
  
  def load(encrypted) when is_binary(encrypted) do
    MyApp.Encryption.decrypt_pii(encrypted)
    |> then(&{:ok, &1})
  end
  def load(nil), do: {:ok, nil}
  
  def dump(value) when is_binary(value) do
    {:ok, MyApp.Encryption.encrypt_pii(value)}
  end
  def dump(nil), do: {:ok, nil}
  def dump(_), do: :error
end

# Use in schema
defmodule MyApp.User do
  use Ecto.Schema
  
  schema "users" do
    field :email,   :string
    field :phone,   MyApp.EncryptedString  # stored encrypted
    field :ssn,     MyApp.EncryptedString  # stored encrypted
  end
end
```

---

## Step 416: Rate Limiting และ Brute Force Protection

```elixir
defmodule MyApp.BruteForceProtection do
  @max_attempts 5
  @lockout_duration 900  # 15 minutes
  
  def record_attempt(identifier) do
    key = "login_attempts:#{identifier}"
    
    {:ok, count} = Redix.pipeline(:redis, [
      ["INCR", key],
      ["EXPIRE", key, @lockout_duration]
    ])
    
    count = List.first(count)
    
    if count >= @max_attempts do
      {:error, :locked_out, remaining_lockout(key)}
    else
      {:ok, @max_attempts - count}
    end
  end
  
  def reset(identifier) do
    Redix.command(:redis, ["DEL", "login_attempts:#{identifier}"])
  end
  
  def locked_out?(identifier) do
    case Redix.command(:redis, ["GET", "login_attempts:#{identifier}"]) do
      {:ok, count} when not is_nil(count) ->
        String.to_integer(count) >= @max_attempts
      _ -> false
    end
  end
  
  defp remaining_lockout(key) do
    case Redix.command(:redis, ["TTL", key]) do
      {:ok, ttl} when ttl > 0 -> ttl
      _ -> @lockout_duration
    end
  end
end

defmodule MyApp.AccountsController do
  def login(conn, %{"email" => email, "password" => password}) do
    identifier = "#{email}:#{conn.remote_ip |> :inet.ntoa()}"
    
    if MyApp.BruteForceProtection.locked_out?(identifier) do
      conn
      |> put_status(:too_many_requests)
      |> json(%{error: "Account temporarily locked"})
    else
      case authenticate(email, password) do
        {:ok, user} ->
          MyApp.BruteForceProtection.reset(identifier)
          login_success(conn, user)
        
        {:error, _} ->
          MyApp.BruteForceProtection.record_attempt(identifier)
          conn |> put_status(:unauthorized) |> json(%{error: "Invalid credentials"})
      end
    end
  end
end
```

---

## Step 417: CSRF Protection

```elixir
# Phoenix includes CSRF protection by default
# router.ex
pipeline :browser do
  plug :accepts, ["html"]
  plug :fetch_session
  plug :fetch_live_flash
  plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
  plug :protect_from_forgery      # ← CSRF protection
  plug :put_secure_browser_headers
end

# In forms: automatically includes CSRF token
# <.form for={@form} action={~p"/users/login"}>
#   <%= hidden_input @form, :_csrf_token %>  ← auto-added by Phoenix

# For AJAX requests:
# Add header: X-CSRF-TOKEN with value from meta tag
# <meta name="csrf-token" content={get_csrf_token()}>

# Custom CSRF check
defmodule MyAppWeb.Plugs.CSRFCheck do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    if conn.method in ["GET", "HEAD", "OPTIONS"] do
      conn
    else
      case get_csrf_token(conn) do
        nil ->
          conn |> put_status(:forbidden) |> halt()
        _token ->
          conn
      end
    end
  end
  
  defp get_csrf_token(conn) do
    get_req_header(conn, "x-csrf-token") |> List.first()
  end
end
```

---

## Step 418: Security Headers

```elixir
defmodule MyAppWeb.Plugs.SecurityHeaders do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    conn
    |> put_resp_header("strict-transport-security", "max-age=31536000; includeSubDomains")
    |> put_resp_header("x-frame-options", "SAMEORIGIN")
    |> put_resp_header("x-content-type-options", "nosniff")
    |> put_resp_header("x-xss-protection", "1; mode=block")
    |> put_resp_header("referrer-policy", "strict-origin-when-cross-origin")
    |> put_resp_header("permissions-policy",
        "camera=(), microphone=(), geolocation=()")
    |> put_csp()
  end
  
  defp put_csp(conn) do
    nonce = :crypto.strong_rand_bytes(16) |> Base.encode64(padding: false)
    
    conn
    |> assign(:csp_nonce, nonce)
    |> put_resp_header("content-security-policy", csp_policy(nonce))
  end
  
  defp csp_policy(nonce) do
    [
      "default-src 'self'",
      "script-src 'self' 'nonce-#{nonce}'",
      "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
      "font-src 'self' https://fonts.gstatic.com",
      "img-src 'self' data: https:",
      "connect-src 'self' wss:",
      "frame-ancestors 'none'"
    ]
    |> Enum.join("; ")
  end
end
```

---

## Step 419: Secrets Scanning

```bash
# mix.exs
{:sobelow, "~> 0.13", only: [:dev, :test], runtime: false}

# Check for security issues
mix sobelow --config

# Create .sobelow-conf
# %{
#   skip: ["XSS.Raw"],  # if you intentionally use raw HTML
#   exit: "high"        # exit with error on high severity
# }

# mix audit - check for vulnerable dependencies
mix deps.audit

# Common sobelow checks:
# - SQL injection via Ecto
# - XSS via raw HTML
# - Directory traversal
# - Missing CSRF
# - Weak encryption
# - Exposed secrets in config
```

```elixir
# Never log sensitive data
defmodule MyApp.SafeLogger do
  require Logger
  
  @sensitive_keys [:password, :token, :secret, :credit_card, :ssn, :cvv]
  
  def log_request(conn) do
    params = conn.params
      |> mask_sensitive([:password, :token, :credit_card])
    
    Logger.info("Request",
      method:  conn.method,
      path:    conn.request_path,
      user_id: get_in(conn.assigns, [:current_user, :id]),
      params:  inspect(params)
    )
  end
  
  defp mask_sensitive(map, keys) when is_map(map) do
    Enum.reduce(keys, map, fn key, acc ->
      if Map.has_key?(acc, to_string(key)) do
        Map.put(acc, to_string(key), "[REDACTED]")
      else
        acc
      end
    end)
  end
end
```

---

## Step 420: Penetration Testing

```elixir
# Security testing with ExUnit
defmodule MyAppWeb.SecurityTest do
  use MyAppWeb.ConnCase
  
  describe "authentication" do
    test "rejects unauthenticated requests to protected routes" do
      conn = get(build_conn(), ~p"/api/v1/users")
      assert json_response(conn, 401)
    end
    
    test "prevents brute force with rate limiting" do
      for _ <- 1..10 do
        post(build_conn(), ~p"/api/v1/auth/login", %{
          email: "test@test.com",
          password: "wrong"
        })
      end
      
      conn = post(build_conn(), ~p"/api/v1/auth/login", %{
        email: "test@test.com",
        password: "wrong"
      })
      
      assert json_response(conn, 429)
    end
    
    test "CSRF protection on state-changing endpoints" do
      conn = build_conn()
        |> delete_req_header("x-csrf-token")
        |> post(~p"/users", %{name: "Hacker"})
      
      assert conn.status in [403, 422]
    end
  end
  
  describe "authorization" do
    test "user cannot access other user's data" do
      user1 = user_fixture()
      user2 = user_fixture()
      
      conn = user1 |> log_in_user() |> get(~p"/api/v1/users/#{user2.id}/private")
      
      assert json_response(conn, 403)
    end
    
    test "SQL injection in search" do
      malicious_input = "'; DROP TABLE users; --"
      
      conn = get(build_conn(), ~p"/api/v1/users?search=#{malicious_input}")
      
      # Should return empty results, not error
      assert json_response(conn, 200)
      assert MyApp.Repo.aggregate(MyApp.Accounts.User, :count, :id) > 0
    end
  end
end
```

---

## สรุป Part 38

✅ **Step 411** - Security checklist  
✅ **Step 412** - SQL injection prevention  
✅ **Step 413** - XSS prevention  
✅ **Step 414** - Password security  
✅ **Step 415** - Encryption (AES-256-GCM)  
✅ **Step 416** - Brute force protection  
✅ **Step 417** - CSRF protection  
✅ **Step 418** - Security headers  
✅ **Step 419** - Secrets scanning  
✅ **Step 420** - Security testing  

➡️ [Part 39: Observability และ Monitoring](./part-39-observability.md)
