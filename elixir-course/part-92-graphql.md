# Part 92: GraphQL API (Steps 981-990)

## Step 981: Absinthe Setup

```elixir
# mix.exs: {:absinthe, "~> 1.7"}, {:absinthe_plug, "~> 1.5"}, {:dataloader, "~> 2.0"}

defmodule MyAppWeb.Schema do
  use Absinthe.Schema
  import_types Absinthe.Type.Custom
  import_types MyAppWeb.Schema.Types

  query     do
    import_fields :product_queries
    import_fields :user_queries
    import_fields :order_queries
  end

  mutation do
    import_fields :product_mutations
    import_fields :auth_mutations
    import_fields :order_mutations
  end

  subscription do
    import_fields :order_subscriptions
  end

  def context(ctx) do
    loader = Dataloader.new() |> Dataloader.add_source(MyApp.Repo, Dataloader.Ecto.new(MyApp.Repo))
    Map.put(ctx, :loader, loader)
  end

  def plugins do
    [Absinthe.Middleware.Dataloader] ++ Absinthe.Plugin.defaults()
  end
end

# Router
defmodule MyAppWeb.Router do
  use MyAppWeb, :router

  pipeline :graphql do
    plug :fetch_session
    plug MyAppWeb.Plugs.APIAuth
  end

  scope "/api" do
    pipe_through :graphql
    forward "/graphql", Absinthe.Plug, schema: MyAppWeb.Schema
    forward "/graphiql", Absinthe.Plug.GraphiQL, schema: MyAppWeb.Schema
  end
end
```

---

## Step 982: Types and Queries

```elixir
defmodule MyAppWeb.Schema.Types do
  use Absinthe.Schema.Notation
  import Absinthe.Resolution.Helpers, only: [dataloader: 1]

  object :product do
    field :id,          :id
    field :name,        :string
    field :price,       :float
    field :description, :string
    field :stock,       :integer
    field :in_stock,    :boolean, resolve: fn product, _, _ ->
      {:ok, product.stock > 0}
    end

    field :category, :category, resolve: dataloader(MyApp.Repo)

    field :reviews, list_of(:review) do
      arg :limit, :integer, default_value: 5
      resolve &MyAppWeb.Resolvers.Products.list_reviews/3
    end
  end

  object :category do
    field :id,   :id
    field :name, :string
  end

  object :review do
    field :id,      :id
    field :rating,  :integer
    field :body,    :string
    field :author,  :user, resolve: dataloader(MyApp.Repo)
  end

  object :user do
    field :id,    :id
    field :name,  :string
    field :email, :string
  end

  input_object :product_input do
    field :name,        non_null(:string)
    field :price,       non_null(:float)
    field :description, :string
    field :stock,       :integer
    field :category_id, non_null(:id)
  end
end

defmodule MyAppWeb.Schema.ProductQueries do
  use Absinthe.Schema.Notation

  object :product_queries do
    field :product, :product do
      arg :id, non_null(:id)
      resolve &MyAppWeb.Resolvers.Products.get/3
    end

    field :products, list_of(:product) do
      arg :search,      :string
      arg :category_id, :id
      arg :page,        :integer, default_value: 1
      arg :per_page,    :integer, default_value: 20
      resolve &MyAppWeb.Resolvers.Products.list/3
    end
  end
end
```

---

## Step 983: Resolvers

```elixir
defmodule MyAppWeb.Resolvers.Products do
  import Ecto.Query

  def get(_parent, %{id: id}, _resolution) do
    case MyApp.Repo.get(Product, id) do
      nil     -> {:error, "Product not found"}
      product -> {:ok, product}
    end
  end

  def list(_parent, args, _resolution) do
    query = from p in Product, order_by: [asc: p.name]
    
    query = if search = args[:search] do
      tsquery = String.split(search) |> Enum.join(" & ")
      where(query, [p], fragment("search_vector @@ to_tsquery('english', ?)", ^tsquery))
    else
      query
    end

    query = if cat_id = args[:category_id] do
      where(query, [p], p.category_id == ^cat_id)
    else
      query
    end

    offset = (args.page - 1) * args.per_page
    
    {:ok, MyApp.Repo.all(from q in query, limit: ^args.per_page, offset: ^offset)}
  end

  def list_reviews(product, %{limit: limit}, _resolution) do
    {:ok, MyApp.Repo.all(
      from r in Review,
      where: r.product_id == ^product.id,
      order_by: [desc: r.inserted_at],
      limit: ^limit
    )}
  end

  def create(_parent, %{input: input}, %{context: %{current_user: user}}) do
    input = Map.put(input, :user_id, user.id)
    
    case MyApp.Catalog.create_product(input) do
      {:ok, product}    -> {:ok, product}
      {:error, cs}      -> {:error, format_changeset_errors(cs)}
    end
  end

  defp format_changeset_errors(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Enum.reduce(opts, msg, fn {k, v}, acc ->
        String.replace(acc, "%{#{k}}", to_string(v))
      end)
    end)
    |> Enum.map_join(", ", fn {field, errors} ->
      "#{field}: #{Enum.join(errors, ", ")}"
    end)
  end
end
```

---

## Step 984: Mutations

```elixir
defmodule MyAppWeb.Schema.AuthMutations do
  use Absinthe.Schema.Notation

  object :auth_mutations do
    field :login, :session do
      arg :email,    non_null(:string)
      arg :password, non_null(:string)
      resolve &MyAppWeb.Resolvers.Auth.login/3
    end

    field :register, :session do
      arg :input, non_null(:user_registration_input)
      resolve &MyAppWeb.Resolvers.Auth.register/3
    end

    field :refresh_token, :session do
      arg :refresh_token, non_null(:string)
      resolve &MyAppWeb.Resolvers.Auth.refresh/3
    end
  end

  object :session do
    field :access_token,  :string
    field :refresh_token, :string
    field :user,          :user
  end

  input_object :user_registration_input do
    field :name,     non_null(:string)
    field :email,    non_null(:string)
    field :password, non_null(:string)
  end
end

defmodule MyAppWeb.Resolvers.Auth do
  def login(_parent, %{email: email, password: password}, _resolution) do
    with {:ok, user}   <- MyApp.Auth.authenticate(email, password),
         {:ok, tokens} <- MyApp.Auth.JWT.generate_tokens(user) do
      {:ok, Map.put(tokens, :user, user)}
    else
      {:error, :invalid_credentials} -> {:error, "Invalid email or password"}
    end
  end

  def register(_parent, %{input: input}, _resolution) do
    with {:ok, user}   <- MyApp.Accounts.create_user(input),
         {:ok, tokens} <- MyApp.Auth.JWT.generate_tokens(user) do
      {:ok, Map.put(tokens, :user, user)}
    else
      {:error, changeset} -> {:error, format_errors(changeset)}
    end
  end
end
```

---

## Step 985: Authentication in GraphQL

```elixir
defmodule MyAppWeb.Schema.Middleware.Authenticate do
  @behaviour Absinthe.Middleware

  def call(resolution, opts) do
    case resolution.context[:current_user] do
      nil ->
        Absinthe.Resolution.put_result(resolution, {:error, "unauthenticated"})
      user ->
        if role = Keyword.get(opts, :role) do
          if MyApp.Auth.RBAC.has_role?(user, role) do
            resolution
          else
            Absinthe.Resolution.put_result(resolution, {:error, "forbidden"})
          end
        else
          resolution
        end
    end
  end
end

# Usage in schema
defmodule MyAppWeb.Schema.AdminQueries do
  use Absinthe.Schema.Notation
  alias MyAppWeb.Schema.Middleware

  object :admin_queries do
    field :all_users, list_of(:user) do
      middleware Middleware.Authenticate, role: :admin
      resolve &MyAppWeb.Resolvers.Admin.list_users/3
    end
  end
end

# Plug to inject current user into context
defmodule MyAppWeb.Plugs.AbsintheContext do
  @behaviour Plug
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    context = %{current_user: conn.assigns[:current_user]}
    Absinthe.Plug.put_options(conn, context: context)
  end
end
```

---

## Step 986: Dataloader N+1 Prevention

```elixir
defmodule MyApp.Dataloader do
  def data do
    Dataloader.Ecto.new(MyApp.Repo, query: &query/2)
  end

  defp query(Product, %{scope: :user_products, user_id: uid}) do
    from p in Product, where: p.user_id == ^uid
  end

  defp query(Review, %{approved: true}) do
    from r in Review, where: r.status == "approved", order_by: [desc: r.inserted_at]
  end

  defp query(queryable, _), do: queryable
end

# In schema
def context(ctx) do
  loader = Dataloader.new()
    |> Dataloader.add_source(:db, MyApp.Dataloader.data())
  Map.put(ctx, :loader, loader)
end

# In type
object :post do
  field :author, :user do
    resolve dataloader(:db)
  end

  field :comments, list_of(:comment) do
    resolve dataloader(:db, :comments, args: %{approved: true})
  end
end
```

---

## Step 987: Subscriptions

```elixir
defmodule MyAppWeb.Schema.OrderSubscriptions do
  use Absinthe.Schema.Notation

  object :order_subscriptions do
    field :order_updated, :order do
      arg :order_id, non_null(:id)
      
      config fn args, %{context: %{current_user: user}} ->
        {:ok, topic: "order:#{args.order_id}:user:#{user.id}"}
      end
    end
  end
end

# Publishing events
defmodule MyApp.Orders do
  def update_status(order, status) do
    with {:ok, updated} <- MyApp.Repo.update(
           Ecto.Changeset.change(order, status: status)) do
      Absinthe.Subscription.publish(
        MyAppWeb.Endpoint,
        updated,
        order_updated: "order:#{order.id}:user:#{order.user_id}"
      )
      {:ok, updated}
    end
  end
end

# In endpoint.ex
defmodule MyAppWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :my_app
  use Absinthe.Phoenix.Endpoint
  # ...
end
```

---

## Step 988: Pagination in GraphQL

```elixir
defmodule MyAppWeb.Schema.Pagination do
  use Absinthe.Schema.Notation

  object :page_info do
    field :has_next_page,     :boolean
    field :has_previous_page, :boolean
    field :start_cursor,      :string
    field :end_cursor,        :string
  end

  defmacro connection_type(node_type) do
    quote do
      object :"#{unquote(node_type)}_edge" do
        field :node,   unquote(node_type)
        field :cursor, :string
      end

      object :"#{unquote(node_type)}_connection" do
        field :edges,     list_of(:"#{unquote(node_type)}_edge")
        field :page_info, :page_info
        field :total,     :integer
      end
    end
  end
end

defmodule MyAppWeb.Resolvers.Pagination do
  def paginate(query, %{first: first, after: cursor} = args) do
    {id, _} = if cursor, do: Base.decode64!(cursor) |> Integer.parse(), else: {0, nil}
    
    items = MyApp.Repo.all(
      from(q in query, where: q.id > ^id, limit: ^(first + 1))
    )
    
    has_next = length(items) > first
    items    = Enum.take(items, first)
    
    edges = Enum.map(items, fn item ->
      %{node: item, cursor: Base.encode64(to_string(item.id))}
    end)

    %{
      edges:     edges,
      page_info: %{
        has_next_page:     has_next,
        has_previous_page: cursor != nil,
        start_cursor:      edges |> List.first() |> Map.get(:cursor),
        end_cursor:        edges |> List.last()  |> Map.get(:cursor)
      },
      total: MyApp.Repo.aggregate(query, :count)
    }
  end
end
```

---

## Step 989: Error Handling in GraphQL

```elixir
defmodule MyAppWeb.Schema.Middleware.HandleErrors do
  @behaviour Absinthe.Middleware

  def call(%{errors: []} = resolution, _opts), do: resolution

  def call(resolution, _opts) do
    errors = Enum.map(resolution.errors, fn
      %Ecto.Changeset{} = cs ->
        %{
          message: "Validation failed",
          extensions: %{
            code:   "VALIDATION_ERROR",
            fields: format_changeset_errors(cs)
          }
        }
      %{message: message, code: code} ->
        %{message: message, extensions: %{code: code}}
      message when is_binary(message) ->
        %{message: message, extensions: %{code: "UNKNOWN_ERROR"}}
      other ->
        %{message: inspect(other), extensions: %{code: "INTERNAL_ERROR"}}
    end)

    %{resolution | errors: errors}
  end

  defp format_changeset_errors(cs) do
    Ecto.Changeset.traverse_errors(cs, fn {msg, opts} ->
      Enum.reduce(opts, msg, fn {k, v}, acc ->
        String.replace(acc, "%{#{k}}", to_string(v))
      end)
    end)
  end
end

# Telemetry
defmodule MyAppWeb.Schema.Instrumentation do
  def before_send({conn, blueprint}) do
    start = System.monotonic_time(:millisecond)
    {Map.put(conn, :graphql_start, start), blueprint}
  end

  def after_send(conn, blueprint) do
    duration = System.monotonic_time(:millisecond) - conn.graphql_start
    :telemetry.execute([:graphql, :query], %{duration: duration}, %{
      operation: blueprint.name
    })
    conn
  end
end
```

---

## Step 990: GraphQL Complexity Limits

```elixir
defmodule MyAppWeb.Schema.Complexity do
  # Prevent DoS via deeply nested queries

  def complexity({:list_of, _}, args, child_complexity) do
    limit = Map.get(args, :limit, 20)
    limit * child_complexity
  end

  def complexity(_, _args, child_complexity) do
    child_complexity + 1
  end
end

# In schema
defmodule MyAppWeb.Schema do
  use Absinthe.Schema

  def complexity(blueprint, server_opts) do
    max_complexity = Keyword.get(server_opts, :max_complexity, 1000)
    
    case Absinthe.Blueprint.current_operation(blueprint) do
      %{complexity: complexity} when complexity > max_complexity ->
        {:error, "Query complexity #{complexity} exceeds maximum #{max_complexity}"}
      _ ->
        {:ok, blueprint}
    end
  end
end

# In Absinthe.Plug config
plug Absinthe.Plug,
  schema:         MyAppWeb.Schema,
  max_complexity: 500,
  analyze_complexity: true
```

---

## สรุป Part 92

✅ **Step 981** - Absinthe setup  
✅ **Step 982** - Types and queries  
✅ **Step 983** - Resolvers  
✅ **Step 984** - Mutations  
✅ **Step 985** - Authentication  
✅ **Step 986** - Dataloader N+1 prevention  
✅ **Step 987** - Subscriptions  
✅ **Step 988** - Pagination  
✅ **Step 989** - Error handling  
✅ **Step 990** - Complexity limits  

➡️ [Part 93: Analytics & Reporting](./part-93-analytics.md)
