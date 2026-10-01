# Part 25: Authentication และ Authorization (Steps 271-290)

## Step 271: Session-based Authentication

```elixir
# mix phx.gen.auth Accounts User users
# สร้าง full auth system อัตโนมัติ

# lib/my_app/accounts/user.ex
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "users" do
    field :email,            :string
    field :password,         :string, virtual: true, redact: true
    field :hashed_password,  :string, redact: true
    field :confirmed_at,     :naive_datetime
    
    has_many :tokens, MyApp.Accounts.UserToken
    
    timestamps()
  end
  
  def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :password])
    |> validate_email()
    |> validate_password()
  end
  
  defp validate_email(changeset) do
    changeset
    |> validate_required([:email])
    |> validate_format(:email, ~r/^[^\s]+@[^\s]+$/)
    |> validate_length(:email, max: 160)
    |> unsafe_validate_unique(:email, MyApp.Repo)
    |> unique_constraint(:email)
  end
  
  defp validate_password(changeset) do
    changeset
    |> validate_required([:password])
    |> validate_length(:password, min: 12, max: 72)
    |> validate_format(:password, ~r/[a-z]/, message: "at least one lower case character")
    |> validate_format(:password, ~r/[A-Z]/, message: "at least one upper case character")
    |> validate_format(:password, ~r/[!?@#$%^&*_0-9]/, message: "at least one digit or special character")
    |> hash_password()
  end
  
  defp hash_password(changeset) do
    password = get_change(changeset, :password)
    
    if password && changeset.valid? do
      changeset
      |> validate_length(:password, max: 72, count: :bytes)
      |> put_change(:hashed_password, Bcrypt.hash_pwd_salt(password))
      |> delete_change(:password)
    else
      changeset
    end
  end
  
  def valid_password?(%User{hashed_password: hashed}, password) when is_binary(hashed) do
    Bcrypt.verify_pass(password, hashed)
  end
  def valid_password?(_, _) do
    Bcrypt.no_user_verify()
    false
  end
end
```

---

## Step 272: JWT Authentication

```elixir
# mix.exs
{:joken, "~> 2.6"}

defmodule MyApp.Auth.JWT do
  use Joken.Config
  
  @impl true
  def token_config do
    default_claims(
      iss: "my_app",
      aud: "my_app_api",
      default_exp: 3600  # 1 hour
    )
  end
  
  def generate_token(user) do
    extra_claims = %{
      "user_id"  => user.id,
      "email"    => user.email,
      "role"     => user.role
    }
    
    generate_and_sign!(extra_claims)
  end
  
  def verify_token(token) do
    case verify_and_validate(token) do
      {:ok, claims} -> {:ok, claims}
      {:error, reason} -> {:error, reason}
    end
  end
end

# Auth Controller
defmodule MyAppWeb.API.AuthController do
  use MyAppWeb, :controller
  
  alias MyApp.Accounts
  alias MyApp.Auth.JWT
  
  def login(conn, %{"email" => email, "password" => password}) do
    case Accounts.authenticate_user(email, password) do
      {:ok, user} ->
        token = JWT.generate_token(user)
        refresh_token = generate_refresh_token(user)
        
        conn
        |> put_status(:ok)
        |> json(%{
             access_token:  token,
             refresh_token: refresh_token,
             expires_in:    3600,
             token_type:    "Bearer"
           })
      
      {:error, :invalid_credentials} ->
        conn
        |> put_status(:unauthorized)
        |> json(%{error: "Invalid credentials"})
    end
  end
  
  def refresh(conn, %{"refresh_token" => refresh_token}) do
    case verify_refresh_token(refresh_token) do
      {:ok, user} ->
        new_token = JWT.generate_token(user)
        json(conn, %{access_token: new_token, expires_in: 3600})
      
      {:error, _} ->
        conn
        |> put_status(:unauthorized)
        |> json(%{error: "Invalid refresh token"})
    end
  end
  
  defp generate_refresh_token(user) do
    token = :crypto.strong_rand_bytes(32) |> Base.url_encode64(padding: false)
    # Store in DB with expiry
    Accounts.store_refresh_token(user.id, token)
    token
  end
  
  defp verify_refresh_token(token) do
    case Accounts.get_user_by_refresh_token(token) do
      nil  -> {:error, :invalid}
      user -> {:ok, user}
    end
  end
end

# Auth Plug
defmodule MyAppWeb.Plugs.VerifyJWT do
  import Plug.Conn
  alias MyApp.Auth.JWT
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    case get_token_from_header(conn) do
      {:ok, token} ->
        case JWT.verify_token(token) do
          {:ok, claims} ->
            assign(conn, :current_user_id, claims["user_id"])
          {:error, _} ->
            unauthorized(conn)
        end
      {:error, _} ->
        unauthorized(conn)
    end
  end
  
  defp get_token_from_header(conn) do
    case get_req_header(conn, "authorization") do
      ["Bearer " <> token] -> {:ok, token}
      _                    -> {:error, :missing}
    end
  end
  
  defp unauthorized(conn) do
    conn
    |> put_status(:unauthorized)
    |> Phoenix.Controller.json(%{error: "Unauthorized"})
    |> halt()
  end
end
```

---

## Step 273: Role-Based Access Control (RBAC)

```elixir
defmodule MyApp.Auth.Policy do
  @moduledoc "Authorization policies"
  
  alias MyApp.Accounts.User
  alias MyApp.Blog.Post
  
  # Define permissions
  @permissions %{
    admin: [:read_all, :write_all, :delete_all, :manage_users],
    editor: [:read_all, :write_posts, :publish_posts, :moderate_comments],
    user: [:read_published, :write_own_posts, :write_comments]
  }
  
  def can?(%User{role: role}, action) do
    permissions = Map.get(@permissions, String.to_atom(role), [])
    action in permissions
  end
  
  def can?(%User{} = user, :update, %Post{} = post) do
    can?(user, :write_all) || (can?(user, :write_own_posts) && post.user_id == user.id)
  end
  
  def can?(%User{} = user, :delete, %Post{} = post) do
    can?(user, :delete_all) || (post.user_id == user.id)
  end
  
  def can?(_user, _action, _resource), do: false
  
  def authorize!(%User{} = user, action, resource \\ nil) do
    permitted = if resource do
      can?(user, action, resource)
    else
      can?(user, action)
    end
    
    if permitted do
      :ok
    else
      {:error, :forbidden}
    end
  end
end

# Controller with authorization
defmodule MyAppWeb.PostController do
  use MyAppWeb, :controller
  
  alias MyApp.{Blog, Auth.Policy}
  
  plug :require_auth
  
  def update(conn, %{"id" => id, "post" => params}) do
    post = Blog.get_post!(id)
    
    with :ok <- Policy.authorize!(conn.assigns.current_user, :update, post),
         {:ok, post} <- Blog.update_post(post, params) do
      render(conn, :show, post: post)
    else
      {:error, :forbidden} ->
        conn |> put_status(:forbidden) |> json(%{error: "Forbidden"})
      {:error, changeset} ->
        conn |> put_status(:unprocessable_entity) |> render(:errors, changeset: changeset)
    end
  end
  
  defp require_auth(conn, _opts) do
    case conn.assigns[:current_user] do
      nil ->
        conn |> put_status(:unauthorized) |> json(%{error: "Unauthorized"}) |> halt()
      _ ->
        conn
    end
  end
end
```

---

## Step 274: OAuth2 Integration

```elixir
# mix.exs
{:ueberauth, "~> 0.10"},
{:ueberauth_google, "~> 0.12"},
{:ueberauth_github, "~> 0.8"}

# config/config.exs
config :ueberauth, Ueberauth,
  providers: [
    google: {Ueberauth.Strategy.Google, []},
    github: {Ueberauth.Strategy.Github, [default_scope: "user:email"]}
  ]

config :ueberauth, Ueberauth.Strategy.Google.OAuth,
  client_id:     System.get_env("GOOGLE_CLIENT_ID"),
  client_secret: System.get_env("GOOGLE_CLIENT_SECRET")

# Router
scope "/auth", MyAppWeb do
  pipe_through :browser
  
  get  "/:provider",          AuthController, :request
  get  "/:provider/callback", AuthController, :callback
  delete "/logout",           AuthController, :delete
end

# Controller
defmodule MyAppWeb.AuthController do
  use MyAppWeb, :controller
  plug Ueberauth
  
  def callback(%{assigns: %{ueberauth_auth: auth}} = conn, _params) do
    case MyApp.Accounts.find_or_create_from_oauth(auth) do
      {:ok, user} ->
        token = MyApp.Accounts.generate_user_session_token(user)
        
        conn
        |> put_session(:user_token, token)
        |> put_flash(:info, "Logged in successfully!")
        |> redirect(to: ~p"/dashboard")
      
      {:error, reason} ->
        conn
        |> put_flash(:error, "Authentication failed: #{inspect(reason)}")
        |> redirect(to: ~p"/login")
    end
  end
  
  def callback(%{assigns: %{ueberauth_failure: failure}} = conn, _params) do
    conn
    |> put_flash(:error, "Failed: #{inspect(failure)}")
    |> redirect(to: ~p"/login")
  end
  
  def delete(conn, _params) do
    token = get_session(conn, :user_token)
    token && MyApp.Accounts.delete_user_session_token(token)
    
    conn
    |> clear_session()
    |> put_flash(:info, "Logged out.")
    |> redirect(to: ~p"/")
  end
end

defmodule MyApp.Accounts do
  def find_or_create_from_oauth(%Ueberauth.Auth{} = auth) do
    email = auth.info.email
    
    case get_user_by_email(email) do
      nil ->
        create_user_from_oauth(auth)
      user ->
        ensure_provider_connected(user, auth)
    end
  end
  
  defp create_user_from_oauth(auth) do
    %User{}
    |> User.oauth_changeset(%{
         email: auth.info.email,
         name:  auth.info.name,
         avatar_url: auth.info.image,
         provider: auth.provider,
         provider_id: auth.uid
       })
    |> Repo.insert()
  end
end
```

---

## Step 275: Two-Factor Authentication

```elixir
defmodule MyApp.Auth.TOTP do
  @app_name "My App"
  
  def generate_secret do
    :crypto.strong_rand_bytes(20) |> Base32.encode()
  end
  
  def generate_otp_uri(user, secret) do
    "otpauth://totp/#{@app_name}:#{user.email}?secret=#{secret}&issuer=#{@app_name}"
  end
  
  def verify_code(secret, code) do
    # Use NimbleTOTP or similar library
    NimbleTOTP.valid?(secret, code)
  end
  
  def generate_backup_codes(count \\ 10) do
    Enum.map(1..count, fn _ ->
      :crypto.strong_rand_bytes(5)
      |> Base.encode16()
      |> String.downcase()
    end)
  end
end

defmodule MyAppWeb.TwoFactorController do
  use MyAppWeb, :controller
  
  def setup(conn, _params) do
    user = conn.assigns.current_user
    secret = MyApp.Auth.TOTP.generate_secret()
    uri = MyApp.Auth.TOTP.generate_otp_uri(user, secret)
    
    conn
    |> put_session(:pending_2fa_secret, secret)
    |> render(:setup, qr_uri: uri, secret: secret)
  end
  
  def verify_setup(conn, %{"code" => code}) do
    user = conn.assigns.current_user
    secret = get_session(conn, :pending_2fa_secret)
    
    if MyApp.Auth.TOTP.verify_code(secret, code) do
      backup_codes = MyApp.Auth.TOTP.generate_backup_codes()
      {:ok, _} = MyApp.Accounts.enable_2fa(user, secret, backup_codes)
      
      conn
      |> delete_session(:pending_2fa_secret)
      |> put_flash(:info, "2FA enabled!")
      |> render(:backup_codes, codes: backup_codes)
    else
      conn
      |> put_flash(:error, "Invalid code")
      |> redirect(to: ~p"/settings/2fa")
    end
  end
end
```

---

## Step 276: API Keys

```elixir
defmodule MyApp.Auth.APIKey do
  import Ecto.Query
  alias MyApp.{Repo, Auth.APIKey}
  
  schema "api_keys" do
    field :key_hash,     :string
    field :name,         :string
    field :permissions,  {:array, :string}
    field :last_used_at, :naive_datetime
    field :expires_at,   :naive_datetime
    
    belongs_to :user, MyApp.Accounts.User
    
    timestamps()
  end
  
  def generate(user, name, permissions \\ ["read"]) do
    raw_key = "sk_" <> (:crypto.strong_rand_bytes(32) |> Base.url_encode64(padding: false))
    key_hash = hash_key(raw_key)
    
    %__MODULE__{}
    |> cast(%{
         user_id:    user.id,
         name:       name,
         key_hash:   key_hash,
         permissions: permissions,
         expires_at: NaiveDateTime.add(NaiveDateTime.utc_now(), 365 * 24 * 3600)
       }, [:user_id, :name, :key_hash, :permissions, :expires_at])
    |> Repo.insert()
    |> case do
         {:ok, api_key} -> {:ok, %{api_key | key_hash: raw_key}}  # return raw key once
         error -> error
       end
  end
  
  def verify(raw_key) do
    key_hash = hash_key(raw_key)
    
    case Repo.get_by(__MODULE__, key_hash: key_hash) do
      nil -> {:error, :invalid}
      key ->
        cond do
          key.expires_at && NaiveDateTime.compare(key.expires_at, NaiveDateTime.utc_now()) == :lt ->
            {:error, :expired}
          true ->
            update_last_used(key)
            {:ok, key}
        end
    end
  end
  
  defp hash_key(key) do
    :crypto.hash(:sha256, key) |> Base.url_encode64()
  end
  
  defp update_last_used(key) do
    key
    |> Ecto.Changeset.change(last_used_at: NaiveDateTime.utc_now())
    |> Repo.update()
  end
end

# API Key Plug
defmodule MyAppWeb.Plugs.VerifyAPIKey do
  import Plug.Conn
  alias MyApp.Auth.APIKey
  
  def init(opts), do: opts
  
  def call(conn, opts) do
    required_permission = Keyword.get(opts, :permission, "read")
    
    case get_req_header(conn, "x-api-key") do
      [key] ->
        case APIKey.verify(key) do
          {:ok, api_key} ->
            if required_permission in api_key.permissions do
              conn
              |> assign(:api_key, api_key)
              |> assign(:current_user_id, api_key.user_id)
            else
              forbidden(conn)
            end
          
          {:error, _} -> unauthorized(conn)
        end
      
      [] -> unauthorized(conn)
    end
  end
  
  defp unauthorized(conn) do
    conn
    |> put_status(:unauthorized)
    |> Phoenix.Controller.json(%{error: "Invalid API key"})
    |> halt()
  end
  
  defp forbidden(conn) do
    conn
    |> put_status(:forbidden)
    |> Phoenix.Controller.json(%{error: "Insufficient permissions"})
    |> halt()
  end
end
```

---

## Step 277: Session Security

```elixir
defmodule MyAppWeb.SessionPlug do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    conn
    |> load_current_user()
    |> check_session_validity()
  end
  
  defp load_current_user(conn) do
    case get_session(conn, :user_token) do
      nil -> assign(conn, :current_user, nil)
      token ->
        case MyApp.Accounts.get_user_by_session_token(token) do
          nil  -> conn |> clear_session() |> assign(:current_user, nil)
          user -> assign(conn, :current_user, user)
        end
    end
  end
  
  defp check_session_validity(conn) do
    if user = conn.assigns[:current_user] do
      # Check if account is still active
      if user.deactivated_at do
        conn
        |> clear_session()
        |> assign(:current_user, nil)
        |> Phoenix.Controller.put_flash(:error, "Your account has been deactivated.")
        |> Phoenix.Controller.redirect(to: "/")
        |> halt()
      else
        conn
      end
    else
      conn
    end
  end
end
```

---

## Step 278: Password Reset Flow

```elixir
defmodule MyApp.Accounts do
  def deliver_user_reset_password_instructions(user, reset_url_fun) do
    {encoded_token, user_token} = UserToken.build_email_token(user, "reset_password")
    Repo.insert!(user_token)
    
    UserNotifier.deliver_reset_password_instructions(
      user,
      reset_url_fun.(encoded_token)
    )
  end
  
  def get_user_by_reset_password_token(token) do
    with {:ok, query} <- UserToken.verify_email_token_query(token, "reset_password"),
         %User{} = user <- Repo.one(query) do
      user
    else
      _ -> nil
    end
  end
  
  def reset_user_password(user, attrs) do
    Ecto.Multi.new()
    |> Ecto.Multi.update(:user, User.password_changeset(user, attrs))
    |> Ecto.Multi.delete_all(:tokens, UserToken.by_user_and_contexts_query(user, ["reset_password"]))
    |> Repo.transaction()
    |> case do
         {:ok, %{user: user}} -> {:ok, user}
         {:error, :user, changeset, _} -> {:error, changeset}
       end
  end
end

defmodule MyAppWeb.UserResetPasswordController do
  use MyAppWeb, :controller
  
  def new(conn, _params) do
    render(conn, :new)
  end
  
  def create(conn, %{"user" => %{"email" => email}}) do
    # Always show success even if email not found (prevent enumeration)
    if user = MyApp.Accounts.get_user_by_email(email) do
      MyApp.Accounts.deliver_user_reset_password_instructions(
        user,
        fn token -> url(~p"/users/reset_password/#{token}") end
      )
    end
    
    conn
    |> put_flash(:info, "If that email is registered, you'll receive password reset instructions.")
    |> redirect(to: ~p"/")
  end
  
  def edit(conn, %{"token" => token}) do
    case MyApp.Accounts.get_user_by_reset_password_token(token) do
      nil ->
        conn
        |> put_flash(:error, "Reset password link is invalid or has expired.")
        |> redirect(to: ~p"/users/reset_password")
      
      user ->
        changeset = MyApp.Accounts.change_user_password(user)
        render(conn, :edit, changeset: changeset, token: token)
    end
  end
end
```

---

## Step 279: Audit Logging

```elixir
defmodule MyApp.AuditLog do
  use Ecto.Schema
  import Ecto.Query
  
  schema "audit_logs" do
    field :user_id,    :integer
    field :action,     :string
    field :resource,   :string
    field :resource_id, :integer
    field :ip_address, :string
    field :metadata,   :map
    
    timestamps(updated_at: false)
  end
  
  def log(user_id, action, resource, resource_id, opts \\ []) do
    attrs = %{
      user_id:    user_id,
      action:     action,
      resource:   resource,
      resource_id: resource_id,
      ip_address: Keyword.get(opts, :ip),
      metadata:   Keyword.get(opts, :metadata, %{})
    }
    
    %__MODULE__{}
    |> Ecto.Changeset.cast(attrs, [:user_id, :action, :resource, :resource_id, :ip_address, :metadata])
    |> MyApp.Repo.insert()
  end
  
  def get_user_activity(user_id, limit \\ 50) do
    from(al in __MODULE__,
      where: al.user_id == ^user_id,
      order_by: [desc: al.inserted_at],
      limit: ^limit
    )
    |> MyApp.Repo.all()
  end
end

# Plug to auto-log
defmodule MyAppWeb.Plugs.AuditLog do
  def init(opts), do: opts
  
  def call(conn, _opts) do
    Plug.Conn.register_before_send(conn, fn conn ->
      if conn.assigns[:current_user] && conn.method in ["POST", "PUT", "PATCH", "DELETE"] do
        user = conn.assigns.current_user
        ip = conn.remote_ip |> Tuple.to_list() |> Enum.join(".")
        
        MyApp.AuditLog.log(
          user.id,
          conn.method,
          conn.request_path,
          nil,
          ip: ip
        )
      end
      conn
    end)
  end
end
```

---

## Step 280: Complete Auth System

```elixir
defmodule MyAppWeb.Router do
  use MyAppWeb, :router
  
  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
    plug :protect_from_forgery
    plug :put_secure_browser_headers
    plug MyAppWeb.SessionPlug      # load current user
    plug MyAppWeb.Plugs.AuditLog   # audit logging
  end
  
  pipeline :api do
    plug :accepts, ["json"]
  end
  
  pipeline :authenticated do
    plug MyAppWeb.Plugs.RequireAuth
  end
  
  pipeline :api_authenticated do
    plug MyAppWeb.Plugs.VerifyJWT
  end
  
  # Public routes
  scope "/", MyAppWeb do
    pipe_through :browser
    get "/", PageController, :home
  end
  
  # Auth routes (not logged in)
  scope "/users", MyAppWeb do
    pipe_through [:browser, :redirect_if_authenticated]
    
    get "/register", RegistrationController, :new
    post "/register", RegistrationController, :create
    get "/log_in", SessionController, :new
    post "/log_in", SessionController, :create
    get "/reset_password", UserResetPasswordController, :new
    post "/reset_password", UserResetPasswordController, :create
  end
  
  # Protected routes
  scope "/", MyAppWeb do
    pipe_through [:browser, :authenticated]
    
    delete "/users/log_out", SessionController, :delete
    get "/dashboard", DashboardController, :index
    resources "/posts", PostController
  end
  
  # OAuth
  scope "/auth", MyAppWeb do
    pipe_through :browser
    get "/:provider", AuthController, :request
    get "/:provider/callback", AuthController, :callback
  end
  
  # API
  scope "/api/v1", MyAppWeb.API do
    pipe_through [:api]
    post "/login", AuthController, :login
    post "/refresh", AuthController, :refresh
  end
  
  scope "/api/v1", MyAppWeb.API do
    pipe_through [:api, :api_authenticated]
    resources "/users", UserController, except: [:new, :edit]
    resources "/posts", PostController, except: [:new, :edit]
  end
end
```

---

## สรุป Part 25

✅ **Step 271** - Session-based auth  
✅ **Step 272** - JWT authentication  
✅ **Step 273** - RBAC authorization  
✅ **Step 274** - OAuth2 integration  
✅ **Step 275** - Two-factor authentication  
✅ **Step 276** - API keys  
✅ **Step 277** - Session security  
✅ **Step 278** - Password reset flow  
✅ **Step 279** - Audit logging  
✅ **Step 280** - Complete auth system  

➡️ [Part 26: Performance และ Optimization](./part-26-performance.md)
