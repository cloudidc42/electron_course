# Part 24: Ecto Advanced (Steps 261-280)

## Step 261: Migrations

```elixir
# สร้าง migration
# mix ecto.gen.migration create_users

defmodule MyApp.Repo.Migrations.CreateUsers do
  use Ecto.Migration
  
  def change do
    create table(:users) do
      add :name,            :string, null: false
      add :email,           :string, null: false
      add :hashed_password, :string
      add :role,            :string, default: "user"
      add :confirmed_at,    :naive_datetime
      add :avatar_url,      :string
      
      timestamps()
    end
    
    create unique_index(:users, [:email])
    create index(:users, [:role])
  end
end

# Foreign key
defmodule MyApp.Repo.Migrations.CreatePosts do
  use Ecto.Migration
  
  def change do
    create table(:posts) do
      add :title,     :string, null: false
      add :content,   :text
      add :status,    :string, default: "draft"
      add :user_id,   references(:users, on_delete: :cascade), null: false
      
      timestamps()
    end
    
    create index(:posts, [:user_id])
    create index(:posts, [:status])
    
    # Full-text search index (PostgreSQL)
    execute(
      "CREATE INDEX posts_search_idx ON posts USING GIN(to_tsvector('english', title || ' ' || coalesce(content, '')))",
      "DROP INDEX posts_search_idx"
    )
  end
end

# Add column migration
defmodule MyApp.Repo.Migrations.AddAgeToUsers do
  use Ecto.Migration
  
  def change do
    alter table(:users) do
      add :age,         :integer
      add :bio,         :text
      modify :name, :string, size: 200  # change column
    end
  end
end
```

---

## Step 262: Associations

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  
  schema "users" do
    field :name,  :string
    field :email, :string
    
    has_many :posts,    MyApp.Blog.Post
    has_many :comments, MyApp.Blog.Comment
    has_one  :profile,  MyApp.Accounts.Profile
    
    # through association
    has_many :liked_posts, through: [:likes, :post]
    
    timestamps()
  end
end

defmodule MyApp.Blog.Post do
  use Ecto.Schema
  
  schema "posts" do
    field :title,   :string
    field :content, :text
    field :status,  Ecto.Enum, values: [:draft, :published, :archived]
    
    belongs_to :user, MyApp.Accounts.User
    
    has_many :comments, MyApp.Blog.Comment
    has_many :tags, through: [:post_tags, :tag]
    
    many_to_many :categories, MyApp.Blog.Category,
      join_through: "post_categories"
    
    timestamps()
  end
end

defmodule MyApp.Blog.Tag do
  use Ecto.Schema
  
  schema "tags" do
    field :name, :string
    
    many_to_many :posts, MyApp.Blog.Post,
      join_through: MyApp.Blog.PostTag,
      on_replace: :delete
    
    timestamps()
  end
end
```

---

## Step 263: Preloading

```elixir
import Ecto.Query
alias MyApp.{Repo, Blog.Post}

# Simple preload
posts = Post |> Repo.all() |> Repo.preload(:user)

# Multiple preloads
posts = Post
  |> Repo.all()
  |> Repo.preload([:user, :comments, :tags])

# Nested preload
posts = Post
  |> Repo.all()
  |> Repo.preload([comments: :user, user: :profile])

# Preload in query (more efficient)
from(p in Post,
  preload: [:user, comments: :user]
)
|> Repo.all()

# Preload with conditions
from(p in Post,
  join: u in assoc(p, :user),
  where: u.role == "admin",
  preload: [user: u, comments: ^from(c in Comment, order_by: c.inserted_at)]
)
|> Repo.all()

# Preload single record
post = Repo.get!(Post, 1)
post = Repo.preload(post, [:user, :comments])
```

---

## Step 264: Changesets ขั้นสูง

```elixir
defmodule MyApp.Blog.Post do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "posts" do
    field :title,       :string
    field :content,     :text
    field :slug,        :string
    field :status,      Ecto.Enum, values: [:draft, :published]
    field :published_at, :naive_datetime
    
    belongs_to :user, MyApp.Accounts.User
    
    timestamps()
  end
  
  def changeset(post, attrs) do
    post
    |> cast(attrs, [:title, :content, :status])
    |> validate_required([:title, :content])
    |> validate_length(:title, min: 5, max: 200)
    |> validate_length(:content, min: 50)
    |> auto_generate_slug()
    |> set_published_at()
    |> unique_constraint(:slug)
  end
  
  def publish_changeset(post) do
    post
    |> change(status: :published, published_at: NaiveDateTime.utc_now())
    |> validate_required([:content])
    |> validate_length(:content, min: 100)
  end
  
  defp auto_generate_slug(changeset) do
    case get_change(changeset, :title) do
      nil   -> changeset
      title -> put_change(changeset, :slug, slugify(title))
    end
  end
  
  defp set_published_at(changeset) do
    case get_change(changeset, :status) do
      :published when is_nil(get_field(changeset, :published_at)) ->
        put_change(changeset, :published_at, NaiveDateTime.utc_now())
      _ ->
        changeset
    end
  end
  
  defp slugify(str) do
    str
    |> String.downcase()
    |> String.replace(~r/[^a-z0-9\s-]/, "")
    |> String.replace(~r/\s+/, "-")
    |> String.trim("-")
  end
end
```

---

## Step 265: Multi-table Operations

```elixir
defmodule MyApp.Accounts do
  alias Ecto.Multi
  
  def register_user_with_profile(user_attrs, profile_attrs) do
    Multi.new()
    |> Multi.insert(:user, User.registration_changeset(%User{}, user_attrs))
    |> Multi.insert(:profile, fn %{user: user} ->
         Profile.changeset(%Profile{user_id: user.id}, profile_attrs)
       end)
    |> Multi.insert(:welcome_notification, fn %{user: user} ->
         Notification.changeset(%Notification{}, %{
           user_id: user.id,
           type: :welcome,
           message: "Welcome to the platform!"
         })
       end)
    |> Repo.transaction()
    |> case do
         {:ok, %{user: user}} -> {:ok, user}
         {:error, :user, changeset, _} -> {:error, changeset}
         {:error, _operation, reason, _} -> {:error, reason}
       end
  end
  
  def transfer_credits(from_user_id, to_user_id, amount) do
    Multi.new()
    |> Multi.one(:from_user, fn _ ->
         from(u in User, where: u.id == ^from_user_id, lock: "FOR UPDATE")
       end)
    |> Multi.one(:to_user, fn _ ->
         from(u in User, where: u.id == ^to_user_id, lock: "FOR UPDATE")
       end)
    |> Multi.run(:validate, fn _repo, %{from_user: from, to_user: to} ->
         cond do
           is_nil(from) -> {:error, :from_not_found}
           is_nil(to)   -> {:error, :to_not_found}
           from.credits < amount -> {:error, :insufficient_credits}
           true -> {:ok, :valid}
         end
       end)
    |> Multi.update(:deduct, fn %{from_user: from} ->
         User.changeset(from, %{credits: from.credits - amount})
       end)
    |> Multi.update(:add, fn %{to_user: to} ->
         User.changeset(to, %{credits: to.credits + amount})
       end)
    |> Multi.insert(:log, fn %{from_user: from, to_user: to} ->
         CreditLog.changeset(%CreditLog{}, %{
           from_user_id: from.id,
           to_user_id: to.id,
           amount: amount
         })
       end)
    |> Repo.transaction()
  end
end
```

---

## Step 266: Dynamic Queries

```elixir
defmodule MyApp.Blog.PostQuery do
  import Ecto.Query
  alias MyApp.Blog.Post
  
  def base_query do
    from p in Post, as: :post
  end
  
  def filter(query, filters) do
    Enum.reduce(filters, query, fn {key, value}, q ->
      apply_filter(q, key, value)
    end)
  end
  
  defp apply_filter(query, :status, status) do
    where(query, [post: p], p.status == ^status)
  end
  
  defp apply_filter(query, :search, search_term) do
    term = "%#{search_term}%"
    where(query, [post: p], ilike(p.title, ^term) or ilike(p.content, ^term))
  end
  
  defp apply_filter(query, :user_id, user_id) do
    where(query, [post: p], p.user_id == ^user_id)
  end
  
  defp apply_filter(query, :tags, tags) when is_list(tags) do
    query
    |> join(:inner, [post: p], t in assoc(p, :tags), as: :tag)
    |> where([tag: t], t.name in ^tags)
    |> distinct(true)
  end
  
  defp apply_filter(query, :inserted_after, date) do
    where(query, [post: p], p.inserted_at >= ^date)
  end
  
  defp apply_filter(query, _key, _value), do: query
  
  def sort(query, field, direction \\ :asc) do
    order_by(query, [post: p], [{^direction, field(p, ^field)}])
  end
  
  def paginate(query, page, per_page \\ 10) do
    offset = (page - 1) * per_page
    query |> limit(^per_page) |> offset(^offset)
  end
end

# Usage
posts = PostQuery.base_query()
  |> PostQuery.filter(status: :published, search: "elixir", tags: ["beginner"])
  |> PostQuery.sort(:inserted_at, :desc)
  |> PostQuery.paginate(1, 20)
  |> MyApp.Repo.all()
```

---

## Step 267: Embedded Schemas

```elixir
defmodule MyApp.Orders.LineItem do
  use Ecto.Schema
  import Ecto.Changeset
  
  embedded_schema do
    field :product_id, :integer
    field :product_name, :string
    field :quantity, :integer
    field :unit_price, :decimal
    field :discount, :decimal, default: Decimal.new("0")
  end
  
  def changeset(line_item, attrs) do
    line_item
    |> cast(attrs, [:product_id, :product_name, :quantity, :unit_price, :discount])
    |> validate_required([:product_id, :quantity, :unit_price])
    |> validate_number(:quantity, greater_than: 0)
    |> validate_number(:unit_price, greater_than: 0)
  end
  
  def total(%__MODULE__{quantity: qty, unit_price: price, discount: discount}) do
    Decimal.mult(qty, price) |> Decimal.sub(discount)
  end
end

defmodule MyApp.Orders.Order do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "orders" do
    field :status, Ecto.Enum, values: [:pending, :confirmed, :shipped, :delivered]
    
    embeds_many :line_items, MyApp.Orders.LineItem, on_replace: :delete
    
    belongs_to :user, MyApp.Accounts.User
    
    timestamps()
  end
  
  def changeset(order, attrs) do
    order
    |> cast(attrs, [:status])
    |> cast_embed(:line_items, required: true)
    |> validate_required([:status])
    |> validate_has_items()
  end
  
  defp validate_has_items(changeset) do
    case get_field(changeset, :line_items) do
      [] -> add_error(changeset, :line_items, "must have at least one item")
      nil -> add_error(changeset, :line_items, "must have at least one item")
      _ -> changeset
    end
  end
  
  def total(%__MODULE__{line_items: items}) do
    Enum.reduce(items, Decimal.new("0"), fn item, acc ->
      Decimal.add(acc, MyApp.Orders.LineItem.total(item))
    end)
  end
end
```

---

## Step 268: Custom Types

```elixir
defmodule MyApp.Types.Money do
  @moduledoc "Stores money as integer (cents) in DB, decimal in Elixir"
  
  use Ecto.Type
  
  def type, do: :integer
  
  def cast(value) when is_integer(value) do
    {:ok, Decimal.new(value) |> Decimal.div(100)}
  end
  
  def cast(%Decimal{} = value), do: {:ok, value}
  
  def cast(value) when is_binary(value) do
    case Decimal.parse(value) do
      {decimal, ""} -> {:ok, decimal}
      _             -> :error
    end
  end
  
  def cast(_), do: :error
  
  def load(value) when is_integer(value) do
    {:ok, Decimal.new(value) |> Decimal.div(100)}
  end
  
  def dump(%Decimal{} = value) do
    cents = value |> Decimal.mult(100) |> Decimal.to_integer()
    {:ok, cents}
  end
  def dump(value) when is_integer(value), do: {:ok, value}
  def dump(_), do: :error
end

# Usage in schema
defmodule MyApp.Products.Product do
  use Ecto.Schema
  
  schema "products" do
    field :name,     :string
    field :price,    MyApp.Types.Money  # stored as integer, returned as Decimal
    field :currency, :string, default: "USD"
    
    timestamps()
  end
end
```

---

## Step 269: Repo Patterns

```elixir
defmodule MyApp.Repo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres
  
  # Custom aggregation helpers
  def count(queryable) do
    queryable |> aggregate(:count, :id) |> one!()
  end
  
  def sum(queryable, field) do
    queryable |> aggregate(:sum, field) |> one!()
  end
  
  def avg(queryable, field) do
    queryable |> aggregate(:avg, field) |> one!()
  end
  
  # Pagination helper
  def paginate(queryable, page, per_page \\ 20) do
    offset = (page - 1) * per_page
    
    total   = aggregate(queryable, :count, :id)
    results = queryable |> limit(^per_page) |> offset(^offset) |> all()
    
    %{
      data:        results,
      page:        page,
      per_page:    per_page,
      total:       total,
      total_pages: ceil(total / per_page),
      has_next:    page * per_page < total,
      has_prev:    page > 1
    }
  end
end

# Multi-tenancy pattern
defmodule MyApp.TenantRepo do
  def all(queryable, tenant_id) do
    queryable
    |> where(tenant_id: ^tenant_id)
    |> MyApp.Repo.all()
  end
end
```

---

## Step 270: Complete Blog Context

```elixir
defmodule MyApp.Blog do
  import Ecto.Query
  alias MyApp.{Repo, Blog.{Post, Comment, Tag}}
  
  # Posts
  
  def create_post(user, attrs) do
    %Post{user_id: user.id}
    |> Post.changeset(attrs)
    |> Repo.insert()
  end
  
  def publish_post(%Post{status: :draft} = post) do
    post
    |> Post.publish_changeset()
    |> Repo.update()
  end
  
  def get_published_posts(opts \\ []) do
    limit = Keyword.get(opts, :limit, 10)
    page  = Keyword.get(opts, :page, 1)
    
    Post
    |> where(status: :published)
    |> order_by([p], desc: p.published_at)
    |> preload([:user, :tags])
    |> Repo.paginate(page, limit)
  end
  
  def search_posts(query) do
    from(p in Post,
      where: fragment(
        "to_tsvector('english', ?) @@ plainto_tsquery('english', ?)",
        p.title, ^query
      ),
      order_by: fragment(
        "ts_rank(to_tsvector('english', ?), plainto_tsquery('english', ?)) DESC",
        p.title, ^query
      )
    )
    |> where(status: :published)
    |> preload(:user)
    |> Repo.all()
  end
  
  # Tags
  
  def create_or_find_tag(name) do
    case Repo.get_by(Tag, name: name) do
      nil -> %Tag{} |> Tag.changeset(%{name: name}) |> Repo.insert()
      tag -> {:ok, tag}
    end
  end
  
  def add_tags_to_post(post, tag_names) do
    tags = Enum.map(tag_names, fn name ->
      {:ok, tag} = create_or_find_tag(name)
      tag
    end)
    
    post
    |> Repo.preload(:tags)
    |> Ecto.Changeset.change()
    |> Ecto.Changeset.put_assoc(:tags, tags)
    |> Repo.update()
  end
  
  # Comments
  
  def add_comment(post, user, body) do
    %Comment{}
    |> Comment.changeset(%{
         post_id: post.id,
         user_id: user.id,
         body: body
       })
    |> Repo.insert()
  end
  
  def get_post_with_comments(slug) do
    case Repo.get_by(Post, slug: slug, status: :published) do
      nil  -> {:error, :not_found}
      post ->
        post = Repo.preload(post, [
          :user,
          :tags,
          comments: [user: [], replies: :user]
        ])
        {:ok, post}
    end
  end
end
```

---

## สรุป Part 24

✅ **Step 261** - Migrations  
✅ **Step 262** - Associations  
✅ **Step 263** - Preloading  
✅ **Step 264** - Advanced changesets  
✅ **Step 265** - Multi-table operations + Ecto.Multi  
✅ **Step 266** - Dynamic queries  
✅ **Step 267** - Embedded schemas  
✅ **Step 268** - Custom types  
✅ **Step 269** - Repo patterns  
✅ **Step 270** - Complete Blog context  

➡️ [Part 25: Authentication และ Authorization](./part-25-auth.md)
