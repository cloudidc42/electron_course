# Part 90: Search & Discovery (Steps 961-970)

## Step 961: Elasticsearch Integration

```elixir
# mix.exs: {:elasticsearch, "~> 1.0"}

defmodule MyApp.Search.ES do
  @index "products"

  def search(query, opts \\ []) do
    body = build_query(query, opts)
    
    Elasticsearch.post(MyApp.Elasticsearch, "/#{@index}/_search", body)
    |> case do
      {:ok, %{"hits" => %{"hits" => hits, "total" => %{"value" => total}}}} ->
        {:ok, %{results: Enum.map(hits, &format_hit/1), total: total}}
      {:error, reason} ->
        {:error, reason}
    end
  end

  def index_product(product) do
    doc = %{
      id:          product.id,
      name:        product.name,
      description: product.description,
      price:       product.price,
      category:    product.category.name,
      tags:        product.tags,
      in_stock:    product.stock > 0
    }
    
    Elasticsearch.put(MyApp.Elasticsearch, "/#{@index}/_doc/#{product.id}", doc)
  end

  defp build_query(text, opts) do
    %{
      query: %{
        bool: %{
          must:   [%{multi_match: %{query: text, fields: ["name^3", "description", "tags"]}}],
          filter: build_filters(opts)
        }
      },
      sort:        build_sort(Keyword.get(opts, :sort, :relevance)),
      from:        (Keyword.get(opts, :page, 1) - 1) * Keyword.get(opts, :per_page, 20),
      size:        Keyword.get(opts, :per_page, 20),
      highlight:   %{fields: %{name: %{}, description: %{number_of_fragments: 3}}}
    }
  end

  defp build_filters(opts) do
    filters = []
    filters = if cat = Keyword.get(opts, :category), do: [%{term: %{category: cat}} | filters], else: filters
    filters = if price_max = Keyword.get(opts, :max_price), do: [%{range: %{price: %{lte: price_max}}} | filters], else: filters
    filters = if Keyword.get(opts, :in_stock), do: [%{term: %{in_stock: true}} | filters], else: filters
    filters
  end

  defp build_sort(:relevance), do: ["_score"]
  defp build_sort(:price_asc), do: [%{price: "asc"}]
  defp build_sort(:newest),    do: [%{inserted_at: "desc"}]

  defp format_hit(%{"_source" => source, "highlight" => highlight}) do
    Map.merge(source, %{"highlights" => highlight})
  end
  defp format_hit(%{"_source" => source}), do: source
end
```

---

## Step 962: PostgreSQL Full-Text Search

```elixir
defmodule MyApp.Search.PG do
  import Ecto.Query

  def search(text, opts \\ []) do
    tsquery = to_tsquery(text)
    
    from(p in Product,
      where: fragment("search_vector @@ to_tsquery('english', ?)", ^tsquery),
      select: %{
        id:    p.id,
        name:  p.name,
        rank:  fragment("ts_rank(search_vector, to_tsquery('english', ?))", ^tsquery),
        headline: fragment(
          "ts_headline('english', ?, to_tsquery('english', ?), 'MaxWords=30, MinWords=15')",
          p.description,
          ^tsquery
        )
      },
      order_by: [desc: fragment("ts_rank(search_vector, to_tsquery('english', ?))", ^tsquery)],
      limit: ^Keyword.get(opts, :limit, 20)
    )
    |> filter_by(opts)
    |> MyApp.Repo.all()
  end

  # Trigram similarity search (for typos)
  def fuzzy_search(text) do
    from(p in Product,
      where: fragment("similarity(name, ?) > 0.3", ^text),
      order_by: [desc: fragment("similarity(name, ?)", ^text)],
      limit: 10
    )
    |> MyApp.Repo.all()
  end

  defp to_tsquery(text) do
    text
    |> String.split()
    |> Enum.map(&"#{&1}:*")
    |> Enum.join(" & ")
  end

  defp filter_by(query, opts) do
    Enum.reduce(opts, query, fn
      {:category_id, id}, q -> where(q, [p], p.category_id == ^id)
      {:min_price, price},  q -> where(q, [p], p.price >= ^price)
      {:max_price, price},  q -> where(q, [p], p.price <= ^price)
      _, q -> q
    end)
  end
end
```

---

## Step 963: Faceted Search

```elixir
defmodule MyApp.Search.Facets do
  import Ecto.Query

  def search_with_facets(text, filters \\ %{}) do
    base_query = build_base_query(text, filters)
    
    results = MyApp.Repo.all(base_query)
    
    facets = %{
      categories: compute_facet(base_query, :category_id, [:category_name]),
      price_ranges: compute_price_ranges(base_query),
      ratings: compute_facet(base_query, :avg_rating, []),
      in_stock: compute_boolean_facet(base_query, :stock)
    }
    
    %{results: results, facets: facets}
  end

  defp compute_facet(query, field, extra_fields) do
    from(p in subquery(query),
      group_by: [^field | extra_fields],
      select: %{
        value: field(p, ^field),
        count: count(p.id)
      },
      order_by: [desc: count(p.id)],
      limit: 20
    )
    |> MyApp.Repo.all()
  end

  defp compute_price_ranges(query) do
    ranges = [{0, 25}, {25, 50}, {50, 100}, {100, 250}, {250, nil}]
    
    Enum.map(ranges, fn {min, max} ->
      count = from(p in subquery(query),
        where: p.price >= ^min and (is_nil(^max) or p.price < ^max),
        select: count(p.id)
      ) |> MyApp.Repo.one()
      
      %{min: min, max: max, count: count}
    end)
  end

  defp compute_boolean_facet(query, field) do
    from(p in subquery(query),
      group_by: ^field,
      select: {field(p, ^field), count(p.id)}
    )
    |> MyApp.Repo.all()
    |> Map.new()
  end

  defp build_base_query(text, filters) do
    tsquery = text && MyApp.Search.PG.to_tsquery(text)
    
    from(p in Product,
      join: c in assoc(p, :category), as: :category,
      where: ^(if text, do: dynamic([p], fragment("search_vector @@ to_tsquery('english', ?)", ^tsquery)), else: true)
    )
    |> apply_filters(filters)
  end

  defp apply_filters(query, filters) do
    Enum.reduce(filters, query, fn
      {"category_id", id}, q -> where(q, [p], p.category_id == ^id)
      {"price_min", val},  q -> where(q, [p], p.price >= ^val)
      {"price_max", val},  q -> where(q, [p], p.price <= ^val)
      {"in_stock", "true"}, q -> where(q, [p], p.stock > 0)
      _, q -> q
    end)
  end
end
```

---

## Step 964: Autocomplete

```elixir
defmodule MyApp.Search.Autocomplete do
  # Redis-based autocomplete

  def add_to_index(text) do
    text_lower = String.downcase(text)
    
    # Add all prefixes
    prefixes = for i <- 1..String.length(text_lower) do
      String.slice(text_lower, 0, i)
    end
    
    Enum.each(prefixes, fn prefix ->
      Redix.command(:redix, ["ZADD", "autocomplete", 0, prefix])
    end)
    
    # Mark completion with special char
    Redix.command(:redix, ["ZADD", "autocomplete", 0, text_lower <> "*"])
  end

  def suggest(prefix, limit \\ 10) do
    prefix_lower = String.downcase(prefix)
    
    # Find range in sorted set
    {:ok, entries} = Redix.command(:redix, [
      "ZRANGEBYLEX", "autocomplete",
      "[#{prefix_lower}",
      "[#{prefix_lower}\xff",
      "LIMIT", "0", "50"
    ])
    
    entries
    |> Enum.filter(&String.ends_with?(&1, "*"))
    |> Enum.map(&String.trim_trailing(&1, "*"))
    |> Enum.take(limit)
  end

  # Typo-tolerant with Levenshtein distance
  def suggest_fuzzy(text, limit \\ 5) do
    from(p in Product,
      where: fragment("levenshtein(lower(name), ?) <= 2", ^String.downcase(text)),
      order_by: fragment("levenshtein(lower(name), ?)", ^String.downcase(text)),
      select: p.name,
      limit: ^limit
    )
    |> MyApp.Repo.all()
  end
end
```

---

## Step 965: Search Analytics

```elixir
defmodule MyApp.Search.Analytics do
  use Ecto.Schema
  import Ecto.Query

  schema "search_events" do
    field :query,       :string
    field :user_id,     :integer
    field :result_count, :integer
    field :clicked_ids, {:array, :integer}, default: []
    field :converted,   :boolean, default: false

    timestamps(updated_at: false)
  end

  def track_search(query, user_id, result_count) do
    %__MODULE__{
      query:        query,
      user_id:      user_id,
      result_count: result_count
    }
    |> MyApp.Repo.insert()
  end

  def track_click(search_id, product_id) do
    from(e in __MODULE__, where: e.id == ^search_id)
    |> MyApp.Repo.update_all(
      push: [clicked_ids: product_id]
    )
  end

  # Popular searches
  def trending(limit \\ 10) do
    cutoff = DateTime.add(DateTime.utc_now(), -7 * 24 * 3600)
    
    from(e in __MODULE__,
      where: e.inserted_at > ^cutoff and e.result_count > 0,
      group_by: e.query,
      having: count(e.id) >= 5,
      select: %{query: e.query, count: count(e.id)},
      order_by: [desc: count(e.id)],
      limit: ^limit
    )
    |> MyApp.Repo.all()
  end

  # Zero-result searches
  def zero_results(limit \\ 20) do
    from(e in __MODULE__,
      where: e.result_count == 0,
      group_by: e.query,
      select: %{query: e.query, count: count(e.id)},
      order_by: [desc: count(e.id)],
      limit: ^limit
    )
    |> MyApp.Repo.all()
  end
end
```

---

## Step 966: Recommendation Engine

```elixir
defmodule MyApp.Recommendations do
  import Ecto.Query

  # Collaborative filtering - "users like you also bought"
  def for_user(user_id, limit \\ 10) do
    # Find similar users
    user_purchases = get_purchased_ids(user_id)
    
    similar_users = from(o in Order,
      join: oi in assoc(o, :items),
      where: oi.product_id in ^user_purchases and o.user_id != ^user_id,
      group_by: o.user_id,
      having: count(oi.product_id) >= 2,
      select: o.user_id
    ) |> MyApp.Repo.all()

    # Get products those users bought that this user hasn't
    from(oi in OrderItem,
      join: o in assoc(oi, :order),
      where: o.user_id in ^similar_users and oi.product_id not in ^user_purchases,
      group_by: oi.product_id,
      select: %{product_id: oi.product_id, score: count(oi.id)},
      order_by: [desc: count(oi.id)],
      limit: ^limit
    )
    |> MyApp.Repo.all()
    |> preload_products()
  end

  # Content-based - "similar to this product"
  def similar_to(product_id, limit \\ 5) do
    product = MyApp.Repo.get!(Product, product_id)
    
    from(p in Product,
      where: p.id != ^product_id,
      where: p.category_id == ^product.category_id,
      order_by: fragment("similarity(name, ?) DESC", ^product.name),
      limit: ^limit
    )
    |> MyApp.Repo.all()
  end

  # Trending products
  def trending(period_hours \\ 24, limit \\ 10) do
    cutoff = DateTime.add(DateTime.utc_now(), -period_hours * 3600)
    
    from(oi in OrderItem,
      join: o in assoc(oi, :order),
      where: o.inserted_at > ^cutoff,
      group_by: oi.product_id,
      select: %{product_id: oi.product_id, score: sum(oi.quantity)},
      order_by: [desc: sum(oi.quantity)],
      limit: ^limit
    )
    |> MyApp.Repo.all()
    |> preload_products()
  end

  defp get_purchased_ids(user_id) do
    from(oi in OrderItem,
      join: o in assoc(oi, :order),
      where: o.user_id == ^user_id,
      select: oi.product_id,
      distinct: true
    ) |> MyApp.Repo.all()
  end

  defp preload_products(items) do
    product_ids = Enum.map(items, & &1.product_id)
    products = MyApp.Repo.all(from p in Product, where: p.id in ^product_ids) |> Map.new(&{&1.id, &1})
    Enum.map(items, fn item -> Map.put(item, :product, products[item.product_id]) end)
  end
end
```

---

## Step 967: Algolia Integration

```elixir
defmodule MyApp.Search.Algolia do
  @app_id  System.get_env("ALGOLIA_APP_ID")
  @api_key System.get_env("ALGOLIA_API_KEY")
  @index   "products"

  def search(query, params \\ %{}) do
    request_body = Map.merge(%{
      "query"            => query,
      "hitsPerPage"      => 20,
      "attributesToRetrieve" => ["id", "name", "price", "category"],
      "attributesToHighlight" => ["name", "description"],
      "facets"           => ["category", "price_range"],
      "numericFilters"   => build_numeric_filters(params)
    }, params)

    Req.post("https://#{@app_id}-dsn.algolia.net/1/indexes/#{@index}/query",
      json:    request_body,
      headers: [
        {"x-algolia-application-id", @app_id},
        {"x-algolia-api-key",        @api_key}
      ]
    )
    |> case do
      {:ok, %{status: 200, body: body}} -> {:ok, body}
      {:error, reason}                  -> {:error, reason}
    end
  end

  def index_product(product) do
    object = %{
      "objectID"    => to_string(product.id),
      "name"        => product.name,
      "description" => product.description,
      "price"       => product.price,
      "category"    => product.category.name,
      "tags"        => product.tags,
      "_tags"       => product.tags  # Algolia tag format
    }

    Req.put("https://#{@app_id}.algolia.net/1/indexes/#{@index}/#{product.id}",
      json: object,
      headers: algolia_headers(:write)
    )
  end

  defp build_numeric_filters(%{"price_max" => max}), do: ["price <= #{max}"]
  defp build_numeric_filters(_), do: []

  defp algolia_headers(:write) do
    [{"x-algolia-application-id", @app_id}, {"x-algolia-api-key", System.get_env("ALGOLIA_WRITE_KEY")}]
  end
end
```

---

## Step 968: Vector Search / Semantic Search

```elixir
defmodule MyApp.Search.Semantic do
  # Semantic search using vector embeddings

  def search(text, limit \\ 10) do
    # Get embedding for search query
    embedding = get_embedding(text)
    
    # Query PostgreSQL with pgvector extension
    {:ok, result} = MyApp.Repo.query(
      """
      SELECT id, name, description,
             1 - (embedding <=> $1::vector) AS similarity
      FROM products
      WHERE 1 - (embedding <=> $1::vector) > 0.7
      ORDER BY embedding <=> $1::vector
      LIMIT $2
      """,
      [embedding, limit]
    )
    
    Enum.map(result.rows, fn [id, name, desc, sim] ->
      %{id: id, name: name, description: desc, similarity: sim}
    end)
  end

  def index_product_embedding(product) do
    text = "#{product.name} #{product.description}"
    embedding = get_embedding(text)
    
    MyApp.Repo.query!(
      "UPDATE products SET embedding = $1::vector WHERE id = $2",
      [embedding, product.id]
    )
  end

  defp get_embedding(text) do
    # Use OpenAI embeddings API or Bumblebee
    case Req.post("https://api.openai.com/v1/embeddings",
           json: %{input: text, model: "text-embedding-3-small"},
           headers: [{"authorization", "Bearer #{System.get_env("OPENAI_API_KEY")}"}]) do
      {:ok, %{status: 200, body: %{"data" => [%{"embedding" => emb}]}}} -> emb
      _ -> raise "Failed to get embedding"
    end
  end
end
```

---

## Step 969: Search Spell Check

```elixir
defmodule MyApp.Search.SpellCheck do
  # Word correction using SymSpell algorithm (simplified)
  # or external API

  def correct(text) do
    words = String.split(text)
    corrected = Enum.map(words, &correct_word/1)
    
    if corrected != words do
      {:corrected, Enum.join(corrected, " ")}
    else
      :no_correction
    end
  end

  defp correct_word(word) do
    # Check against known good words in Redis
    case Redix.command(:redix, ["ZSCORE", "known_words", String.downcase(word)]) do
      {:ok, score} when not is_nil(score) ->
        word
      {:ok, nil} ->
        # Find closest word
        suggestion = find_suggestion(String.downcase(word))
        suggestion || word
    end
  end

  defp find_suggestion(word) do
    edits = generate_edits(word)
    
    edits
    |> Enum.find_value(fn edit ->
      case Redix.command(:redix, ["ZSCORE", "known_words", edit]) do
        {:ok, score} when not is_nil(score) -> edit
        _ -> nil
      end
    end)
  end

  defp generate_edits(word) do
    chars = String.graphemes(word)
    len   = length(chars)
    
    # Deletions (remove one char)
    deletions = for i <- 0..(len - 1) do
      (Enum.take(chars, i) ++ Enum.drop(chars, i + 1)) |> Enum.join()
    end
    
    # Transpositions (swap adjacent chars)
    transpositions = for i <- 0..(len - 2) do
      chars2 = chars
        |> List.replace_at(i,     Enum.at(chars, i + 1))
        |> List.replace_at(i + 1, Enum.at(chars, i))
      Enum.join(chars2)
    end

    deletions ++ transpositions
  end

  def populate_dictionary do
    MyApp.Repo.all(from p in Product, select: p.name)
    |> Enum.flat_map(&String.split/1)
    |> Enum.map(&String.downcase/1)
    |> Enum.each(fn word ->
      Redix.command(:redix, ["ZINCRBY", "known_words", 1, word])
    end)
  end
end
```

---

## Step 970: Search Personalization

```elixir
defmodule MyApp.Search.Personalization do
  def personalized_search(user_id, query, opts \\ []) do
    # Get user preferences
    profile = get_user_search_profile(user_id)
    
    # Boost based on category preferences
    boosted_opts = opts
      |> Keyword.merge(category_boosts: profile.preferred_categories)
      |> Keyword.merge(price_range: profile.typical_price_range)

    results = MyApp.Search.PG.search(query, boosted_opts)
    
    # Re-rank based on personal history
    reranked = rerank(results, profile)
    
    # Track this search
    Task.start(fn -> update_profile(user_id, query, reranked) end)
    
    reranked
  end

  defp get_user_search_profile(user_id) do
    cache_key = "search_profile:#{user_id}"
    
    case Redix.command(:redix, ["GET", cache_key]) do
      {:ok, nil}     -> build_profile(user_id)
      {:ok, cached}  -> Jason.decode!(cached, keys: :atoms)
    end
  end

  defp build_profile(user_id) do
    # Analyze purchase history
    purchases = MyApp.Orders.user_purchases(user_id, limit: 50)
    
    categories = purchases
      |> Enum.group_by(& &1.category_id)
      |> Enum.map(fn {cat, items} -> {cat, length(items)} end)
      |> Enum.sort_by(&elem(&1, 1), :desc)
      |> Enum.take(5)
      |> Enum.map(&elem(&1, 0))

    avg_price = purchases
      |> Enum.map(& &1.price)
      |> Enum.sum()
      |> Kernel./(max(length(purchases), 1))

    profile = %{
      preferred_categories: categories,
      typical_price_range:  {avg_price * 0.5, avg_price * 2.0}
    }
    
    Redix.command(:redix, ["SETEX", "search_profile:#{user_id}", 3600, Jason.encode!(profile)])
    profile
  end

  defp rerank(results, %{preferred_categories: cats}) do
    Enum.sort_by(results, fn r ->
      category_bonus = if r.category_id in cats, do: -10, else: 0
      r.rank + category_bonus
    end)
  end

  defp update_profile(user_id, _query, _results) do
    Redix.command(:redix, ["DEL", "search_profile:#{user_id}"])
  end
end
```

---

## สรุป Part 90

✅ **Step 961** - Elasticsearch integration  
✅ **Step 962** - PostgreSQL full-text search  
✅ **Step 963** - Faceted search  
✅ **Step 964** - Autocomplete  
✅ **Step 965** - Search analytics  
✅ **Step 966** - Recommendation engine  
✅ **Step 967** - Algolia integration  
✅ **Step 968** - Vector / semantic search  
✅ **Step 969** - Spell check  
✅ **Step 970** - Search personalization  

➡️ [Part 91: Payment Processing](./part-91-payments.md)
