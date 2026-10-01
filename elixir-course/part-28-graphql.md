# Part 28: GraphQL ด้วย Absinthe (Steps 301-320)

## Step 301: GraphQL Overview

```
REST vs GraphQL:
- REST: หลาย endpoints, over/under-fetching
- GraphQL: single endpoint, client กำหนดว่าจะดึง field ไหน

Absinthe = Elixir GraphQL library
- Type-safe schema
- Resolver functions
- Subscriptions (real-time)
- DataLoader (N+1 prevention)
```

---

## Step 302: Setup

```elixir
# mix.exs
defp deps do
  [
    {:absinthe, "~> 1.7"},
    {:absinthe_plug, "~> 1.5"},
    {:absinthe_phoenix, "~> 2.0"},  # subscriptions
    {:dataloader, "~> 2.0"}
  ]
end

# router.ex
defmodule MyAppWeb.Router do
  use MyAppWeb, :router
  
  forward "/api/graphql", Absinthe.Plug,
    schema: MyAppWeb.Schema
  
  if Mix.env() == :dev do
    forward "/api/graphiql", Absinthe.Plug.GraphiQL,
      schema: MyAppWeb.Schema,
      socket: MyAppWeb.UserSocket
  end
end
```

---

## Step 303: Schema

```elixir
defmodule MyAppWeb.Schema do
  use Absinthe.Schema
  
  import_types Absinthe.Type.Custom
  import_types MyAppWeb.Schema.AccountTypes
  import_types MyAppWeb.Schema.BlogTypes
  
  query do
    import_fields :account_queries
    import_fields :blog_queries
  end
  
  mutation do
    import_fields :account_mutations
    import_fields :blog_mutations
  end
  
  subscription do
    import_fields :blog_subscriptions
  end
end
```

---

## Step 304: Types

```elixir
defmodule MyAppWeb.Schema.BlogTypes do
  use Absinthe.Schema.Notation
  
  object :post do
    field :id,         :id
    field :title,      :string
    field :body,       :string
    field :status,     :post_status
    field :slug,       :string
    field :views,      :integer
    field :inserted_at, :datetime
    
    field :author, :user do
      resolve &Resolvers.Blog.author/3
    end
    
    field :tags, list_of(:tag) do
      resolve &Resolvers.Blog.tags/3
    end
    
    field :comment_count, :integer do
      resolve fn post, _, _ ->
        {:ok, length(post.comments || [])}
      end
    end
  end
  
  enum :post_status do
    value :draft
    value :published
    value :archived
  end
  
  object :user do
    field :id,       :id
    field :name,     :string
    field :email,    :string
    field :bio,      :string
    field :avatar,   :string
    
    field :posts, list_of(:post) do
      arg :status, :post_status
      resolve &Resolvers.Blog.user_posts/3
    end
  end
  
  object :tag do
    field :id,   :id
    field :name, :string
  end
  
  # Input types for mutations
  input_object :post_input do
    field :title,  non_null(:string)
    field :body,   non_null(:string)
    field :status, :post_status
    field :tags,   list_of(:string)
  end
  
  # Pagination
  object :post_connection do
    field :items,       list_of(:post)
    field :total_count, :integer
    field :has_more,    :boolean
    field :next_cursor, :string
  end
end
```

---

## Step 305: Queries

```elixir
defmodule MyAppWeb.Schema.BlogTypes do
  use Absinthe.Schema.Notation
  
  object :blog_queries do
    field :posts, :post_connection do
      arg :first,  :integer, default_value: 20
      arg :cursor, :string
      arg :status, :post_status
      arg :tag,    :string
      
      resolve &Resolvers.Blog.list_posts/3
    end
    
    field :post, :post do
      arg :id,   :id
      arg :slug, :string
      
      resolve &Resolvers.Blog.get_post/3
    end
    
    field :search_posts, list_of(:post) do
      arg :query, non_null(:string)
      resolve &Resolvers.Blog.search/3
    end
  end
end

defmodule MyAppWeb.Resolvers.Blog do
  alias MyApp.Blog
  
  def list_posts(_parent, args, _resolution) do
    {:ok, Blog.list_posts(args)}
  end
  
  def get_post(_parent, %{id: id}, _resolution) do
    case Blog.get_post(id) do
      nil  -> {:error, "Post not found"}
      post -> {:ok, post}
    end
  end
  
  def get_post(_parent, %{slug: slug}, _resolution) do
    case Blog.get_post_by_slug(slug) do
      nil  -> {:error, "Post not found"}
      post -> {:ok, post}
    end
  end
  
  def author(post, _args, _resolution) do
    {:ok, post.author}  # preloaded by DataLoader
  end
  
  def user_posts(user, %{status: status}, _resolution) do
    {:ok, Blog.get_user_posts(user.id, status)}
  end
  
  def user_posts(user, _args, _resolution) do
    {:ok, Blog.get_user_posts(user.id)}
  end
  
  def search(_parent, %{query: query}, _resolution) do
    {:ok, Blog.search_posts(query)}
  end
end
```

---

## Step 306: Mutations

```elixir
defmodule MyAppWeb.Schema.BlogTypes do
  use Absinthe.Schema.Notation
  
  object :blog_mutations do
    field :create_post, :post do
      arg :input, non_null(:post_input)
      
      middleware MyAppWeb.Middleware.RequireAuth
      
      resolve &Resolvers.Blog.create_post/3
    end
    
    field :update_post, :post do
      arg :id,    non_null(:id)
      arg :input, non_null(:post_input)
      
      middleware MyAppWeb.Middleware.RequireAuth
      
      resolve &Resolvers.Blog.update_post/3
    end
    
    field :delete_post, :boolean do
      arg :id, non_null(:id)
      
      middleware MyAppWeb.Middleware.RequireAuth
      
      resolve &Resolvers.Blog.delete_post/3
    end
    
    field :publish_post, :post do
      arg :id, non_null(:id)
      
      middleware MyAppWeb.Middleware.RequireAuth
      
      resolve &Resolvers.Blog.publish_post/3
    end
  end
end

defmodule MyAppWeb.Resolvers.Blog do
  def create_post(_parent, %{input: input}, %{context: %{current_user: user}}) do
    attrs = Map.put(input, :user_id, user.id)
    
    case MyApp.Blog.create_post(attrs) do
      {:ok, post}    -> {:ok, post}
      {:error, cs}   -> {:error, format_errors(cs)}
    end
  end
  
  def update_post(_parent, %{id: id, input: input}, %{context: %{current_user: user}}) do
    with {:ok, post} <- get_own_post(id, user.id),
         {:ok, post} <- MyApp.Blog.update_post(post, input) do
      {:ok, post}
    end
  end
  
  def delete_post(_parent, %{id: id}, %{context: %{current_user: user}}) do
    with {:ok, post} <- get_own_post(id, user.id) do
      MyApp.Blog.delete_post(post)
      {:ok, true}
    end
  end
  
  defp get_own_post(id, user_id) do
    case MyApp.Blog.get_post(id) do
      nil  -> {:error, "Post not found"}
      %{user_id: ^user_id} = post -> {:ok, post}
      _    -> {:error, "Unauthorized"}
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

## Step 307: Authentication Middleware

```elixir
defmodule MyAppWeb.Middleware.RequireAuth do
  @behaviour Absinthe.Middleware
  
  def call(resolution, _config) do
    case resolution.context do
      %{current_user: _user} ->
        resolution
      _ ->
        resolution
        |> Absinthe.Resolution.put_result({:error, "Not authenticated"})
    end
  end
end

# Context plug
defmodule MyAppWeb.Context do
  @behaviour Plug
  
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    context = build_context(conn)
    Absinthe.Plug.put_options(conn, context: context)
  end
  
  defp build_context(conn) do
    with ["Bearer " <> token] <- get_req_header(conn, "authorization"),
         {:ok, user} <- MyApp.Auth.verify_token(token) do
      %{current_user: user}
    else
      _ -> %{}
    end
  end
end

# router.ex
pipeline :graphql do
  plug :accepts, ["json"]
  plug MyAppWeb.Context
end

scope "/api" do
  pipe_through :graphql
  forward "/graphql", Absinthe.Plug, schema: MyAppWeb.Schema
end
```

---

## Step 308: DataLoader (N+1 Prevention)

```elixir
defmodule MyAppWeb.Schema do
  use Absinthe.Schema
  
  def context(ctx) do
    loader =
      Dataloader.new()
      |> Dataloader.add_source(:blog, MyApp.Blog.data())
      |> Dataloader.add_source(:accounts, MyApp.Accounts.data())
    
    Map.put(ctx, :loader, loader)
  end
  
  def plugins do
    [Absinthe.Middleware.Dataloader] ++ Absinthe.Plugin.defaults()
  end
end

defmodule MyApp.Blog do
  def data do
    Dataloader.Ecto.new(Repo, query: &query/2)
  end
  
  defp query(Post, %{status: status}) do
    where(Post, [p], p.status == ^status)
  end
  defp query(queryable, _), do: queryable
end

# In types - use dataloader macro
defmodule MyAppWeb.Schema.BlogTypes do
  use Absinthe.Schema.Notation
  import Absinthe.Resolution.Helpers, only: [dataloader: 1, dataloader: 2]
  
  object :post do
    field :id,    :id
    field :title, :string
    
    # ✅ Batch loads all authors in ONE query
    field :author, :user do
      resolve dataloader(:accounts)
    end
    
    field :tags, list_of(:tag) do
      resolve dataloader(:blog)
    end
  end
end
```

---

## Step 309: Subscriptions

```elixir
defmodule MyAppWeb.Schema.BlogTypes do
  use Absinthe.Schema.Notation
  
  object :blog_subscriptions do
    field :post_created, :post do
      config fn _args, _res ->
        {:ok, topic: "posts:created"}
      end
    end
    
    field :post_updated, :post do
      arg :id, non_null(:id)
      
      config fn %{id: id}, _res ->
        {:ok, topic: "post:#{id}:updated"}
      end
    end
    
    field :comment_added, :comment do
      arg :post_id, non_null(:id)
      
      config fn %{post_id: post_id}, _res ->
        {:ok, topic: "post:#{post_id}:comments"}
      end
    end
  end
end

# Publish subscription event
defmodule MyApp.Blog do
  def create_post(attrs) do
    with {:ok, post} <- do_create_post(attrs) do
      # Notify subscribers
      Absinthe.Subscription.publish(
        MyAppWeb.Endpoint,
        post,
        post_created: "posts:created"
      )
      {:ok, post}
    end
  end
  
  def update_post(post, attrs) do
    with {:ok, updated} <- do_update_post(post, attrs) do
      Absinthe.Subscription.publish(
        MyAppWeb.Endpoint,
        updated,
        post_updated: "post:#{post.id}:updated"
      )
      {:ok, updated}
    end
  end
end

# GraphQL subscription query:
# subscription {
#   postCreated {
#     id title author { name }
#   }
# }
```

---

## Step 310: Error Handling

```elixir
defmodule MyAppWeb.Schema do
  use Absinthe.Schema
  
  # Custom error format
  def middleware(middleware, field, object) do
    middleware
    |> Absinthe.Schema.replace_default(field, object)
    |> add_error_handler()
  end
  
  defp add_error_handler(middleware) do
    middleware ++ [MyAppWeb.Middleware.ErrorHandler]
  end
end

defmodule MyAppWeb.Middleware.ErrorHandler do
  @behaviour Absinthe.Middleware
  
  def call(resolution, _config) do
    %{resolution | errors: Enum.map(resolution.errors, &transform/1)}
  end
  
  defp transform(%Ecto.Changeset{} = cs) do
    errors = Ecto.Changeset.traverse_errors(cs, fn {msg, opts} ->
      Enum.reduce(opts, msg, fn {key, val}, acc ->
        String.replace(acc, "%{#{key}}", to_string(val))
      end)
    end)
    
    %{message: "Validation failed", details: errors, code: "VALIDATION_ERROR"}
  end
  
  defp transform(message) when is_binary(message) do
    %{message: message}
  end
  
  defp transform(other), do: other
end

# Response format:
# {
#   "data": null,
#   "errors": [
#     {
#       "message": "Validation failed",
#       "details": {"title": ["can't be blank"]},
#       "code": "VALIDATION_ERROR"
#     }
#   ]
# }
```

---

## สรุป Part 28

✅ **Step 301** - GraphQL overview  
✅ **Step 302** - Absinthe setup  
✅ **Step 303** - Schema definition  
✅ **Step 304** - Type definitions  
✅ **Step 305** - Queries + resolvers  
✅ **Step 306** - Mutations  
✅ **Step 307** - Authentication middleware  
✅ **Step 308** - DataLoader (N+1)  
✅ **Step 309** - Subscriptions  
✅ **Step 310** - Error handling  

➡️ [Part 29: Background Jobs ด้วย Oban](./part-29-oban.md)
