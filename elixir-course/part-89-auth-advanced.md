# Part 89: Authentication & Authorization (Steps 951-960)

## Step 951: JWT Authentication

```elixir
# mix.exs: {:jose, "~> 1.11"}

defmodule MyApp.Auth.JWT do
  @secret   System.get_env("JWT_SECRET") || "dev-secret"
  @issuer   "myapp"
  @access_ttl  15 * 60        # 15 minutes
  @refresh_ttl 30 * 24 * 3600 # 30 days

  def generate_tokens(user) do
    access  = sign(access_claims(user), @access_ttl)
    refresh = sign(refresh_claims(user), @refresh_ttl)
    {:ok, %{access_token: access, refresh_token: refresh}}
  end

  def verify(token) do
    case JOSE.JWT.verify_strict(jwk(), ["HS256"], token) do
      {true, %JOSE.JWT{fields: claims}, _} ->
        if DateTime.utc_now() |> DateTime.to_unix() < claims["exp"] do
          {:ok, claims}
        else
          {:error, :expired}
        end
      _ -> {:error, :invalid}
    end
  end

  def refresh(refresh_token) do
    with {:ok, %{"sub" => user_id, "type" => "refresh"}} <- verify(refresh_token),
         user <- MyApp.Accounts.get_user!(user_id) do
      generate_tokens(user)
    end
  end

  defp sign(claims, ttl) do
    exp = DateTime.utc_now() |> DateTime.to_unix() |> Kernel.+(ttl)
    payload = Map.merge(claims, %{"exp" => exp, "iss" => @issuer})
    {_, token} = JOSE.JWT.sign(jwk(), %{"alg" => "HS256"}, payload) |> JOSE.JWS.compact()
    token
  end

  defp jwk, do: JOSE.JWK.from_oct(@secret)

  defp access_claims(user) do
    %{"sub" => to_string(user.id), "type" => "access",
      "email" => user.email, "roles" => user.roles}
  end

  defp refresh_claims(user) do
    %{"sub" => to_string(user.id), "type" => "refresh"}
  end
end
```

---

## Step 952: OAuth2 Social Login

```elixir
# mix.exs: {:assent, "~> 0.2"}

defmodule MyApp.Auth.OAuth do
  # Google OAuth2

  def google_auth_url(redirect_uri, state) do
    Assent.Strategy.Google.authorize_url(%{
      client_id:     System.get_env("GOOGLE_CLIENT_ID"),
      redirect_uri:  redirect_uri,
      state:         state,
      scope:         "email profile"
    })
  end

  def google_callback(code, redirect_uri) do
    with {:ok, %{user: profile}} <-
           Assent.Strategy.Google.callback(%{
             client_id:     System.get_env("GOOGLE_CLIENT_ID"),
             client_secret: System.get_env("GOOGLE_CLIENT_SECRET"),
             redirect_uri:  redirect_uri,
             code:          code
           }) do
      find_or_create_user(profile, :google)
    end
  end

  defp find_or_create_user(%{"sub" => uid, "email" => email, "name" => name}, provider) do
    case MyApp.Accounts.find_by_oauth(provider, uid) do
      nil ->
        MyApp.Accounts.create_oauth_user(%{
          email:         email,
          name:          name,
          oauth_provider: provider,
          oauth_uid:     uid
        })
      user ->
        {:ok, user}
    end
  end
end

defmodule MyAppWeb.Auth.OAuthController do
  use MyAppWeb, :controller

  def request(conn, %{"provider" => "google"}) do
    state = Base.url_encode64(:crypto.strong_rand_bytes(16))
    redirect_uri = url(~p"/auth/google/callback")

    {:ok, %{url: auth_url}} = MyApp.Auth.OAuth.google_auth_url(redirect_uri, state)

    conn
    |> put_session(:oauth_state, state)
    |> redirect(external: auth_url)
  end

  def callback(conn, %{"provider" => "google", "code" => code, "state" => state}) do
    if get_session(conn, :oauth_state) != state do
      conn |> put_flash(:error, "Invalid state") |> redirect(to: ~p"/login")
    else
      redirect_uri = url(~p"/auth/google/callback")
      case MyApp.Auth.OAuth.google_callback(code, redirect_uri) do
        {:ok, user} ->
          {:ok, tokens} = MyApp.Auth.JWT.generate_tokens(user)
          conn
          |> put_session(:user_id, user.id)
          |> put_resp_cookie("refresh_token", tokens.refresh_token, http_only: true, secure: true)
          |> redirect(to: ~p"/dashboard")
        {:error, _} ->
          conn |> put_flash(:error, "Authentication failed") |> redirect(to: ~p"/login")
      end
    end
  end
end
```

---

## Step 953: Role-Based Access Control

```elixir
defmodule MyApp.Auth.RBAC do
  # Role hierarchy
  @role_hierarchy %{
    :superadmin => [:admin, :moderator, :user],
    :admin      => [:moderator, :user],
    :moderator  => [:user],
    :user       => []
  }

  # Permissions per role
  @permissions %{
    :superadmin => [:all],
    :admin      => [:manage_users, :manage_content, :view_reports, :manage_settings],
    :moderator  => [:manage_content, :view_reports],
    :user       => [:view_content, :create_content, :edit_own_content]
  }

  def has_permission?(user, permission) do
    user_roles = [user.role | Map.get(@role_hierarchy, user.role, [])]
    
    Enum.any?(user_roles, fn role ->
      perms = Map.get(@permissions, role, [])
      :all in perms or permission in perms
    end)
  end

  def has_role?(user, required_role) do
    user.role == required_role or
      required_role in Map.get(@role_hierarchy, user.role, [])
  end
end

# Policy module
defmodule MyApp.Policies.PostPolicy do
  def can?(%{role: :admin}, _action, _resource), do: true
  def can?(user, :show,   %{published: true}),   do: true
  def can?(user, :edit,   %{user_id: uid}),       do: user.id == uid
  def can?(user, :delete, %{user_id: uid}),       do: user.id == uid
  def can?(_user, _action, _resource),            do: false
end

# Plug
defmodule MyAppWeb.Plugs.Authorize do
  import Plug.Conn
  import Phoenix.Controller

  def init(opts), do: opts

  def call(conn, permission: permission) do
    user = conn.assigns.current_user
    
    if MyApp.Auth.RBAC.has_permission?(user, permission) do
      conn
    else
      conn |> put_status(403) |> json(%{error: "forbidden"}) |> halt()
    end
  end
end
```

---

## Step 954: Two-Factor Authentication

```elixir
# mix.exs: {:nimble_totp, "~> 1.0"}

defmodule MyApp.Auth.TOTP do
  def setup(user) do
    secret = NimbleTOTP.secret()
    uri    = NimbleTOTP.otpauth_uri("MyApp:#{user.email}", secret, issuer: "MyApp")
    
    {:ok, %{secret: Base.encode32(secret), uri: uri}}
  end

  def verify(user, code) do
    secret = Base.decode32!(user.totp_secret)
    NimbleTOTP.valid?(secret, code, since: user.last_otp_at)
  end

  def enable(user, code) do
    if verify(user, code) do
      MyApp.Accounts.update_user(user, %{
        totp_enabled: true,
        totp_secret:  user.pending_totp_secret,
        last_otp_at:  DateTime.utc_now()
      })
    else
      {:error, :invalid_code}
    end
  end

  def generate_backup_codes(user) do
    codes = for _ <- 1..10, do: :crypto.strong_rand_bytes(5) |> Base.encode16(case: :lower)
    hashed = Enum.map(codes, &Bcrypt.hash_pwd_salt/1)
    
    MyApp.Accounts.update_user(user, %{backup_codes: hashed})
    {:ok, codes}  # Show plain codes once
  end

  def use_backup_code(user, code) do
    matching = Enum.find_index(user.backup_codes, &Bcrypt.verify_pass(code, &1))
    
    if matching do
      remaining = List.delete_at(user.backup_codes, matching)
      MyApp.Accounts.update_user(user, %{backup_codes: remaining})
      :ok
    else
      {:error, :invalid_code}
    end
  end
end
```

---

## Step 955: Session Management

```elixir
defmodule MyApp.Auth.Sessions do
  use Ecto.Schema
  import Ecto.Query

  schema "user_sessions" do
    field :user_id,     :integer
    field :token,       :string
    field :ip_address,  :string
    field :user_agent,  :string
    field :last_active, :utc_datetime
    field :expires_at,  :utc_datetime

    timestamps()
  end

  def create(user, conn) do
    token = Base.url_encode64(:crypto.strong_rand_bytes(32))
    
    %__MODULE__{
      user_id:    user.id,
      token:      Bcrypt.hash_pwd_salt(token),
      ip_address: to_string(:inet.ntoa(conn.remote_ip)),
      user_agent: get_req_header(conn, "user-agent") |> List.first(),
      last_active: DateTime.utc_now(),
      expires_at: DateTime.add(DateTime.utc_now(), 30 * 86400)
    }
    |> MyApp.Repo.insert!()
    
    {:ok, token}
  end

  def verify(token) do
    sessions = from s in __MODULE__,
      where: s.expires_at > ^DateTime.utc_now(),
      preload: [:user]
    
    # Compare token hash (slow, but secure)
    sessions
    |> MyApp.Repo.all()
    |> Enum.find(fn s -> Bcrypt.verify_pass(token, s.token) end)
    |> case do
      nil     -> {:error, :invalid_session}
      session ->
        update_last_active(session)
        {:ok, session.user}
    end
  end

  def list_active(user_id) do
    from s in __MODULE__,
      where: s.user_id == ^user_id and s.expires_at > ^DateTime.utc_now(),
      order_by: [desc: s.last_active]
    |> MyApp.Repo.all()
  end

  def revoke(session_id, user_id) do
    from(s in __MODULE__, where: s.id == ^session_id and s.user_id == ^user_id)
    |> MyApp.Repo.delete_all()
  end

  defp update_last_active(session) do
    # Update at most every minute to reduce writes
    if DateTime.diff(DateTime.utc_now(), session.last_active) > 60 do
      MyApp.Repo.update_all(
        from(s in __MODULE__, where: s.id == ^session.id),
        set: [last_active: DateTime.utc_now()]
      )
    end
  end
end
```

---

## Step 956: Password Security

```elixir
defmodule MyApp.Auth.Password do
  @min_length 8

  def hash(password) do
    Bcrypt.hash_pwd_salt(password, rounds: 12)
  end

  def verify(password, hash) do
    Bcrypt.verify_pass(password, hash)
  end

  def validate(password) do
    errors = []
    errors = if String.length(password) < @min_length,
      do: ["must be at least #{@min_length} characters" | errors], else: errors
    errors = if !Regex.match?(~r/[A-Z]/, password),
      do: ["must contain uppercase letter" | errors], else: errors
    errors = if !Regex.match?(~r/[0-9]/, password),
      do: ["must contain a number" | errors], else: errors
    errors = if !Regex.match?(~r/[^a-zA-Z0-9]/, password),
      do: ["must contain special character" | errors], else: errors
    
    if errors == [], do: :ok, else: {:error, errors}
  end

  def check_pwned(password) do
    # HaveIBeenPwned API
    hash   = :crypto.hash(:sha1, password) |> Base.encode16()
    prefix = String.slice(hash, 0, 5)
    suffix = String.slice(hash, 5, -1)

    case Req.get("https://api.pwnedpasswords.com/range/#{prefix}") do
      {:ok, %{status: 200, body: body}} ->
        found = body
          |> String.split("\r\n")
          |> Enum.any?(fn line ->
            [h | _] = String.split(line, ":")
            String.downcase(h) == String.downcase(suffix)
          end)
        if found, do: {:error, :pwned}, else: :ok
      _ -> :ok  # Fail open if API unavailable
    end
  end

  def generate_reset_token do
    Base.url_encode64(:crypto.strong_rand_bytes(32))
  end
end
```

---

## Step 957: API Key Management

```elixir
defmodule MyApp.APIKeys do
  use Ecto.Schema
  import Ecto.Query

  schema "api_keys" do
    field :user_id,    :integer
    field :name,       :string
    field :key_hash,   :string
    field :prefix,     :string
    field :scopes,     {:array, :string}, default: []
    field :last_used,  :utc_datetime
    field :expires_at, :utc_datetime
    field :revoked,    :boolean, default: false

    timestamps()
  end

  def generate(user_id, name, opts \\ []) do
    raw_key = "mak_#{Base.url_encode64(:crypto.strong_rand_bytes(32))}"
    prefix  = String.slice(raw_key, 0, 10)
    
    %__MODULE__{
      user_id:   user_id,
      name:      name,
      key_hash:  :crypto.hash(:sha256, raw_key) |> Base.encode16(case: :lower),
      prefix:    prefix,
      scopes:    Keyword.get(opts, :scopes, ["read"]),
      expires_at: Keyword.get(opts, :expires_at)
    }
    |> MyApp.Repo.insert!()
    
    {:ok, raw_key}  # Return only once; hash is stored
  end

  def verify(raw_key) do
    hash = :crypto.hash(:sha256, raw_key) |> Base.encode16(case: :lower)
    
    from(k in __MODULE__,
      where: k.key_hash == ^hash and k.revoked == false,
      where: is_nil(k.expires_at) or k.expires_at > ^DateTime.utc_now()
    )
    |> MyApp.Repo.one()
    |> case do
      nil -> :error
      key -> {:ok, key}
    end
  end

  def record_usage(%__MODULE__{} = key) do
    MyApp.Repo.update_all(
      from(k in __MODULE__, where: k.id == ^key.id),
      set: [last_used: DateTime.utc_now()]
    )
  end

  def revoke(key_id, user_id) do
    from(k in __MODULE__, where: k.id == ^key_id and k.user_id == ^user_id)
    |> MyApp.Repo.update_all(set: [revoked: true])
  end
end
```

---

## Step 958: Attribute-Based Access Control

```elixir
defmodule MyApp.Auth.ABAC do
  # Attribute-Based Access Control

  def authorize(subject, action, resource, context \\ %{}) do
    policies = applicable_policies(subject, action, resource)
    
    Enum.reduce_while(policies, {:deny, "no policy matched"}, fn policy, _acc ->
      case evaluate_policy(policy, subject, action, resource, context) do
        {:permit, reason} -> {:halt, {:permit, reason}}
        {:deny, reason}   -> {:cont, {:deny, reason}}
      end
    end)
  end

  defp applicable_policies(subject, action, resource) do
    MyApp.Policies.all()
    |> Enum.filter(fn policy ->
      policy.targets?(subject, action, resource)
    end)
  end

  defp evaluate_policy(policy, subject, action, resource, context) do
    conditions_met = Enum.all?(policy.conditions, fn condition ->
      evaluate_condition(condition, subject, action, resource, context)
    end)
    
    if conditions_met, do: {:permit, policy.name}, else: {:deny, "conditions not met"}
  end

  defp evaluate_condition({:attribute, path, op, value}, subject, _action, resource, _ctx) do
    actual = get_in(subject, path) || get_in(resource, path)
    apply_operator(op, actual, value)
  end

  defp evaluate_condition({:time, :business_hours}, _subj, _action, _res, _ctx) do
    now = DateTime.utc_now()
    now.hour >= 9 and now.hour < 18
  end

  defp apply_operator(:eq, a, b),  do: a == b
  defp apply_operator(:in, a, b),  do: a in b
  defp apply_operator(:gte, a, b), do: a >= b
  defp apply_operator(:lte, a, b), do: a <= b
end
```

---

## Step 959: Passwordless Authentication

```elixir
defmodule MyApp.Auth.Passwordless do
  @token_ttl 600  # 10 minutes

  def send_magic_link(email) do
    case MyApp.Accounts.get_by_email(email) do
      nil  ->
        # Prevent email enumeration
        :ok
      user ->
        token = Base.url_encode64(:crypto.strong_rand_bytes(32))
        
        Redix.command(:redix, ["SETEX",
          "magic_link:#{token}",
          @token_ttl,
          to_string(user.id)])
        
        link = MyAppWeb.Endpoint.url() <> "/auth/magic?token=#{token}"
        MyApp.Emails.magic_link(user, link) |> MyApp.Mailer.deliver_later()
        :ok
    end
  end

  def verify_token(token) do
    case Redix.command(:redix, ["GETDEL", "magic_link:#{token}"]) do
      {:ok, nil}     -> {:error, :invalid_token}
      {:ok, user_id} ->
        user = MyApp.Accounts.get_user!(String.to_integer(user_id))
        {:ok, user}
    end
  end

  # Passkey / WebAuthn (simplified)
  def register_passkey(user, credential) do
    MyApp.Repo.insert!(%UserPasskey{
      user_id:     user.id,
      credential_id: credential.id,
      public_key:  credential.public_key,
      device_name: credential.device_name
    })
  end

  def verify_passkey(credential_id, response) do
    passkey = MyApp.Repo.get_by!(UserPasskey, credential_id: credential_id)
    # WebAuthn verification
    Wax.authenticate(response.client_data_json, response.authenticator_data,
      response.signature, passkey.public_key)
    |> case do
      {:ok, _} ->
        user = MyApp.Accounts.get_user!(passkey.user_id)
        {:ok, user}
      {:error, reason} -> {:error, reason}
    end
  end
end
```

---

## Step 960: Security Headers

```elixir
defmodule MyAppWeb.Plugs.SecurityHeaders do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    conn
    |> put_resp_header("x-content-type-options",    "nosniff")
    |> put_resp_header("x-frame-options",           "DENY")
    |> put_resp_header("x-xss-protection",          "1; mode=block")
    |> put_resp_header("referrer-policy",           "strict-origin-when-cross-origin")
    |> put_resp_header("permissions-policy",        "camera=(), microphone=(), geolocation=()")
    |> put_resp_header("strict-transport-security", "max-age=31536000; includeSubDomains")
    |> put_resp_header("content-security-policy",   csp())
  end

  defp csp do
    [
      "default-src 'self'",
      "script-src 'self' 'nonce-#{generate_nonce()}'",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self' wss:",
      "frame-ancestors 'none'",
      "base-uri 'self'"
    ] |> Enum.join("; ")
  end

  defp generate_nonce, do: Base.encode64(:crypto.strong_rand_bytes(16))
end

# Rate limiting with IP reputation
defmodule MyApp.Security.IPReputation do
  def check(ip) do
    cond do
      in_blocklist?(ip)    -> {:block, "IP blocked"}
      in_allowlist?(ip)    -> {:allow, "IP allowlisted"}
      rate_limited?(ip)    -> {:block, "Rate limited"}
      abuse_detected?(ip)  -> {:challenge, "Suspicious activity"}
      true                 -> :allow
    end
  end

  defp in_blocklist?(ip) do
    case Redix.command(:redix, ["SISMEMBER", "ip_blocklist", ip]) do
      {:ok, 1} -> true
      _        -> false
    end
  end

  defp in_allowlist?(ip) do
    case Redix.command(:redix, ["SISMEMBER", "ip_allowlist", ip]) do
      {:ok, 1} -> true
      _        -> false
    end
  end

  defp rate_limited?(ip) do
    {:ok, count} = Redix.command(:redix, ["INCR", "rate:#{ip}"])
    if count == 1, do: Redix.command(:redix, ["EXPIRE", "rate:#{ip}", 60])
    count > 100
  end

  defp abuse_detected?(_ip), do: false
end
```

---

## สรุป Part 89

✅ **Step 951** - JWT authentication  
✅ **Step 952** - OAuth2 social login  
✅ **Step 953** - Role-based access control  
✅ **Step 954** - Two-factor authentication  
✅ **Step 955** - Session management  
✅ **Step 956** - Password security  
✅ **Step 957** - API key management  
✅ **Step 958** - Attribute-based access control  
✅ **Step 959** - Passwordless authentication  
✅ **Step 960** - Security headers  

➡️ [Part 90: Search & Discovery](./part-90-search.md)
