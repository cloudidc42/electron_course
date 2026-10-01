# Part 30: REST API Best Practices (Steps 321-340)

## Step 321: API Design Principles

```
REST Best Practices:
- Use nouns not verbs: /users not /getUsers
- Use HTTP methods: GET/POST/PUT/PATCH/DELETE
- Use proper status codes
- Version your API: /api/v1/
- Consistent error format
- Pagination
- Rate limiting
- Authentication (JWT Bearer)
- OpenAPI/Swagger docs
```

---

## Step 322: API Router Structure

```elixir
defmodule MyAppWeb.Router do
  use MyAppWeb, :router
  
  pipeline :api do
    plug :accepts, ["json"]
    plug MyAppWeb.Plugs.RateLimiter
    plug MyAppWeb.Plugs.RequestLogger
  end
  
  pipeline :authenticated do
    plug MyAppWeb.Plugs.BearerAuth
  end
  
  pipeline :admin do
    plug MyAppWeb.Plugs.RequireRole, role: :admin
  end
  
  scope "/api/v1", MyAppWeb.API.V1 do
    pipe_through :api
    
    post "/auth/register", AuthController, :register
    post "/auth/login",    AuthController, :login
    post "/auth/refresh",  AuthController, :refresh
    
    scope "/" do
      pipe_through :authenticated
      
      delete "/auth/logout", AuthController, :logout
      
      resources "/users",    UserController, except: [:new, :edit] do
        resources "/posts",  PostController, except: [:new, :edit]
      end
      
      resources "/posts", PostController, except: [:new, :edit] do
        resources "/comments", CommentController, except: [:new, :edit]
        post "/like",    PostController, :like
        delete "/like",  PostController, :unlike
      end
      
      scope "/me" do
        get  "/",      ProfileController, :show
        put  "/",      ProfileController, :update
        get  "/posts", ProfileController, :posts
      end
    end
    
    scope "/admin" do
      pipe_through [:authenticated, :admin]
      resources "/users", Admin.UserController, except: [:new, :edit]
    end
  end
end
```

---

## Step 323: Controller Pattern

```elixir
defmodule MyAppWeb.API.V1.PostController do
  use MyAppWeb, :controller
  
  alias MyApp.Blog
  alias MyAppWeb.API.V1.PostView
  
  action_fallback MyAppWeb.FallbackController
  
  def index(conn, params) do
    with {:ok, query} <- validate_index_params(params) do
      posts = Blog.list_posts(query)
      render(conn, :index, posts: posts)
    end
  end
  
  def show(conn, %{"id" => id}) do
    with {:ok, post} <- Blog.get_post(id) do
      render(conn, :show, post: post)
    end
  end
  
  def create(conn, %{"post" => post_params}) do
    user = conn.assigns.current_user
    
    with {:ok, post} <- Blog.create_post(user, post_params) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", ~p"/api/v1/posts/#{post}")
      |> render(:show, post: post)
    end
  end
  
  def update(conn, %{"id" => id, "post" => post_params}) do
    user = conn.assigns.current_user
    
    with {:ok, post} <- Blog.get_post(id),
         :ok         <- authorize(user, post, :update),
         {:ok, post} <- Blog.update_post(post, post_params) do
      render(conn, :show, post: post)
    end
  end
  
  def delete(conn, %{"id" => id}) do
    user = conn.assigns.current_user
    
    with {:ok, post} <- Blog.get_post(id),
         :ok         <- authorize(user, post, :delete),
         {:ok, _}    <- Blog.delete_post(post) do
      send_resp(conn, :no_content, "")
    end
  end
  
  defp validate_index_params(params) do
    # validate pagination, filters, sorting
    {:ok, params}
  end
  
  defp authorize(user, post, action) do
    if MyApp.Policy.can?(user, action, post) do
      :ok
    else
      {:error, :forbidden}
    end
  end
end
```

---

## Step 324: FallbackController

```elixir
defmodule MyAppWeb.FallbackController do
  use MyAppWeb, :controller
  
  def call(conn, {:error, %Ecto.Changeset{} = cs}) do
    conn
    |> put_status(:unprocessable_entity)
    |> put_view(json: MyAppWeb.ChangesetJSON)
    |> render(:error, changeset: cs)
  end
  
  def call(conn, {:error, :not_found}) do
    conn
    |> put_status(:not_found)
    |> json(%{error: %{code: "NOT_FOUND", message: "Resource not found"}})
  end
  
  def call(conn, {:error, :unauthorized}) do
    conn
    |> put_status(:unauthorized)
    |> json(%{error: %{code: "UNAUTHORIZED", message: "Authentication required"}})
  end
  
  def call(conn, {:error, :forbidden}) do
    conn
    |> put_status(:forbidden)
    |> json(%{error: %{code: "FORBIDDEN", message: "Access denied"}})
  end
  
  def call(conn, {:error, :conflict}) do
    conn
    |> put_status(:conflict)
    |> json(%{error: %{code: "CONFLICT", message: "Resource conflict"}})
  end
  
  def call(conn, {:error, reason}) when is_binary(reason) do
    conn
    |> put_status(:bad_request)
    |> json(%{error: %{code: "BAD_REQUEST", message: reason}})
  end
end
```

---

## Step 325: JSON Views

```elixir
defmodule MyAppWeb.API.V1.PostJSON do
  def index(%{posts: posts}) do
    %{
      data: Enum.map(posts, &data/1),
      meta: %{count: length(posts)}
    }
  end
  
  def show(%{post: post}) do
    %{data: data(post)}
  end
  
  defp data(post) do
    base = %{
      id:         post.id,
      type:       "post",
      attributes: %{
        title:      post.title,
        body:       post.body,
        status:     post.status,
        slug:       post.slug,
        views:      post.views,
        created_at: post.inserted_at,
        updated_at: post.updated_at
      }
    }
    
    add_relationships(base, post)
  end
  
  defp add_relationships(base, %{author: %Ecto.Association.NotLoaded{}}), do: base
  defp add_relationships(base, %{author: author}) do
    put_in(base, [:relationships], %{
      author: %{
        data: %{id: author.id, type: "user"},
        attributes: %{name: author.name, avatar: author.avatar}
      }
    })
  end
end

# Changeset errors
defmodule MyAppWeb.ChangesetJSON do
  def error(%{changeset: changeset}) do
    %{
      error: %{
        code: "VALIDATION_ERROR",
        message: "Validation failed",
        details: translate_errors(changeset)
      }
    }
  end
  
  defp translate_errors(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## Step 326: Pagination

```elixir
defmodule MyAppWeb.API.Pagination do
  import Ecto.Query
  
  @default_page 1
  @default_per_page 20
  @max_per_page 100
  
  def paginate(query, params) do
    page     = get_page(params)
    per_page = get_per_page(params)
    offset   = (page - 1) * per_page
    
    total = query |> exclude(:order_by) |> MyApp.Repo.aggregate(:count, :id)
    
    items = query
      |> limit(^per_page)
      |> offset(^offset)
      |> MyApp.Repo.all()
    
    %{
      data:  items,
      meta: %{
        page:        page,
        per_page:    per_page,
        total:       total,
        total_pages: ceil(total / per_page),
        has_next:    page * per_page < total,
        has_prev:    page > 1
      }
    }
  end
  
  defp get_page(%{"page" => p}) when is_binary(p) do
    case Integer.parse(p) do
      {n, ""} when n > 0 -> n
      _                  -> @default_page
    end
  end
  defp get_page(_), do: @default_page
  
  defp get_per_page(%{"per_page" => p}) when is_binary(p) do
    case Integer.parse(p) do
      {n, ""} when n > 0 -> min(n, @max_per_page)
      _                  -> @default_per_page
    end
  end
  defp get_per_page(_), do: @default_per_page
end

# Usage in controller:
def index(conn, params) do
  result =
    Post
    |> where(status: :published)
    |> order_by([p], desc: p.inserted_at)
    |> MyAppWeb.API.Pagination.paginate(params)
  
  conn
  |> put_status(:ok)
  |> json(result)
end
```

---

## Step 327: Rate Limiting

```elixir
defmodule MyAppWeb.Plugs.RateLimiter do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    identifier = get_identifier(conn)
    
    case MyApp.RateLimit.check(identifier) do
      {:ok, remaining} ->
        conn
        |> put_resp_header("x-ratelimit-limit", "100")
        |> put_resp_header("x-ratelimit-remaining", to_string(remaining))
        
      {:error, retry_after} ->
        conn
        |> put_status(:too_many_requests)
        |> put_resp_header("retry-after", to_string(retry_after))
        |> json(%{error: %{code: "RATE_LIMIT_EXCEEDED", message: "Too many requests"}})
        |> halt()
    end
  end
  
  defp get_identifier(conn) do
    case conn.assigns[:current_user] do
      nil  -> "ip:#{conn.remote_ip |> :inet.ntoa() |> to_string()}"
      user -> "user:#{user.id}"
    end
  end
end

defmodule MyApp.RateLimit do
  @limit 100
  @window 60  # seconds
  
  def check(key) do
    full_key = "rate_limit:#{key}:#{window_key()}"
    
    {:ok, count} = Redix.command(:redis, ["INCR", full_key])
    
    if count == 1 do
      Redix.command(:redis, ["EXPIRE", full_key, @window])
    end
    
    if count > @limit do
      {:error, @window}
    else
      {:ok, @limit - count}
    end
  end
  
  defp window_key do
    div(System.os_time(:second), @window)
  end
end
```

---

## Step 328: API Versioning

```elixir
# URL versioning (recommended)
scope "/api/v1", MyAppWeb.API.V1 do
  resources "/users", UserController
end

scope "/api/v2", MyAppWeb.API.V2 do
  resources "/users", UserController  # breaking changes
end

# Header versioning
defmodule MyAppWeb.Plugs.APIVersion do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    version = get_req_header(conn, "api-version") |> List.first() || "2024-01-01"
    assign(conn, :api_version, version)
  end
end

# Content-type versioning
# Accept: application/vnd.myapp.v2+json

# Version deprecation header
defmodule MyAppWeb.Plugs.DeprecationHeader do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(%{request_path: "/api/v1/" <> _} = conn, _opts) do
    conn
    |> put_resp_header("deprecation", "true")
    |> put_resp_header("sunset", "Sat, 01 Jan 2025 00:00:00 GMT")
    |> put_resp_header("link", "</api/v2>; rel=\"successor-version\"")
  end
  def call(conn, _), do: conn
end
```

---

## Step 329: OpenAPI Documentation

```elixir
# mix.exs
{:open_api_spex, "~> 3.19"}

defmodule MyAppWeb.API.Spec do
  alias OpenApiSpex.{Info, OpenApi, Paths, Server}
  
  @behaviour OpenApiSpex.SpexProvider
  
  @impl OpenApiSpex.SpexProvider
  def spec do
    %OpenApi{
      info: %Info{
        title: "MyApp API",
        version: "1.0.0",
        description: "MyApp REST API documentation"
      },
      servers: [
        %Server{url: "https://api.myapp.com/v1"},
        %Server{url: "http://localhost:4000/api/v1"}
      ],
      paths: Paths.from_router(MyAppWeb.Router)
    }
    |> OpenApiSpex.resolve_schema_modules()
  end
end

# Controller with OpenAPI annotations
defmodule MyAppWeb.API.V1.UserController do
  use MyAppWeb, :controller
  use OpenApiSpex.ControllerSpecs
  
  operation :show,
    summary: "Get user",
    parameters: [
      id: [in: :path, description: "User ID", type: :integer, required: true]
    ],
    responses: [
      ok: {"User", "application/json", MyAppWeb.Schemas.UserResponse},
      not_found: {"Not Found", "application/json", MyAppWeb.Schemas.ErrorResponse}
    ]
  
  def show(conn, %{"id" => id}) do
    with {:ok, user} <- MyApp.Accounts.get_user(id) do
      render(conn, :show, user: user)
    end
  end
end

# Swagger UI route
forward "/api/swaggerui", OpenApiSpex.Plug.SwaggerUI,
  path: "/api/openapi",
  default_model_expand_depth: 3
```

---

## Step 330: API Testing

```elixir
defmodule MyAppWeb.API.V1.PostControllerTest do
  use MyAppWeb.ConnCase
  
  setup %{conn: conn} do
    user = user_fixture()
    token = generate_jwt(user)
    
    conn = conn
      |> put_req_header("accept", "application/json")
      |> put_req_header("content-type", "application/json")
      |> put_req_header("authorization", "Bearer #{token}")
    
    {:ok, conn: conn, user: user}
  end
  
  describe "GET /api/v1/posts" do
    test "returns list", %{conn: conn} do
      post_fixture()
      post_fixture()
      
      conn = get(conn, ~p"/api/v1/posts")
      
      assert %{"data" => posts, "meta" => meta} = json_response(conn, 200)
      assert length(posts) == 2
      assert meta["total"] == 2
    end
    
    test "paginates", %{conn: conn} do
      for _ <- 1..25, do: post_fixture()
      
      conn = get(conn, ~p"/api/v1/posts?per_page=10&page=2")
      
      assert %{"data" => posts, "meta" => %{"page" => 2}} = json_response(conn, 200)
      assert length(posts) == 10
    end
  end
  
  describe "POST /api/v1/posts" do
    test "creates post", %{conn: conn} do
      params = %{post: %{title: "Hello", body: "World"}}
      
      conn = post(conn, ~p"/api/v1/posts", params)
      
      assert %{"data" => %{"attributes" => %{"title" => "Hello"}}} = json_response(conn, 201)
    end
    
    test "returns 422 on invalid params", %{conn: conn} do
      conn = post(conn, ~p"/api/v1/posts", %{post: %{title: ""}})
      
      assert %{"error" => %{"code" => "VALIDATION_ERROR", "details" => details}} =
               json_response(conn, 422)
      
      assert details["title"] != []
    end
    
    test "returns 401 without token" do
      conn = build_conn()
      conn = post(conn, ~p"/api/v1/posts", %{post: %{title: "Hello", body: "World"}})
      
      assert json_response(conn, 401)
    end
  end
end
```

---

## สรุป Part 30

✅ **Step 321** - API design principles  
✅ **Step 322** - Router structure  
✅ **Step 323** - Controller pattern  
✅ **Step 324** - FallbackController  
✅ **Step 325** - JSON views  
✅ **Step 326** - Pagination  
✅ **Step 327** - Rate limiting  
✅ **Step 328** - API versioning  
✅ **Step 329** - OpenAPI docs  
✅ **Step 330** - API testing  

➡️ [Part 31: Docker และ Kubernetes](./part-31-docker-k8s.md)
