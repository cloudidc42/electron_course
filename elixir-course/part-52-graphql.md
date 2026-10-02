# Part 52: GraphQL / Absinthe Production (Steps 571-590)

## Step 571: Absinthe Schema Design

```elixir
defmodule MyAppWeb.Schema do
  use Absinthe.Schema

  import_types MyAppWeb.Schema.Types.{UserType, ProductType, OrderType}
  import_types Absinthe.Type.Custom

  query do
    import_fields :user_queries
    import_fields :product_queries
    import_fields :order_queries
  end

  mutation do
    import_fields :user_mutations
    import_fields :product_mutations
  end

  subscription do
    import_fields :order_subscriptions
  end

  def plugins, do: [Absinthe.Middleware.Dataloader | Absinthe.Plugin.defaults()]

  def context(ctx) do
    loader = Dataloader.new()
      |> Dataloader.add_source(:db, Dataloader.Ecto.new(MyApp.Repo))
    
    Map.put(ctx, :loader, loader)
  end
end

# types/user_type.ex
defmodule MyAppWeb.Schema.Types.UserType do
  use Absinthe.Schema.Notation

  object :user do
    field :id,    :id
    field :name,  :string
    field :email, :string

    field :orders, list_of(:order) do
      resolve dataloader(:db)
    end

    field :order_count, :integer do
      resolve fn user, _, _ ->
        count = MyApp.Orders.count_for_user(user.id)
        {:ok, count}
      end
    end
  end

  object :user_queries do
    field :me, :user do
      middleware MyAppWeb.Middleware.RequireAuth
      resolve &MyAppWeb.Resolvers.Users.me/3
    end

    field :user, :user do
      arg :id, non_null(:id)
      resolve &MyAppWeb.Resolvers.Users.find/3
    end
  end
end
```

---

## Step 572: Resolvers

```elixir
defmodule MyAppWeb.Resolvers.Products do
  alias MyApp.{Repo, Products}

  def list(args, _info) do
    products = Products.list(
      limit:    Map.get(args, :limit, 20),
      offset:   Map.get(args, :offset, 0),
      category: Map.get(args, :category)
    )
    {:ok, products}
  end

  def find(%{id: id}, _info) do
    case Products.get(id) do
      nil     -> {:error, "Product not found"}
      product -> {:ok, product}
    end
  end

  def search(%{query: q}, _info) do
    {:ok, Products.search(q)}
  end

  def create(args, %{context: %{current_user: user}}) do
    case Products.create(Map.put(args, :created_by, user.id)) do
      {:ok, product}    -> {:ok, product}
      {:error, cs}      ->
        errors = format_errors(cs)
        {:error, errors}
    end
  end

  def update(%{id: id} = args, _info) do
    with {:ok, product} <- Products.get_with_check(id),
         {:ok, updated} <- Products.update(product, args) do
      {:ok, updated}
    end
  end

  defp format_errors(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## Step 573: Middleware

```elixir
defmodule MyAppWeb.Middleware.RequireAuth do
  @behaviour Absinthe.Middleware

  def call(resolution, _config) do
    case resolution.context do
      %{current_user: %{} = _user} ->
        resolution

      _ ->
        resolution
        |> Absinthe.Resolution.put_result({:error, "Unauthorized"})
    end
  end
end

defmodule MyAppWeb.Middleware.RequireRole do
  @behaviour Absinthe.Middleware

  def call(resolution, role) do
    case resolution.context do
      %{current_user: %{role: ^role}} ->
        resolution
      %{current_user: _} ->
        Absinthe.Resolution.put_result(resolution, {:error, "Forbidden: requires #{role} role"})
      _ ->
        Absinthe.Resolution.put_result(resolution, {:error, "Unauthorized"})
    end
  end
end

defmodule MyAppWeb.Middleware.ErrorHandler do
  @behaviour Absinthe.Middleware

  def call(resolution, _) do
    %{resolution |
      errors: Enum.map(resolution.errors, fn
        %Ecto.Changeset{} = cs -> format_changeset(cs)
        error -> error
      end)
    }
  end

  defp format_changeset(cs) do
    %{
      message: "Validation failed",
      details: Ecto.Changeset.traverse_errors(cs, fn {m, _} -> m end)
    }
  end
end

# Schema plugin to add error handler to all fields
def middleware(middleware, _field, _object) do
  middleware ++ [MyAppWeb.Middleware.ErrorHandler]
end
```

---

## Step 574: Subscriptions

```elixir
defmodule MyAppWeb.Schema.Types.OrderType do
  use Absinthe.Schema.Notation

  object :order_subscriptions do
    field :order_updated, :order do
      arg :id, non_null(:id)

      config fn %{id: order_id}, %{context: %{current_user: user}} ->
        {:ok, topic: "order:#{order_id}:user:#{user.id}"}
      end

      trigger [:update_order], topic: fn order ->
        "order:#{order.id}:user:#{order.user_id}"
      end

      resolve fn order, _, _ -> {:ok, order} end
    end

    field :cart_updated, :cart do
      config fn _, %{context: %{current_user: user}} ->
        {:ok, topic: "cart:user:#{user.id}"}
      end
    end
  end
end

# Publish subscription
Absinthe.Subscription.publish(
  MyAppWeb.Endpoint,
  updated_order,
  order_updated: "order:#{order_id}:user:#{user_id}"
)

# Phoenix endpoint
defmodule MyAppWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :my_app
  use Absinthe.Phoenix.Endpoint
  # ...
end
```

---

## Step 575: Pagination (Relay-style Cursor)

```elixir
defmodule MyAppWeb.Schema.Relay do
  def connection(node_type) do
    %{
      edge_type: %{
        name:   "#{node_type}Edge",
        fields: %{
          node:   %{type: node_type},
          cursor: %{type: :string}
        }
      },
      connection_type: %{
        name:   "#{node_type}Connection",
        fields: %{
          edges:    %{type: {:list, :edge}},
          page_info: %{type: :page_info}
        }
      }
    }
  end
end

defmodule MyApp.Pagination do
  import Ecto.Query

  def paginate(query, %{first: first} = args) do
    cursor = Map.get(args, :after)
    
    query = if cursor do
      from q in query, where: q.id > ^decode_cursor(cursor)
    else
      query
    end
    
    records = query
      |> limit(^(first + 1))
      |> MyApp.Repo.all()
    
    has_next = length(records) > first
    edges    = records |> Enum.take(first) |> Enum.map(fn r ->
      %{node: r, cursor: encode_cursor(r.id)}
    end)
    
    %{
      edges: edges,
      page_info: %{
        has_next_page:     has_next,
        has_previous_page: cursor != nil,
        start_cursor:      List.first(edges)[:cursor],
        end_cursor:        List.last(edges)[:cursor]
      }
    }
  end

  defp encode_cursor(id), do: Base.encode64("cursor:#{id}")
  defp decode_cursor(cursor) do
    cursor
    |> Base.decode64!()
    |> String.replace_prefix("cursor:", "")
    |> String.to_integer()
  end
end
```

---

## Step 576: DataLoader Batching

```elixir
defmodule MyAppWeb.Resolvers.BatchResolver do
  import Absinthe.Resolution.Helpers, only: [dataloader: 2]

  # Batch load using DataLoader (auto-batches N+1 queries)
  def load_user_orders(user, _args, %{context: %{loader: loader}}) do
    loader
    |> Dataloader.load(:db, {:many, MyApp.Order}, user_id: user.id)
    |> on_load(fn loader ->
      orders = Dataloader.get(loader, :db, {:many, MyApp.Order}, user_id: user.id)
      {:ok, orders}
    end)
  end

  # Custom data source with custom batch function
  defmodule ReviewCountSource do
    def data, do: Dataloader.KV.new(&fetch/2)

    defp fetch(_, product_ids) do
      counts = MyApp.Repo.all(
        from r in Review,
          where: r.product_id in ^Enum.to_list(product_ids),
          group_by: r.product_id,
          select: {r.product_id, count(r.id)}
      )
      |> Map.new()

      Map.new(product_ids, fn id -> {id, Map.get(counts, id, 0)} end)
    end
  end

  def review_count(product, _args, %{context: %{loader: loader}}) do
    loader
    |> Dataloader.load(ReviewCountSource, :review_count, product.id)
    |> on_load(fn loader ->
      count = Dataloader.get(loader, ReviewCountSource, :review_count, product.id)
      {:ok, count}
    end)
  end
end
```

---

## Step 577: Rate Limiting GraphQL

```elixir
defmodule MyAppWeb.Middleware.RateLimit do
  @behaviour Absinthe.Middleware
  @limit 100
  @window 60_000  # 1 minute

  def call(resolution, opts) do
    user_id = get_in(resolution.context, [:current_user, :id]) || :anonymous
    field   = resolution.definition.name
    key     = "graphql:#{user_id}:#{field}"
    cost    = Keyword.get(opts, :cost, 1)

    case check_rate(key, cost) do
      :ok    -> resolution
      :error ->
        Absinthe.Resolution.put_result(resolution, {:error, %{
          message: "Rate limit exceeded",
          code:    "RATE_LIMITED"
        }})
    end
  end

  defp check_rate(key, cost) do
    case Redix.command(:redix, ["INCRBY", key, cost]) do
      {:ok, count} when count <= @limit ->
        if count == cost do  # first request in window
          Redix.command(:redix, ["PEXPIRE", key, @window])
        end
        :ok
      {:ok, _} ->
        :error
    end
  end
end

# Apply to expensive fields:
# field :search_products, list_of(:product) do
#   middleware {MyAppWeb.Middleware.RateLimit, cost: 5}
#   resolve &search_products/3
# end
```

---

## Step 578: Complexity Analysis

```elixir
defmodule MyAppWeb.Schema do
  use Absinthe.Schema

  # Limit query complexity to prevent abuse
  def plugins, do: [
    Absinthe.Middleware.Dataloader,
    {Absinthe.Phase.Document.Complexity, max_complexity: 1000}
    | Absinthe.Plugin.defaults()
  ]
end

# Define complexity on fields
object :product do
  field :id, :id

  field :reviews, list_of(:review) do
    complexity fn args, child_complexity ->
      limit = Map.get(args, :limit, 10)
      limit * child_complexity  # scales with requested count
    end
    resolve &load_reviews/3
  end

  field :recommendations, list_of(:product) do
    complexity 50  # flat cost for expensive ML operation
    resolve &load_recommendations/3
  end
end
```

---

## Step 579: Apollo-compatible Error Format

```elixir
defmodule MyAppWeb.Schema.ErrorFormatter do
  def format_error(%{message: message, extensions: ext}) do
    %{message: message, extensions: ext}
  end

  def format_error(%Ecto.Changeset{} = cs) do
    errors = Ecto.Changeset.traverse_errors(cs, fn {msg, opts} ->
      Enum.reduce(opts, msg, fn {k, v}, acc ->
        String.replace(acc, "%{#{k}}", to_string(v))
      end)
    end)

    %{
      message:    "Validation failed",
      extensions: %{
        code:   "VALIDATION_FAILED",
        fields: errors
      }
    }
  end

  def format_error(error), do: %{message: to_string(error)}
end

# In router
plug Absinthe.Plug,
  schema: MyAppWeb.Schema,
  error_formatter: MyAppWeb.Schema.ErrorFormatter
```

---

## Step 580: GraphQL Testing

```elixir
defmodule MyAppWeb.Schema.ProductQueriesTest do
  use MyAppWeb.ConnCase, async: true

  import MyApp.Factory

  @list_products """
    query ListProducts($category: String) {
      products(category: $category) {
        id
        name
        price
      }
    }
  """

  test "lists products", %{conn: conn} do
    insert_list(3, :product)

    conn = post(conn, "/api/graphql", %{
      query:     @list_products,
      variables: %{}
    })

    assert %{"data" => %{"products" => products}} = json_response(conn, 200)
    assert length(products) == 3
  end

  test "filters by category", %{conn: conn} do
    insert(:product, category: "books")
    insert(:product, category: "electronics")

    conn = post(conn, "/api/graphql", %{
      query:     @list_products,
      variables: %{category: "books"}
    })

    assert %{"data" => %{"products" => [%{"name" => _}]}} = json_response(conn, 200)
  end

  test "requires authentication for mutations", %{conn: conn} do
    mutation = """
      mutation { createProduct(name: "Test", price: 9.99) { id } }
    """

    conn = post(conn, "/api/graphql", %{query: mutation})
    response = json_response(conn, 200)
    assert get_in(response, ["errors", Access.at(0), "message"]) == "Unauthorized"
  end
end
```

---

## สรุป Part 52

✅ **Step 571** - Schema design  
✅ **Step 572** - Resolvers  
✅ **Step 573** - Middleware  
✅ **Step 574** - Subscriptions  
✅ **Step 575** - Cursor pagination  
✅ **Step 576** - DataLoader batching  
✅ **Step 577** - Rate limiting  
✅ **Step 578** - Complexity analysis  
✅ **Step 579** - Error formatting  
✅ **Step 580** - Testing  

➡️ [Part 53: Kubernetes Advanced](./part-53-kubernetes.md)
