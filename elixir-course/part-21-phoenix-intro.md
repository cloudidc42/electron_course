# Part 21: Phoenix Framework - Introduction (Steps 221-240)

## Step 221: Phoenix คืออะไร?

```
Phoenix Framework
- Web framework สำหรับ Elixir โดย Chris McCord
- Built on top of Plug (middleware system)
- Real-time ด้วย Phoenix Channels + LiveView
- High performance: Elixir concurrency model
- Production-ready: Discord, Pinterest, Bleacher Report ใช้งาน
```

### 221.1 สร้าง Phoenix Project

```bash
# ติดตั้ง Phoenix generator
mix archive.install hex phx_new

# สร้าง project ใหม่
mix phx.new my_app
mix phx.new my_app --database postgres  # with Postgres
mix phx.new my_api --no-html --no-assets  # API only
mix phx.new my_app --live               # with LiveView

cd my_app

# Setup database
mix ecto.create
mix ecto.migrate

# Start server
mix phx.server
# หรือด้วย IEx
iex -S mix phx.server

# Visit http://localhost:4000
```

---

## Step 222: Phoenix Project Structure

```
my_app/
├── lib/
│   ├── my_app/                    # Business logic
│   │   ├── accounts.ex            # Context
│   │   ├── accounts/
│   │   │   ├── user.ex            # Schema
│   │   │   └── user_token.ex
│   │   ├── blog.ex                # Another context
│   │   └── repo.ex                # Ecto Repo
│   └── my_app_web/               # Web layer
│       ├── components/
│       │   ├── core_components.ex # Shared components
│       │   └── layouts.ex
│       ├── controllers/
│       │   ├── page_controller.ex
│       │   └── user_controller.ex
│       ├── live/                  # LiveView
│       │   └── user_live/
│       ├── views/                 # (legacy, use components)
│       ├── router.ex              # Routes
│       ├── endpoint.ex            # Cowboy/Bandit config
│       └── telemetry.ex
├── assets/
│   ├── css/
│   │   └── app.css
│   └── js/
│       └── app.js
├── priv/
│   └── repo/
│       └── migrations/
├── test/
├── config/
└── mix.exs
```

---

## Step 223: Router

```elixir
# lib/my_app_web/router.ex
defmodule MyAppWeb.Router do
  use MyAppWeb, :router
  
  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end
  
  pipeline :api do
    plug :accepts, ["json"]
  end
  
  pipeline :authenticated do
    plug MyAppWeb.Plugs.RequireAuth
  end
  
  # Public routes
  scope "/", MyAppWeb do
    pipe_through :browser
    
    get "/", PageController, :home
    get "/about", PageController, :about
  end
  
  # Auth routes
  scope "/auth", MyAppWeb do
    pipe_through :browser
    
    get  "/login",  SessionController, :new
    post "/login",  SessionController, :create
    delete "/logout", SessionController, :delete
    
    get  "/register", RegistrationController, :new
    post "/register", RegistrationController, :create
  end
  
  # Protected routes
  scope "/", MyAppWeb do
    pipe_through [:browser, :authenticated]
    
    resources "/users", UserController
    resources "/posts", PostController do
      resources "/comments", CommentController, only: [:create, :delete]
    end
    
    live "/dashboard", DashboardLive.Index
    live "/settings",  SettingsLive.Index
  end
  
  # API routes
  scope "/api", MyAppWeb do
    pipe_through :api
    
    scope "/v1" do
      resources "/users",   API.UserController, except: [:new, :edit]
      resources "/products", API.ProductController, except: [:new, :edit]
    end
  end
  
  # Dev routes (only in dev)
  if Mix.env() in [:dev, :test] do
    import Phoenix.LiveDashboard.Router
    
    scope "/dev" do
      pipe_through :browser
      live_dashboard "/dashboard", metrics: MyAppWeb.Telemetry
    end
  end
end
```

---

## Step 224: Controllers

```elixir
defmodule MyAppWeb.UserController do
  use MyAppWeb, :controller
  
  alias MyApp.Accounts
  alias MyApp.Accounts.User
  
  # index - list all users
  def index(conn, _params) do
    users = Accounts.list_users()
    render(conn, :index, users: users)
  end
  
  # show - show single user
  def show(conn, %{"id" => id}) do
    user = Accounts.get_user!(id)
    render(conn, :show, user: user)
  end
  
  # new - show create form
  def new(conn, _params) do
    changeset = Accounts.change_user(%User{})
    render(conn, :new, changeset: changeset)
  end
  
  # create - handle POST
  def create(conn, %{"user" => user_params}) do
    case Accounts.create_user(user_params) do
      {:ok, user} ->
        conn
        |> put_flash(:info, "User created successfully.")
        |> redirect(to: ~p"/users/#{user}")
      
      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, :new, changeset: changeset)
    end
  end
  
  # edit - show edit form
  def edit(conn, %{"id" => id}) do
    user = Accounts.get_user!(id)
    changeset = Accounts.change_user(user)
    render(conn, :edit, user: user, changeset: changeset)
  end
  
  # update - handle PUT/PATCH
  def update(conn, %{"id" => id, "user" => user_params}) do
    user = Accounts.get_user!(id)
    
    case Accounts.update_user(user, user_params) do
      {:ok, user} ->
        conn
        |> put_flash(:info, "User updated successfully.")
        |> redirect(to: ~p"/users/#{user}")
      
      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, :edit, user: user, changeset: changeset)
    end
  end
  
  # delete - handle DELETE
  def delete(conn, %{"id" => id}) do
    user = Accounts.get_user!(id)
    {:ok, _user} = Accounts.delete_user(user)
    
    conn
    |> put_flash(:info, "User deleted successfully.")
    |> redirect(to: ~p"/users")
  end
end
```

---

## Step 225: HEEx Templates

```elixir
# lib/my_app_web/controllers/user_html.ex
defmodule MyAppWeb.UserHTML do
  use MyAppWeb, :html
  
  embed_templates "user_html/*"  # หรือใช้ ~H sigil
end

# lib/my_app_web/controllers/user_html/index.html.heex
# หรือ inline ใน module:
defmodule MyAppWeb.UserHTML do
  use MyAppWeb, :html
  
  def index(assigns) do
    ~H"""
    <div class="container">
      <h1>Users</h1>
      
      <.link href={~p"/users/new"} class="btn btn-primary">
        New User
      </.link>
      
      <table class="table">
        <thead>
          <tr>
            <th>Name</th>
            <th>Email</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          <%= for user <- @users do %>
            <tr>
              <td><%= user.name %></td>
              <td><%= user.email %></td>
              <td>
                <.link href={~p"/users/#{user}"}>Show</.link>
                <.link href={~p"/users/#{user}/edit"}>Edit</.link>
                <.link href={~p"/users/#{user}"} method="delete"
                       data-confirm="Are you sure?">
                  Delete
                </.link>
              </td>
            </tr>
          <% end %>
        </tbody>
      </table>
    </div>
    """
  end
  
  def show(assigns) do
    ~H"""
    <div class="container">
      <h1><%= @user.name %></h1>
      <p>Email: <%= @user.email %></p>
      
      <.link href={~p"/users/#{@user}/edit"} class="btn">Edit</.link>
      <.link href={~p"/users"} class="btn">Back</.link>
    </div>
    """
  end
end
```

---

## Step 226: Plugs

```elixir
# Custom Plug
defmodule MyAppWeb.Plugs.RequireAuth do
  import Plug.Conn
  import Phoenix.Controller
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    if conn.assigns[:current_user] do
      conn
    else
      conn
      |> put_flash(:error, "You must be logged in to access this page.")
      |> redirect(to: "/login")
      |> halt()  # หยุด pipeline
    end
  end
end

defmodule MyAppWeb.Plugs.SetCurrentUser do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    case get_session(conn, :user_id) do
      nil ->
        assign(conn, :current_user, nil)
      
      user_id ->
        case MyApp.Accounts.get_user(user_id) do
          {:ok, user} -> assign(conn, :current_user, user)
          _           -> assign(conn, :current_user, nil)
        end
    end
  end
end

defmodule MyAppWeb.Plugs.RateLimiter do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    ip = conn.remote_ip |> Tuple.to_list() |> Enum.join(".")
    
    case RateLimiter.check(ip) do
      :allow ->
        conn
      {:deny, retry_after} ->
        conn
        |> put_resp_header("retry-after", Integer.to_string(div(retry_after, 1000)))
        |> send_resp(429, "Too Many Requests")
        |> halt()
    end
  end
end
```

---

## Step 227: Contexts (Business Logic Layer)

```elixir
# lib/my_app/accounts.ex
defmodule MyApp.Accounts do
  @moduledoc """
  The Accounts context handles users and authentication.
  """
  
  import Ecto.Query, warn: false
  alias MyApp.Repo
  alias MyApp.Accounts.{User, UserToken}
  
  # User queries
  
  def list_users do
    Repo.all(User)
  end
  
  def get_user(id) do
    case Repo.get(User, id) do
      nil  -> {:error, :not_found}
      user -> {:ok, user}
    end
  end
  
  def get_user!(id) do
    Repo.get!(User, id)
  end
  
  def get_user_by_email(email) do
    Repo.get_by(User, email: email)
  end
  
  # User mutations
  
  def create_user(attrs \\ %{}) do
    %User{}
    |> User.registration_changeset(attrs)
    |> Repo.insert()
  end
  
  def update_user(%User{} = user, attrs) do
    user
    |> User.update_changeset(attrs)
    |> Repo.update()
  end
  
  def delete_user(%User{} = user) do
    Repo.delete(user)
  end
  
  def change_user(%User{} = user, attrs \\ %{}) do
    User.update_changeset(user, attrs)
  end
  
  # Authentication
  
  def authenticate_user(email, password) do
    user = get_user_by_email(email)
    
    cond do
      user && User.valid_password?(user, password) ->
        {:ok, user}
      user ->
        {:error, :invalid_credentials}
      true ->
        User.dummy_hash_password()  # prevent timing attacks
        {:error, :invalid_credentials}
    end
  end
  
  def generate_user_session_token(user) do
    {token, user_token} = UserToken.build_session_token(user)
    Repo.insert!(user_token)
    token
  end
  
  def get_user_by_session_token(token) do
    {:ok, query} = UserToken.verify_session_token_query(token)
    Repo.one(query)
  end
  
  def delete_user_session_token(token) do
    Repo.delete_all(UserToken.by_token_query(token))
    :ok
  end
end
```

---

## Step 228: Ecto Schemas

```elixir
# lib/my_app/accounts/user.ex
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "users" do
    field :name,           :string
    field :email,          :string
    field :password,       :string, virtual: true, redact: true
    field :hashed_password, :string, redact: true
    field :confirmed_at,   :naive_datetime
    
    has_many :posts, MyApp.Blog.Post
    has_one  :profile, MyApp.Accounts.Profile
    
    timestamps()
  end
  
  @doc "Changeset for registration"
  def registration_changeset(user, attrs, opts \\ []) do
    user
    |> cast(attrs, [:name, :email, :password])
    |> validate_required([:name, :email, :password])
    |> validate_email()
    |> validate_password(opts)
  end
  
  @doc "Changeset for update"
  def update_changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :email])
    |> validate_required([:name, :email])
    |> validate_email()
  end
  
  defp validate_email(changeset) do
    changeset
    |> validate_format(:email, ~r/^[^\s]+@[^\s]+$/, message: "must have the @ sign and no spaces")
    |> validate_length(:email, max: 160)
    |> unsafe_validate_unique(:email, MyApp.Repo)
    |> unique_constraint(:email)
  end
  
  defp validate_password(changeset, opts) do
    changeset
    |> validate_required([:password])
    |> validate_length(:password, min: 8, max: 72)
    |> maybe_hash_password(opts)
  end
  
  defp maybe_hash_password(changeset, opts) do
    hash_password? = Keyword.get(opts, :hash_password, true)
    password = get_change(changeset, :password)
    
    if hash_password? && password && changeset.valid? do
      changeset
      |> validate_length(:password, max: 72, count: :bytes)
      |> put_change(:hashed_password, Bcrypt.hash_pwd_salt(password))
      |> delete_change(:password)
    else
      changeset
    end
  end
  
  def valid_password?(%User{hashed_password: hash}, password)
    when is_binary(hash) and byte_size(password) > 0 do
    Bcrypt.verify_pass(password, hash)
  end
  def valid_password?(_, _), do: Bcrypt.no_user_verify() && false
  
  def dummy_hash_password, do: Bcrypt.no_user_verify()
end
```

---

## Step 229: Ecto Queries

```elixir
defmodule MyApp.Blog do
  import Ecto.Query
  alias MyApp.{Repo, Blog.Post}
  
  # Basic queries
  def list_posts do
    Post
    |> order_by([p], desc: p.inserted_at)
    |> Repo.all()
  end
  
  # With preloads
  def list_posts_with_user do
    Post
    |> preload(:user)
    |> order_by([p], desc: p.inserted_at)
    |> Repo.all()
  end
  
  # Filter
  def list_published_posts do
    from(p in Post,
      where: p.status == :published,
      order_by: [desc: p.published_at],
      limit: 20
    )
    |> Repo.all()
  end
  
  # Pagination
  def paginate_posts(page, per_page \\ 10) do
    offset = (page - 1) * per_page
    
    posts = Post
      |> where(status: :published)
      |> order_by([p], desc: p.inserted_at)
      |> limit(^per_page)
      |> offset(^offset)
      |> Repo.all()
    
    total = Post |> where(status: :published) |> Repo.aggregate(:count)
    
    %{
      posts: posts,
      page: page,
      per_page: per_page,
      total: total,
      total_pages: ceil(total / per_page)
    }
  end
  
  # Full-text search
  def search_posts(query_string) do
    search_term = "%#{query_string}%"
    
    from(p in Post,
      where: ilike(p.title, ^search_term) or ilike(p.content, ^search_term),
      order_by: [desc: p.inserted_at]
    )
    |> Repo.all()
  end
  
  # Aggregates
  def post_stats do
    from(p in Post,
      group_by: p.status,
      select: {p.status, count(p.id)}
    )
    |> Repo.all()
    |> Map.new()
  end
  
  # Join
  def posts_by_user(user_id) do
    from(p in Post,
      join: u in assoc(p, :user),
      where: u.id == ^user_id,
      preload: [user: u]
    )
    |> Repo.all()
  end
end
```

---

## Step 230: JSON API Controller

```elixir
defmodule MyAppWeb.API.UserController do
  use MyAppWeb, :controller
  
  alias MyApp.Accounts
  alias MyApp.Accounts.User
  
  action_fallback MyAppWeb.FallbackController
  
  def index(conn, _params) do
    users = Accounts.list_users()
    render(conn, :index, users: users)
  end
  
  def show(conn, %{"id" => id}) do
    with {:ok, user} <- Accounts.get_user(id) do
      render(conn, :show, user: user)
    end
  end
  
  def create(conn, %{"user" => user_params}) do
    with {:ok, %User{} = user} <- Accounts.create_user(user_params) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", ~p"/api/v1/users/#{user}")
      |> render(:show, user: user)
    end
  end
  
  def update(conn, %{"id" => id, "user" => user_params}) do
    with {:ok, user} <- Accounts.get_user(id),
         {:ok, %User{} = user} <- Accounts.update_user(user, user_params) do
      render(conn, :show, user: user)
    end
  end
  
  def delete(conn, %{"id" => id}) do
    with {:ok, user} <- Accounts.get_user(id),
         {:ok, %User{}} <- Accounts.delete_user(user) do
      send_resp(conn, :no_content, "")
    end
  end
end

# JSON rendering
defmodule MyAppWeb.API.UserJSON do
  def index(%{users: users}) do
    %{data: Enum.map(users, &data/1)}
  end
  
  def show(%{user: user}) do
    %{data: data(user)}
  end
  
  defp data(%User{} = user) do
    %{
      id:         user.id,
      name:       user.name,
      email:      user.email,
      created_at: user.inserted_at
    }
  end
end

# FallbackController
defmodule MyAppWeb.FallbackController do
  use MyAppWeb, :controller
  
  def call(conn, {:error, :not_found}) do
    conn
    |> put_status(:not_found)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"404")
  end
  
  def call(conn, {:error, %Ecto.Changeset{} = changeset}) do
    conn
    |> put_status(:unprocessable_entity)
    |> put_view(json: MyAppWeb.ChangesetJSON)
    |> render(:error, changeset: changeset)
  end
  
  def call(conn, {:error, :unauthorized}) do
    conn
    |> put_status(:unauthorized)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"401")
  end
end
```

---

## สรุป Part 21

✅ **Step 221** - Phoenix overview + project creation  
✅ **Step 222** - Project structure  
✅ **Step 223** - Router (pipelines, scopes, resources)  
✅ **Step 224** - Controllers (CRUD actions)  
✅ **Step 225** - HEEx templates  
✅ **Step 226** - Plugs (middleware)  
✅ **Step 227** - Contexts (business logic)  
✅ **Step 228** - Ecto schemas  
✅ **Step 229** - Ecto queries  
✅ **Step 230** - JSON API  

➡️ [Part 22: Phoenix LiveView](./part-22-liveview.md)
