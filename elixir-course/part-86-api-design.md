# Part 86: API Design Patterns (Steps 921-940)

## Step 921: RESTful API Best Practices

```elixir
defmodule MyAppWeb.V1.ProductsController do
  use MyAppWeb, :controller

  # GET /api/v1/products
  def index(conn, params) do
    filters = build_filters(params)
    page    = Pagination.paginate(Product, filters)
    
    conn
    |> put_resp_header("x-total-count",  to_string(page.total_count))
    |> put_resp_header("x-page",         to_string(page.page))
    |> put_resp_header("link",           build_link_header(conn, page))
    |> render(:index, products: page.entries, page: page)
  end

  # GET /api/v1/products/:id
  def show(conn, %{"id" => id}) do
    with {:ok, product} <- MyApp.Catalog.get_product(id) do
      render(conn, :show, product: product)
    end
  end

  # POST /api/v1/products
  def create(conn, %{"product" => attrs}) do
    with {:ok, product} <- MyApp.Catalog.create_product(attrs) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", ~p"/api/v1/products/#{product.id}")
      |> render(:show, product: product)
    end
  end

  # PATCH /api/v1/products/:id
  def update(conn, %{"id" => id, "product" => attrs}) do
    with {:ok, product} <- MyApp.Catalog.get_product(id),
         :ok            <- authorize(conn, :update, product),
         {:ok, updated} <- MyApp.Catalog.update_product(product, attrs) do
      render(conn, :show, product: updated)
    end
  end

  # DELETE /api/v1/products/:id
  def delete(conn, %{"id" => id}) do
    with {:ok, product} <- MyApp.Catalog.get_product(id),
         {:ok, _}       <- MyApp.Catalog.delete_product(product) do
      send_resp(conn, :no_content, "")
    end
  end

  defp build_link_header(conn, page) do
    base = Phoenix.Controller.current_url(conn)
    links = []
    links = if page.has_next, do: [~s(<#{base}?page=#{page.page + 1}>; rel="next") | links], else: links
    links = if page.has_prev, do: [~s(<#{base}?page=#{page.page - 1}>; rel="prev") | links], else: links
    Enum.join(links, ", ")
  end
end
```

---

## Step 922: JSON:API Format

```elixir
defmodule MyAppWeb.V1.ProductJSON do
  def index(%{products: products, page: page}) do
    %{
      data:  Enum.map(products, &resource/1),
      meta:  %{
        total_count:  page.total_count,
        total_pages:  page.total_pages,
        current_page: page.page,
        per_page:     page.per_page
      },
      links: %{
        self:  "/api/v1/products?page=#{page.page}",
        next:  (if page.has_next, do: "/api/v1/products?page=#{page.page + 1}"),
        prev:  (if page.has_prev, do: "/api/v1/products?page=#{page.page - 1}"),
        first: "/api/v1/products?page=1",
        last:  "/api/v1/products?page=#{page.total_pages}"
      }
    }
  end

  def show(%{product: product}) do
    %{data: resource(product)}
  end

  defp resource(product) do
    %{
      id:         to_string(product.id),
      type:       "products",
      attributes: %{
        name:        product.name,
        price:       product.price,
        stock:       product.stock,
        description: product.description,
        inserted_at: product.inserted_at
      },
      relationships: %{
        category: %{
          data: %{id: to_string(product.category_id), type: "categories"}
        }
      },
      links: %{
        self: "/api/v1/products/#{product.id}"
      }
    }
  end
end
```

---

## Step 923: OpenAPI Documentation

```elixir
# mix.exs: {:open_api_spex, "~> 3.21"}

defmodule MyAppWeb.Schemas do
  alias OpenApiSpex.Schema

  defmodule Product do
    require OpenApiSpex

    OpenApiSpex.schema(%{
      title:       "Product",
      description: "A product in the catalog",
      type:        :object,
      properties: %{
        id:          %Schema{type: :integer},
        name:        %Schema{type: :string, minLength: 1, maxLength: 200},
        price:       %Schema{type: :number, format: :float, minimum: 0},
        stock:       %Schema{type: :integer, minimum: 0},
        description: %Schema{type: :string},
        inserted_at: %Schema{type: :string, format: :"date-time"}
      },
      required: [:name, :price]
    })
  end

  defmodule ProductInput do
    require OpenApiSpex

    OpenApiSpex.schema(%{
      title: "ProductInput",
      type: :object,
      properties: %{
        name:        %Schema{type: :string},
        price:       %Schema{type: :number},
        description: %Schema{type: :string},
        category_id: %Schema{type: :integer}
      },
      required: [:name, :price]
    })
  end
end

defmodule MyAppWeb.V1.ProductsControllerSpec do
  use OpenApiSpex.ControllerSpecs

  alias MyAppWeb.Schemas

  operation :index,
    summary:    "List products",
    parameters: [
      page:     [in: :query, type: :integer, description: "Page number"],
      per_page: [in: :query, type: :integer, description: "Items per page"]
    ],
    responses: %{
      200 => {"Products list", "application/json", %OpenApiSpex.Schema{
        type:  :array,
        items: Schemas.Product
      }}
    }

  operation :create,
    summary:     "Create product",
    request_body: {"Product params", "application/json", Schemas.ProductInput, required: true},
    responses: %{
      201 => {"Created product", "application/json", Schemas.Product},
      422 => {"Validation error", "application/json", Schemas.ErrorResponse}
    }
end
```

---

## Step 924: Rate Limiting per Endpoint

```elixir
defmodule MyAppWeb.Plugs.APIRateLimit do
  import Plug.Conn

  @limits %{
    "POST /api/v1/auth/login" => {5, 60},   # 5 per minute
    "POST /api/v1/users"      => {10, 3600}, # 10 per hour
    "GET /api/v1/products"    => {100, 60},  # 100 per minute
    :default                  => {60, 60}    # 60 per minute
  }

  def init(opts), do: opts

  def call(conn, _opts) do
    key   = "#{conn.method} #{conn.request_path}"
    {limit, window} = Map.get(@limits, key, @limits[:default])

    ip = conn.remote_ip |> :inet.ntoa() |> to_string()
    user_id = conn.assigns[:current_user] && conn.assigns.current_user.id
    bucket_key = if user_id, do: "user:#{user_id}:#{key}", else: "ip:#{ip}:#{key}"

    case check_limit(bucket_key, limit, window) do
      {:ok, remaining} ->
        conn
        |> put_resp_header("x-ratelimit-limit",     to_string(limit))
        |> put_resp_header("x-ratelimit-remaining", to_string(remaining))

      {:error, :exceeded, reset_at} ->
        conn
        |> put_resp_header("retry-after", to_string(reset_at))
        |> send_resp(429, Jason.encode!(%{error: "rate_limit_exceeded"}))
        |> halt()
    end
  end

  defp check_limit(key, limit, window) do
    {:ok, count} = Redix.command(:redix, ["INCR", key])
    if count == 1, do: Redix.command(:redix, ["EXPIRE", key, window])
    
    remaining = max(0, limit - count)
    if count > limit do
      {:ok, ttl} = Redix.command(:redix, ["TTL", key])
      {:error, :exceeded, ttl}
    else
      {:ok, remaining}
    end
  end
end
```

---

## Step 925: API Authentication

```elixir
defmodule MyAppWeb.Plugs.APIAuth do
  import Plug.Conn
  import Phoenix.Controller

  def init(opts), do: opts

  def call(conn, _opts) do
    case get_req_header(conn, "authorization") do
      ["Bearer " <> token] -> authenticate_bearer(conn, token)
      ["ApiKey " <> key]   -> authenticate_api_key(conn, key)
      _                    -> unauthorized(conn)
    end
  end

  defp authenticate_bearer(conn, token) do
    case MyApp.Auth.verify_token(token) do
      {:ok, claims} ->
        user = MyApp.Accounts.get_user!(claims["sub"])
        assign(conn, :current_user, user)
      {:error, :expired} ->
        conn
        |> put_resp_header("www-authenticate", ~s(Bearer error="token_expired"))
        |> unauthorized()
      {:error, _} ->
        unauthorized(conn)
    end
  end

  defp authenticate_api_key(conn, key) do
    case MyApp.APIKeys.verify(key) do
      {:ok, api_key} ->
        MyApp.APIKeys.record_usage(api_key)
        assign(conn, :current_api_key, api_key)
      :error ->
        unauthorized(conn)
    end
  end

  defp unauthorized(conn) do
    conn
    |> put_status(401)
    |> json(%{error: "unauthorized"})
    |> halt()
  end
end
```

---

## Step 926: API Response Consistency

```elixir
defmodule MyAppWeb.APIHelpers do
  import Plug.Conn
  import Phoenix.Controller

  # Consistent success response
  def success(conn, data, opts \\ []) do
    status = Keyword.get(opts, :status, 200)
    meta   = Keyword.get(opts, :meta, %{})

    conn
    |> put_status(status)
    |> json(%{
      success: true,
      data:    data,
      meta:    meta
    })
  end

  # Consistent error response
  def error(conn, status, code, message, details \\ nil) do
    body = %{
      success: false,
      error: %{
        code:    code,
        message: message,
        details: details
      }
    }

    conn
    |> put_status(status)
    |> json(body)
  end

  # Paginated response
  def paginated(conn, %{entries: entries, total_count: total, page: page, per_page: per}) do
    conn
    |> put_resp_header("x-total-count",  to_string(total))
    |> put_resp_header("x-page",         to_string(page))
    |> put_resp_header("x-per-page",     to_string(per))
    |> put_resp_header("x-total-pages",  to_string(ceil(total / per)))
    |> json(%{
      success: true,
      data:    entries,
      meta: %{
        total_count:  total,
        page:         page,
        per_page:     per,
        total_pages:  ceil(total / per)
      }
    })
  end
end
```

---

## Step 927: Idempotency Keys

```elixir
defmodule MyAppWeb.Plugs.IdempotencyKey do
  import Plug.Conn

  @supported_methods ~w(POST PUT PATCH)

  def init(opts), do: opts

  def call(%{method: method} = conn, _) when method not in @supported_methods do
    conn
  end

  def call(conn, _opts) do
    case get_req_header(conn, "idempotency-key") do
      [key] -> handle_idempotency(conn, key)
      []    -> conn
    end
  end

  defp handle_idempotency(conn, key) do
    cache_key = "idempotency:#{key}"

    case Redix.command(:redix, ["GET", cache_key]) do
      {:ok, nil} ->
        # First request, proceed
        register_before_send(conn, fn response_conn ->
          # Cache the response for 24 hours
          body = response_conn.resp_body
          Redix.command(:redix, ["SETEX", cache_key, 86_400,
            Jason.encode!(%{status: response_conn.status, body: body})])
          response_conn
        end)

      {:ok, cached} ->
        # Duplicate request, return cached response
        %{"status" => status, "body" => body} = Jason.decode!(cached)
        
        conn
        |> put_resp_header("x-idempotency-replay", "true")
        |> put_resp_content_type("application/json")
        |> send_resp(status, body)
        |> halt()
    end
  end
end
```

---

## Step 928: API Deprecation

```elixir
defmodule MyAppWeb.Plugs.DeprecationWarning do
  import Plug.Conn

  @deprecated_endpoints %{
    "GET /api/v1/users/me"     => {"/api/v2/account", "2025-12-31"},
    "POST /api/v1/auth/login"  => {"/api/v2/auth/sessions", "2025-06-30"}
  }

  def init(opts), do: opts

  def call(conn, _opts) do
    key = "#{conn.method} #{conn.request_path}"

    case Map.get(@deprecated_endpoints, key) do
      {new_path, sunset_date} ->
        conn
        |> put_resp_header("deprecation", "true")
        |> put_resp_header("sunset",      sunset_date)
        |> put_resp_header("link", ~s(<#{new_path}>; rel="successor-version"))
      nil ->
        conn
    end
  end
end

# Version negotiation
defmodule MyAppWeb.Plugs.APIVersion do
  import Plug.Conn

  @supported_versions ["v1", "v2", "v3"]
  @latest_version     "v3"
  @default_version    "v2"

  def init(opts), do: opts

  def call(conn, _opts) do
    version = extract_version(conn)

    if version in @supported_versions do
      conn = assign(conn, :api_version, version)
      if version != @latest_version do
        put_resp_header(conn, "x-api-version-latest", @latest_version)
      else
        conn
      end
    else
      conn
      |> put_status(400)
      |> Phoenix.Controller.json(%{error: "unsupported_api_version"})
      |> halt()
    end
  end

  defp extract_version(conn) do
    case get_req_header(conn, "api-version") do
      [version] when version in @supported_versions -> version
      _ ->
        conn.request_path
        |> String.split("/")
        |> Enum.find(fn part -> String.starts_with?(part, "v") end)
        |> Kernel.||(@default_version)
    end
  end
end
```

---

## Step 929: Batch API

```elixir
defmodule MyAppWeb.V1.BatchController do
  use MyAppWeb, :controller

  @max_operations 20

  def process(conn, %{"operations" => ops}) when length(ops) > @max_operations do
    conn
    |> put_status(400)
    |> json(%{error: "too_many_operations", max: @max_operations})
  end

  def process(conn, %{"operations" => operations}) do
    results = operations
      |> Enum.with_index()
      |> Task.async_stream(fn {op, idx} ->
        result = execute_operation(conn, op)
        {idx, result}
      end, max_concurrency: 5, timeout: 30_000)
      |> Enum.map(fn
        {:ok, {idx, result}} -> {idx, result}
        {:exit, reason}      -> {:error, reason}
      end)
      |> Enum.sort_by(&elem(&1, 0))
      |> Enum.map(&elem(&1, 1))

    json(conn, %{results: results})
  end

  defp execute_operation(conn, %{"method" => method, "url" => url, "body" => body}) do
    fake_conn = conn
      |> Map.put(:method,      String.upcase(method))
      |> Map.put(:request_path, url)
      |> Map.put(:body_params,  body || %{})

    # Execute via router
    MyAppWeb.Router.call(fake_conn, MyAppWeb.Router.init([]))
    |> case do
      %{status: status, resp_body: resp_body} ->
        %{status: status, body: Jason.decode!(resp_body)}
    end
  end
end
```

---

## Step 930: Webhook Delivery

```elixir
defmodule MyApp.Webhooks do
  use Oban.Worker, queue: :webhooks, max_attempts: 5

  def deliver(endpoint_id, event_type, payload) do
    Oban.insert!(__MODULE__.new(%{
      endpoint_id: endpoint_id,
      event_type:  event_type,
      payload:     payload,
      attempt:     0
    }))
  end

  def perform(%Oban.Job{args: args, attempt: attempt}) do
    endpoint = MyApp.Repo.get!(WebhookEndpoint, args["endpoint_id"])
    
    signature = sign_payload(endpoint.secret, args["payload"])
    
    case Req.post(endpoint.url,
           json:    args["payload"],
           headers: [
             {"x-webhook-signature", signature},
             {"x-webhook-event",     args["event_type"]},
             {"x-webhook-id",        args["webhook_id"] || Ecto.UUID.generate()}
           ],
           receive_timeout: 10_000
         ) do
      {:ok, %{status: status}} when status in 200..299 ->
        record_delivery(endpoint, :success)
        :ok
      {:ok, %{status: status}} ->
        record_delivery(endpoint, :failed, "HTTP #{status}")
        {:error, "endpoint returned #{status}"}
      {:error, reason} ->
        backoff = (:math.pow(2, attempt) * 1000) |> round()
        {:snooze, div(backoff, 1000)}
    end
  end

  defp sign_payload(secret, payload) do
    body = Jason.encode!(payload)
    :crypto.mac(:hmac, :sha256, secret, body) |> Base.encode16(case: :lower)
  end

  defp record_delivery(endpoint, status, error \\ nil) do
    MyApp.Repo.insert!(%WebhookDelivery{
      endpoint_id: endpoint.id,
      status:      status,
      error:       error,
      delivered_at: DateTime.utc_now()
    })
  end
end
```

---

## สรุป Part 86

✅ **Step 921** - RESTful API best practices  
✅ **Step 922** - JSON:API format  
✅ **Step 923** - OpenAPI documentation  
✅ **Step 924** - Rate limiting per endpoint  
✅ **Step 925** - API authentication  
✅ **Step 926** - Response consistency  
✅ **Step 927** - Idempotency keys  
✅ **Step 928** - API deprecation  
✅ **Step 929** - Batch API  
✅ **Step 930** - Webhook delivery  

➡️ [Part 87: File Handling & Storage](./part-87-file-storage.md)
