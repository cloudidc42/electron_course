# Part 73: GraphQL with Absinthe (Steps 791-810)

## Step 791: Absinthe Setup

```elixir
# mix.exs
{:absinthe, "~> 1.7"},
{:absinthe_plug, "~> 1.5"},
{:absinthe_phoenix, "~> 2.0"},
{:dataloader, "~> 2.0"}

# router.ex
scope "/api" do
  pipe_through :api

  forward "/graphql",
    Absinthe.Plug,
    schema: MyAppWeb.Schema

  if Mix.env() == :dev do
    forward "/graphiql",
      Absinthe.Plug.GraphiQL,
      schema: MyAppWeb.Schema,
      interface: :playground
  end
end

# lib/my_app_web/schema.ex
defmodule MyAppWeb.Schema do
  use Absinthe.Schema
  import_types MyAppWeb.Schema.Types
  import_types MyAppWeb.Schema.Queries
  import_types MyAppWeb.Schema.Mutations
  import_types MyAppWeb.Schema.Subscriptions

  query     do: import_fields :root_query
  mutation  do: import_fields :root_mutation
  subscription do: import_fields :root_subscription
end
```

---

## Step 792: Types

```elixir
defmodule MyAppWeb.Schema.Types do
  use Absinthe.Schema.Notation

  scalar :datetime do
    description "ISO8601 datetime"
    parse fn
      %Absinthe.Blueprint.Input.String{value: v} -> DateTime.from_iso8601(v) |> elem(1) |> then(&{:ok, &1})
      _ -> :error
    end
    serialize &DateTime.to_iso8601/1
  end

  object :user do
    field :id,         :id
    field :name,       :string
    field :email,      :string
    field :plan,       :string
    field :created_at, :datetime

    field :orders, list_of(:order) do
      resolve &MyAppWeb.Resolvers.User.orders/3
    end

    field :order_count, :integer do
      resolve fn user, _args, _ ->
        {:ok, length(user.orders || [])}
      end
    end
  end

  object :order do
    field :id,         :id
    field :status,     :string
    field :total,      :float
    field :created_at, :datetime

    field :user, :user do
      resolve dataloader(MyApp.Accounts)
    end

    field :items, list_of(:order_item) do
      resolve dataloader(MyApp.Orders)
    end
  end

  object :order_item do
    field :id,       :id
    field :quantity, :integer
    field :price,    :float

    field :product, :product do
      resolve dataloader(MyApp.Catalog)
    end
  end

  object :product do
    field :id,          :id
    field :name,        :string
    field :price,       :float
    field :description, :string
    field :in_stock,    :boolean
  end
end
```

---

## Step 793: Queries

```elixir
defmodule MyAppWeb.Schema.Queries do
  use Absinthe.Schema.Notation
  import Absinthe.Resolution.Helpers

  object :root_query do
    field :me, :user do
      resolve &MyAppWeb.Resolvers.User.me/3
    end

    field :user, :user do
      arg :id, non_null(:id)
      resolve &MyAppWeb.Resolvers.User.get/3
    end

    field :users, list_of(:user) do
      arg :page,     :integer, default_value: 1
      arg :per_page, :integer, default_value: 20
      arg :search,   :string
      resolve &MyAppWeb.Resolvers.User.list/3
    end

    field :products, list_of(:product) do
      arg :search,   :string
      arg :in_stock, :boolean
      arg :min_price, :float
      arg :max_price, :float
      resolve &MyAppWeb.Resolvers.Product.list/3
    end
  end
end

defmodule MyAppWeb.Resolvers.User do
  alias MyApp.{Accounts, Repo}

  def me(_root, _args, %{context: %{current_user: user}}) do
    {:ok, user}
  end
  def me(_root, _args, _info), do: {:error, "Not authenticated"}

  def get(_root, %{id: id}, _info) do
    case Accounts.get_user(id) do
      nil  -> {:error, "User not found"}
      user -> {:ok, user}
    end
  end

  def list(_root, args, _info) do
    users = Accounts.list_users(
      page:      Map.get(args, :page, 1),
      per_page:  Map.get(args, :per_page, 20),
      search:    Map.get(args, :search)
    )
    {:ok, users}
  end

  def orders(user, _args, %{context: %{loader: loader}}) do
    loader
    |> Dataloader.load(MyApp.Orders, :orders, user)
    |> on_load(fn loader ->
      {:ok, Dataloader.get(loader, MyApp.Orders, :orders, user)}
    end)
  end
end
```

---

## Step 794: Mutations

```elixir
defmodule MyAppWeb.Schema.Mutations do
  use Absinthe.Schema.Notation

  object :root_mutation do
    field :create_user, :user do
      arg :name,     non_null(:string)
      arg :email,    non_null(:string)
      arg :password, non_null(:string)
      resolve &MyAppWeb.Resolvers.User.create/3
    end

    field :update_user, :user do
      arg :id,   non_null(:id)
      arg :name, :string
      arg :plan, :string
      resolve &MyAppWeb.Resolvers.User.update/3
    end

    field :place_order, :order do
      arg :items, non_null(list_of(non_null(:order_item_input)))
      resolve &MyAppWeb.Resolvers.Order.place/3
    end
  end

  input_object :order_item_input do
    field :product_id, non_null(:id)
    field :quantity,   non_null(:integer)
  end
end

defmodule MyAppWeb.Resolvers.Order do
  def place(_root, %{items: items}, %{context: %{current_user: user}}) do
    case MyApp.Orders.place_order(user, items) do
      {:ok, order}       -> {:ok, order}
      {:error, :out_of_stock} -> {:error, "One or more items out of stock"}
      {:error, changeset} ->
        errors = Ecto.Changeset.traverse_errors(changeset, fn {msg, _} -> msg end)
        {:error, inspect(errors)}
    end
  end
  def place(_, _, _), do: {:error, "Authentication required"}
end
```

---

## Step 795: Subscriptions

```elixir
defmodule MyAppWeb.Schema.Subscriptions do
  use Absinthe.Schema.Notation

  object :root_subscription do
    field :order_updated, :order do
      arg :order_id, non_null(:id)

      config fn %{order_id: order_id}, %{context: %{current_user: user}} ->
        if MyApp.Orders.can_view?(user, order_id) do
          {:ok, topic: "order:#{order_id}"}
        else
          {:error, "Unauthorized"}
        end
      end

      trigger [:update_order_status],
        topic: fn order -> "order:#{order.id}" end

      resolve fn order, _, _ -> {:ok, order} end
    end

    field :new_message, :message do
      arg :conversation_id, non_null(:id)

      config fn %{conversation_id: id}, _ ->
        {:ok, topic: "conversation:#{id}"}
      end

      resolve fn message, _, _ -> {:ok, message} end
    end
  end
end

# Publish subscription events
defmodule MyApp.Orders do
  def update_order_status(order, new_status) do
    with {:ok, updated} <- do_update(order, new_status) do
      # Trigger subscription
      Absinthe.Subscription.publish(
        MyAppWeb.Endpoint,
        updated,
        order_updated: "order:#{order.id}"
      )
      {:ok, updated}
    end
  end
end
```

---

## Step 796: Dataloader

```elixir
defmodule MyApp.DataloaderSource do
  def data do
    Dataloader.Ecto.new(MyApp.Repo)
  end

  def context(ctx) do
    loader = Dataloader.new()
      |> Dataloader.add_source(MyApp.Accounts, data())
      |> Dataloader.add_source(MyApp.Orders, data())
      |> Dataloader.add_source(MyApp.Catalog, data())

    Map.put(ctx, :loader, loader)
  end

  def plugins do
    [Absinthe.Middleware.Dataloader] ++ Absinthe.Plugin.defaults()
  end
end

# In schema
def context(ctx), do: MyApp.DataloaderSource.context(ctx)
def plugins, do: MyApp.DataloaderSource.plugins()

# Usage in resolver
field :user, :user do
  resolve dataloader(MyApp.Accounts)
end

# Custom dataloader query
field :recent_orders, list_of(:order) do
  resolve fn product, _args, %{context: %{loader: loader}} ->
    loader
    |> Dataloader.load(MyApp.Orders, {:many, MyApp.Order},
        product_id: product.id,
        limit: 10,
        order_by: [desc: :inserted_at])
    |> on_load(fn loader ->
      orders = Dataloader.get(loader, MyApp.Orders, {:many, MyApp.Order},
        product_id: product.id)
      {:ok, orders}
    end)
  end
end
```

---

## Step 797: Authentication Middleware

```elixir
defmodule MyAppWeb.Schema.Middleware.Auth do
  @behaviour Absinthe.Middleware

  def call(resolution, _config) do
    case resolution.context do
      %{current_user: _user} -> resolution
      _ ->
        Absinthe.Resolution.put_result(resolution, {:error, "Authentication required"})
    end
  end
end

defmodule MyAppWeb.Schema.Middleware.AdminOnly do
  @behaviour Absinthe.Middleware

  def call(resolution, _config) do
    case resolution.context do
      %{current_user: %{role: :admin}} -> resolution
      _ ->
        Absinthe.Resolution.put_result(resolution, {:error, "Admin access required"})
    end
  end
end

# Apply to fields
field :admin_stats, :stats do
  middleware MyAppWeb.Schema.Middleware.AdminOnly
  resolve &MyAppWeb.Resolvers.Admin.stats/3
end

# Or apply globally to mutation fields
def middleware(middleware, _field, %{identifier: :mutation}) do
  [MyAppWeb.Schema.Middleware.Auth | middleware]
end
def middleware(middleware, _field, _object), do: middleware
```

---

## Step 798: N+1 Query Prevention

```elixir
# Problem: N+1 without dataloader
# Querying 100 users → 100 separate order queries

# Bad resolver:
def orders(user, _args, _info) do
  orders = MyApp.Repo.preload(user, :orders).orders
  {:ok, orders}
end

# Good: use Dataloader (batches all at once)
def orders(user, _args, %{context: %{loader: loader}}) do
  loader
  |> Dataloader.load(MyApp.Orders, :orders, user)
  |> on_load(fn loader ->
    {:ok, Dataloader.get(loader, MyApp.Orders, :orders, user)}
  end)
end

# Or: use absinthe_ecto batch
field :orders, list_of(:order) do
  resolve fn user, _, _ ->
    batch({MyApp.Orders, :orders_by_user_ids}, user.id, fn batch ->
      {:ok, Map.get(batch, user.id, [])}
    end)
  end
end

defmodule MyApp.Orders do
  def orders_by_user_ids(user_ids) do
    MyApp.Repo.all(
      from o in MyApp.Order,
        where: o.user_id in ^user_ids,
        order_by: [desc: o.inserted_at]
    )
    |> Enum.group_by(& &1.user_id)
  end
end
```

---

## Step 799: Complexity & Depth Limiting

```elixir
defmodule MyAppWeb.Schema do
  use Absinthe.Schema

  # Limit query depth
  def max_complexity, do: 300

  # Complexity calculation
  field :users, list_of(:user) do
    complexity fn args, child_complexity ->
      per_page = Map.get(args, :per_page, 20)
      per_page * child_complexity
    end
    resolve &MyAppWeb.Resolvers.User.list/3
  end
end

# In plug config
forward "/graphql",
  Absinthe.Plug,
  schema:             MyAppWeb.Schema,
  analyze_complexity: true,
  max_complexity:     300
```

---

## Step 800: Testing GraphQL

```elixir
defmodule MyAppWeb.GraphQLTest do
  use MyAppWeb.ConnCase

  @query_get_user """
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
    }
  }
  """

  @mutation_create_user """
  mutation CreateUser($name: String!, $email: String!, $password: String!) {
    createUser(name: $name, email: $email, password: $password) {
      id
      name
      email
    }
  }
  """

  test "get user by id" do
    user = insert(:user)

    conn = build_conn()
      |> put_req_header("authorization", "Bearer #{token_for(user)}")
      |> post("/api/graphql", %{
           query: @query_get_user,
           variables: %{id: user.id}
         })

    assert %{"data" => %{"user" => result}} = json_response(conn, 200)
    assert result["id"]    == to_string(user.id)
    assert result["name"]  == user.name
    assert result["email"] == user.email
  end

  test "create user mutation" do
    conn = post(build_conn(), "/api/graphql", %{
      query: @mutation_create_user,
      variables: %{
        name:     "Test User",
        email:    "test@example.com",
        password: "password123"
      }
    })

    assert %{"data" => %{"createUser" => user}} = json_response(conn, 200)
    assert user["email"] == "test@example.com"
  end

  test "returns error for unauthenticated user" do
    conn = post(build_conn(), "/api/graphql", %{
      query: "{ me { id } }"
    })

    assert %{"errors" => [%{"message" => "Authentication required"}]} =
             json_response(conn, 200)
  end
end
```

---

## สรุป Part 73

✅ **Step 791** - Absinthe setup  
✅ **Step 792** - Types  
✅ **Step 793** - Queries  
✅ **Step 794** - Mutations  
✅ **Step 795** - Subscriptions  
✅ **Step 796** - Dataloader  
✅ **Step 797** - Auth middleware  
✅ **Step 798** - N+1 prevention  
✅ **Step 799** - Complexity limiting  
✅ **Step 800** - Testing GraphQL  

➡️ [Part 74: Advanced Phoenix Patterns](./part-74-phoenix-advanced.md)
