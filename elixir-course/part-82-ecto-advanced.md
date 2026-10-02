# Part 82: Advanced Ecto Patterns (Steps 881-900)

## Step 881: Complex Queries

```elixir
defmodule MyApp.Repo.Queries do
  import Ecto.Query

  # Complex multi-join query
  def orders_with_stats(customer_id) do
    from o in Order,
      join: i in assoc(o, :items),
      join: p in assoc(i, :product),
      left_join: s in Shipment, on: s.order_id == o.id,
      where: o.customer_id == ^customer_id,
      group_by: [o.id, o.status, o.inserted_at, s.tracking_number],
      select: %{
        id:              o.id,
        status:          o.status,
        item_count:      count(i.id),
        total_quantity:  sum(i.quantity),
        tracking_number: s.tracking_number,
        ordered_at:      o.inserted_at
      },
      order_by: [desc: o.inserted_at]
  end

  # Window functions
  def orders_with_running_total(customer_id) do
    from o in Order,
      where: o.customer_id == ^customer_id,
      select: %{
        id:            o.id,
        total:         o.total,
        running_total: fragment(
          "SUM(?) OVER (ORDER BY ? ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)",
          o.total,
          o.inserted_at
        )
      },
      order_by: o.inserted_at
  end

  # Lateral join for top N per group
  def top_products_per_category(n \\ 3) do
    from c in Category,
      join: p in fragment(
        """
        LATERAL (
          SELECT * FROM products p2
          WHERE p2.category_id = ?
          ORDER BY p2.sales_count DESC
          LIMIT ?
        )
        """,
        c.id, ^n
      ), on: true,
      select: {c.name, p}
  end
end
```

---

## Step 882: Dynamic Queries

```elixir
defmodule MyApp.ProductSearch do
  import Ecto.Query

  def search(filters) do
    Product
    |> apply_filters(filters)
    |> apply_sorting(filters)
    |> apply_pagination(filters)
  end

  defp apply_filters(query, filters) do
    Enum.reduce(filters, query, fn
      {:name, name}, q ->
        from p in q, where: ilike(p.name, ^"%#{name}%")

      {:category_id, id}, q ->
        from p in q, where: p.category_id == ^id

      {:min_price, min}, q ->
        from p in q, where: p.price >= ^min

      {:max_price, max}, q ->
        from p in q, where: p.price <= ^max

      {:in_stock, true}, q ->
        from p in q, where: p.stock > 0

      {:tags, tags}, q when is_list(tags) ->
        from p in q, where: fragment("? && ?", p.tags, ^tags)

      _, q -> q
    end)
  end

  defp apply_sorting(query, %{sort_by: :price, sort_dir: :desc}) do
    from p in query, order_by: [desc: p.price]
  end
  defp apply_sorting(query, %{sort_by: :price}) do
    from p in query, order_by: [asc: p.price]
  end
  defp apply_sorting(query, _), do: from(p in query, order_by: [desc: p.inserted_at])

  defp apply_pagination(query, %{page: page, per_page: per}) do
    offset = (page - 1) * per
    from p in query, limit: ^per, offset: ^offset
  end
  defp apply_pagination(query, _), do: query
end
```

---

## Step 883: Ecto Multi Patterns

```elixir
defmodule MyApp.Orders do
  def create_order(attrs) do
    Ecto.Multi.new()
    |> Ecto.Multi.insert(:order, Order.changeset(%Order{}, attrs))
    |> Ecto.Multi.run(:reserve_stock, fn _repo, %{order: order} ->
      Enum.reduce_while(order.items, {:ok, []}, fn item, {:ok, acc} ->
        case reserve_single_item(item) do
          {:ok, reservation} -> {:cont, {:ok, [reservation | acc]}}
          error              -> {:halt, error}
        end
      end)
    end)
    |> Ecto.Multi.run(:calculate_total, fn _repo, %{order: order} ->
      total = calculate_total(order)
      {:ok, total}
    end)
    |> Ecto.Multi.update(:order_with_total, fn %{order: order, calculate_total: total} ->
      Order.changeset(order, %{total: total})
    end)
    |> Ecto.Multi.run(:send_confirmation, fn _repo, %{order_with_total: order} ->
      MyApp.Mailer.send_order_confirmation(order)
      {:ok, :sent}
    end)
    |> MyApp.Repo.transaction()
    |> case do
      {:ok, %{order_with_total: order}} -> {:ok, order}
      {:error, :order, changeset, _}    -> {:error, changeset}
      {:error, :reserve_stock, err, _}  -> {:error, :out_of_stock, err}
      {:error, step, err, _}            -> {:error, step, err}
    end
  end
end
```

---

## Step 884: Custom Ecto Types

```elixir
defmodule MyApp.Ecto.Money do
  @behaviour Ecto.Type

  def type, do: :integer  # stored as cents

  def cast(%Money{} = money), do: {:ok, money}
  def cast(%{amount: amount, currency: currency}) do
    {:ok, %Money{amount: Decimal.new(amount), currency: currency}}
  end
  def cast(value) when is_integer(value) do
    {:ok, %Money{amount: Decimal.new(value / 100), currency: "USD"}}
  end
  def cast(_), do: :error

  def load(cents) when is_integer(cents) do
    {:ok, %Money{amount: Decimal.div(Decimal.new(cents), 100), currency: "USD"}}
  end
  def load(_), do: :error

  def dump(%Money{amount: amount}) do
    cents = amount |> Decimal.mult(100) |> Decimal.round(0) |> Decimal.to_integer()
    {:ok, cents}
  end
  def dump(_), do: :error

  def equal?(%Money{amount: a}, %Money{amount: b}), do: Decimal.eq?(a, b)
  def embed_as(_), do: :self
end

defmodule MyApp.Ecto.Slug do
  @behaviour Ecto.Type

  def type, do: :string

  def cast(value) when is_binary(value) do
    slug = value
      |> String.downcase()
      |> String.replace(~r/[^a-z0-9\s-]/, "")
      |> String.replace(~r/\s+/, "-")
      |> String.replace(~r/-+/, "-")
      |> String.trim("-")

    if String.length(slug) > 0, do: {:ok, slug}, else: :error
  end
  def cast(_), do: :error

  def load(value), do: {:ok, value}
  def dump(value) when is_binary(value), do: {:ok, value}
  def dump(_), do: :error
end
```

---

## Step 885: Ecto Changesets Advanced

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :name,         :string
    field :email,        :string
    field :password,     :string, virtual: true
    field :password_hash, :string
    field :role,         Ecto.Enum, values: [:user, :admin, :moderator], default: :user
    field :settings,     :map, default: %{}
    timestamps()
  end

  def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :email, :password])
    |> validate_required([:name, :email, :password])
    |> validate_format(:email, ~r/@/)
    |> validate_length(:name, min: 2, max: 100)
    |> validate_length(:password, min: 8)
    |> validate_password_strength()
    |> unique_constraint(:email, name: :users_email_lower_index)
    |> hash_password()
  end

  def update_changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :settings])
    |> validate_required([:name])
    |> validate_length(:name, min: 2, max: 100)
    |> validate_settings()
  end

  defp validate_password_strength(changeset) do
    validate_change(changeset, :password, fn :password, password ->
      errors = []
      errors = if String.length(password) < 12, do: [{:password, "should be at least 12 chars"} | errors], else: errors
      errors = if not (password =~ ~r/[A-Z]/), do: [{:password, "needs uppercase letter"} | errors], else: errors
      errors = if not (password =~ ~r/[0-9]/), do: [{:password, "needs a digit"} | errors], else: errors
      errors
    end)
  end

  defp hash_password(%Ecto.Changeset{valid?: true, changes: %{password: pw}} = cs) do
    change(cs, password_hash: Bcrypt.hash_pwd_salt(pw), password: nil)
  end
  defp hash_password(changeset), do: changeset

  defp validate_settings(changeset) do
    validate_change(changeset, :settings, fn :settings, settings ->
      allowed = ~w(notifications theme language)
      invalid = Map.keys(settings) -- allowed
      if Enum.empty?(invalid), do: [], else: [{:settings, "invalid keys: #{Enum.join(invalid, ", ")}"}]
    end)
  end
end
```

---

## Step 886: Schemaless Changesets

```elixir
defmodule MyApp.Forms.SearchForm do
  import Ecto.Changeset

  @types %{
    query:      :string,
    category:   :string,
    min_price:  :float,
    max_price:  :float,
    page:       :integer,
    per_page:   :integer,
    sort_by:    :string,
    in_stock:   :boolean
  }

  def changeset(params) do
    {%{}, @types}
    |> cast(params, Map.keys(@types))
    |> validate_length(:query, max: 100)
    |> validate_number(:min_price, greater_than_or_equal_to: 0)
    |> validate_number(:max_price, greater_than_or_equal_to: 0)
    |> validate_number(:page, greater_than_or_equal_to: 1)
    |> validate_number(:per_page, greater_than_or_equal_to: 1, less_than_or_equal_to: 100)
    |> validate_price_range()
  end

  defp validate_price_range(changeset) do
    min = get_field(changeset, :min_price)
    max = get_field(changeset, :max_price)
    
    if min && max && min > max do
      add_error(changeset, :min_price, "cannot be greater than max_price")
    else
      changeset
    end
  end

  def to_filters(changeset) do
    apply_changes(changeset)
    |> Enum.reject(fn {_, v} -> is_nil(v) end)
    |> Map.new()
  end
end
```

---

## Step 887: Ecto Associations Advanced

```elixir
defmodule MyApp.Orders.Order do
  use Ecto.Schema

  schema "orders" do
    belongs_to :customer, MyApp.Accounts.Customer
    
    has_many :items, MyApp.Orders.OrderItem,
      preload_order: [inserted_at: :asc],
      on_delete:     :delete_all
    
    has_many :products, through: [:items, :product]
    
    has_one :shipment, MyApp.Shipping.Shipment,
      on_replace: :update
    
    has_one :invoice, MyApp.Billing.Invoice
    
    many_to_many :tags, MyApp.Tag,
      join_through: "order_tags",
      on_replace:   :delete
    
    timestamps()
  end

  # Nested changeset
  def changeset(order, attrs) do
    order
    |> cast(attrs, [:customer_id])
    |> cast_assoc(:items, with: &OrderItem.changeset/2, required: true)
    |> cast_assoc(:shipment, with: &Shipment.changeset/2)
    |> validate_required([:customer_id])
    |> validate_length(:items, min: 1)
    |> put_assoc(:tags, parse_tags(attrs))
  end

  defp parse_tags(%{"tags" => tags}) when is_list(tags) do
    Enum.map(tags, fn name ->
      case MyApp.Repo.get_by(Tag, name: name) do
        nil -> %Tag{name: name}
        tag -> tag
      end
    end)
  end
  defp parse_tags(_), do: []
end
```

---

## Step 888: Ecto Pagination

```elixir
defmodule MyApp.Pagination do
  import Ecto.Query

  def paginate(query, opts \\ []) do
    page     = Keyword.get(opts, :page, 1)
    per_page = Keyword.get(opts, :per_page, 20)

    total_count = MyApp.Repo.aggregate(query, :count)

    entries = query
      |> limit(^per_page)
      |> offset(^((page - 1) * per_page))
      |> MyApp.Repo.all()

    total_pages = ceil(total_count / per_page)

    %{
      entries:     entries,
      page:        page,
      per_page:    per_page,
      total_count: total_count,
      total_pages: total_pages,
      has_next:    page < total_pages,
      has_prev:    page > 1
    }
  end

  # Cursor-based pagination (more efficient for large tables)
  def paginate_by_cursor(query, opts \\ []) do
    per_page   = Keyword.get(opts, :per_page, 20)
    after_id   = Keyword.get(opts, :after_id)
    before_id  = Keyword.get(opts, :before_id)

    query = if after_id, do: where(query, [r], r.id > ^after_id), else: query
    query = if before_id, do: where(query, [r], r.id < ^before_id), else: query

    entries = query
      |> order_by([r], asc: r.id)
      |> limit(^(per_page + 1))
      |> MyApp.Repo.all()

    has_more  = length(entries) > per_page
    entries   = if has_more, do: Enum.take(entries, per_page), else: entries
    next_id   = if has_more, do: List.last(entries).id, else: nil

    %{entries: entries, next_id: next_id, has_more: has_more}
  end
end
```

---

## Step 889: Database Migrations Advanced

```elixir
defmodule MyApp.Repo.Migrations.AddFullTextSearch do
  use Ecto.Migration

  def up do
    # Add tsvector column with trigger-based update
    execute """
    ALTER TABLE products ADD COLUMN search_vector tsvector;
    """

    execute """
    CREATE INDEX products_search_idx ON products USING gin(search_vector);
    """

    execute """
    CREATE OR REPLACE FUNCTION update_product_search()
    RETURNS trigger AS $$
    BEGIN
      NEW.search_vector :=
        setweight(to_tsvector('english', coalesce(NEW.name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.description, '')), 'B');
      RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;
    """

    execute """
    CREATE TRIGGER product_search_update
      BEFORE INSERT OR UPDATE ON products
      FOR EACH ROW EXECUTE FUNCTION update_product_search();
    """

    # Update existing rows
    execute "UPDATE products SET search_vector = to_tsvector('english', name || ' ' || coalesce(description, ''));"
  end

  def down do
    execute "DROP TRIGGER IF EXISTS product_search_update ON products;"
    execute "DROP FUNCTION IF EXISTS update_product_search();"
    execute "DROP INDEX IF EXISTS products_search_idx;"
    execute "ALTER TABLE products DROP COLUMN IF EXISTS search_vector;"
  end
end
```

---

## Step 890: Full-Text Search

```elixir
defmodule MyApp.Search do
  import Ecto.Query

  def search_products(query_string, opts \\ []) do
    page    = Keyword.get(opts, :page, 1)
    per     = Keyword.get(opts, :per_page, 20)

    tsquery = format_tsquery(query_string)

    from p in Product,
      where: fragment("? @@ plainto_tsquery('english', ?)", p.search_vector, ^query_string),
      order_by: [
        desc: fragment("ts_rank(?, plainto_tsquery('english', ?))", p.search_vector, ^query_string)
      ],
      select: %{
        id:       p.id,
        name:     p.name,
        price:    p.price,
        rank:     fragment("ts_rank(?, plainto_tsquery('english', ?))", p.search_vector, ^query_string),
        headline: fragment(
          "ts_headline('english', ?, plainto_tsquery('english', ?), 'StartSel=<mark>, StopSel=</mark>')",
          p.description, ^query_string
        )
      },
      limit:  ^per,
      offset: ^((page - 1) * per)
    |> MyApp.Repo.all()
  end

  def suggest_completions(prefix) do
    from p in Product,
      where: ilike(p.name, ^"#{prefix}%"),
      select: p.name,
      limit: 10
    |> MyApp.Repo.all()
  end

  defp format_tsquery(input) do
    input
    |> String.split()
    |> Enum.map(&"#{&1}:*")
    |> Enum.join(" & ")
  end
end
```

---

## สรุป Part 82

✅ **Step 881** - Complex queries  
✅ **Step 882** - Dynamic queries  
✅ **Step 883** - Ecto.Multi patterns  
✅ **Step 884** - Custom Ecto types  
✅ **Step 885** - Changeset advanced  
✅ **Step 886** - Schemaless changesets  
✅ **Step 887** - Associations advanced  
✅ **Step 888** - Pagination  
✅ **Step 889** - Migration advanced  
✅ **Step 890** - Full-text search  

➡️ [Part 83: Performance Optimization](./part-83-performance.md)
