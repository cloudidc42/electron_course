# Part 60: Search Engine Integration (Steps 651-670)

## Step 651: Full-Text Search with PostgreSQL

```elixir
defmodule MyApp.Search.PostgresSearch do
  import Ecto.Query

  # PostgreSQL full-text search with tsvector
  def search_products(query, opts \\ []) do
    limit  = Keyword.get(opts, :limit, 20)
    offset = Keyword.get(opts, :offset, 0)

    from(p in MyApp.Catalog.Product,
      where: fragment(
        "to_tsvector('english', ? || ' ' || ?) @@ plainto_tsquery('english', ?)",
        p.name, p.description, ^query
      ),
      order_by: fragment(
        "ts_rank(to_tsvector('english', ? || ' ' || ?), plainto_tsquery('english', ?)) DESC",
        p.name, p.description, ^query
      ),
      limit:  ^limit,
      offset: ^offset
    )
    |> MyApp.Repo.all()
  end

  # Use GIN index for performance
  # CREATE INDEX products_fts_idx ON products
  #   USING GIN(to_tsvector('english', name || ' ' || description));

  def search_with_highlight(query) do
    from(p in MyApp.Catalog.Product,
      where: fragment(
        "to_tsvector('english', ? || ' ' || ?) @@ plainto_tsquery('english', ?)",
        p.name, p.description, ^query
      ),
      select: %{
        id:          p.id,
        name:        p.name,
        highlight:   fragment(
          "ts_headline('english', ?, plainto_tsquery('english', ?))",
          p.description, ^query
        ),
        rank:        fragment(
          "ts_rank(to_tsvector('english', ? || ' ' || ?), plainto_tsquery('english', ?))",
          p.name, p.description, ^query
        )
      },
      order_by: fragment("rank DESC")
    )
    |> MyApp.Repo.all()
  end
end
```

---

## Step 652: Elasticsearch Integration

```elixir
# mix.exs: {:elasticsearch, "~> 3.0"} or {:elastic_search, "~> 1.0"}

defmodule MyApp.Search.Elasticsearch do
  @index "products"
  @url   "http://localhost:9200"

  def index_product(product) do
    doc = %{
      id:          product.id,
      name:        product.name,
      description: product.description,
      category:    product.category,
      price:       Decimal.to_float(product.price),
      tags:        product.tags,
      created_at:  DateTime.to_iso8601(product.inserted_at)
    }

    HTTPoison.put!(
      "#{@url}/#{@index}/_doc/#{product.id}",
      Jason.encode!(doc),
      [{"Content-Type", "application/json"}]
    )
  end

  def search(query, opts \\ []) do
    body = %{
      query: %{
        multi_match: %{
          query:  query,
          fields: ["name^3", "description", "tags"],
          type:   "best_fields",
          fuzziness: "AUTO"
        }
      },
      highlight: %{
        fields: %{
          name:        %{},
          description: %{fragment_size: 150, number_of_fragments: 3}
        }
      },
      from: Keyword.get(opts, :offset, 0),
      size: Keyword.get(opts, :limit, 20)
    }

    response = HTTPoison.post!(
      "#{@url}/#{@index}/_search",
      Jason.encode!(body),
      [{"Content-Type", "application/json"}]
    )

    %{"hits" => %{"hits" => hits, "total" => %{"value" => total}}} =
      Jason.decode!(response.body)

    %{
      total:   total,
      results: Enum.map(hits, &parse_hit/1)
    }
  end

  defp parse_hit(hit) do
    %{
      id:        hit["_id"],
      score:     hit["_score"],
      highlight: hit["highlight"],
      source:    hit["_source"]
    }
  end

  def create_index do
    mappings = %{
      mappings: %{
        properties: %{
          name:        %{type: "text", analyzer: "english", boost: 3},
          description: %{type: "text", analyzer: "english"},
          category:    %{type: "keyword"},
          price:       %{type: "float"},
          tags:        %{type: "keyword"},
          created_at:  %{type: "date"}
        }
      },
      settings: %{
        number_of_shards:   1,
        number_of_replicas: 1,
        analysis: %{
          analyzer: %{
            english: %{
              tokenizer: "standard",
              filter:    ["lowercase", "english_stop", "english_stemmer"]
            }
          }
        }
      }
    }

    HTTPoison.put!("#{@url}/#{@index}", Jason.encode!(mappings),
      [{"Content-Type", "application/json"}])
  end
end
```

---

## Step 653: Meilisearch Integration

```elixir
# Meilisearch - fast, typo-tolerant search
# mix.exs: {:meilisearch, "~> 1.0"}

defmodule MyApp.Search.Meilisearch do
  @client Meilisearch.Client.new(endpoint: "http://localhost:7700", key: "masterKey")

  def setup_index do
    Meilisearch.Index.create(@client, "products", %{primaryKey: "id"})
    
    Meilisearch.Settings.update(@client, "products", %{
      searchableAttributes: ["name", "description", "tags"],
      filterableAttributes: ["category", "price", "in_stock"],
      sortableAttributes:   ["price", "created_at"],
      rankingRules: [
        "words", "typo", "proximity", "attribute", "sort", "exactness"
      ]
    })
  end

  def index_products(products) do
    docs = Enum.map(products, fn p ->
      %{
        id:          p.id,
        name:        p.name,
        description: p.description,
        category:    p.category,
        price:       Decimal.to_float(p.price),
        tags:        p.tags || [],
        in_stock:    p.stock > 0
      }
    end)

    Meilisearch.Documents.add_or_replace(@client, "products", docs)
  end

  def search(query, opts \\ []) do
    params = %{
      q:                   query,
      limit:               Keyword.get(opts, :limit, 20),
      offset:              Keyword.get(opts, :offset, 0),
      filter:              build_filter(opts),
      sort:                Keyword.get(opts, :sort, []),
      attributesToHighlight: ["name", "description"],
      highlightPreTag:     "<mark>",
      highlightPostTag:    "</mark>"
    }

    {:ok, result} = Meilisearch.Search.search(@client, "products", params)
    result
  end

  defp build_filter(opts) do
    filters = []
    filters = if cat = Keyword.get(opts, :category),
      do: ["category = \"#{cat}\"" | filters], else: filters
    filters = if Keyword.get(opts, :in_stock),
      do: ["in_stock = true" | filters], else: filters
    filters = if {min, max} = Keyword.get(opts, :price_range),
      do: ["price >= #{min} AND price <= #{max}" | filters], else: filters
    Enum.join(filters, " AND ")
  end
end
```

---

## Step 654: Algolia Integration

```elixir
# mix.exs: {:ex_algolia, "~> 0.3"}

defmodule MyApp.Search.Algolia do
  @app_id  System.get_env("ALGOLIA_APP_ID")
  @api_key System.get_env("ALGOLIA_API_KEY")
  @index   "products"

  def client do
    ExAlgolia.new_client(@app_id, @api_key)
  end

  def index_product(product) do
    doc = %{
      objectID:    Integer.to_string(product.id),
      name:        product.name,
      description: product.description,
      category:    product.category,
      price:       Decimal.to_float(product.price),
      image_url:   product.image_url,
      _tags:       product.tags || []
    }

    ExAlgolia.save_object(client(), @index, doc)
  end

  def search(query, opts \\ []) do
    params = %{
      hitsPerPage:     Keyword.get(opts, :limit, 20),
      page:            Keyword.get(opts, :page, 0),
      filters:         build_algolia_filters(opts),
      highlightPreTag: "<mark>",
      highlightPostTag: "</mark>",
      attributesToHighlight: ["name", "description"]
    }

    ExAlgolia.search(client(), @index, query, params)
  end

  defp build_algolia_filters(opts) do
    parts = []
    parts = if cat = Keyword.get(opts, :category),
      do: ["category:\"#{cat}\"" | parts], else: parts
    Enum.join(parts, " AND ")
  end

  # Sync all products to Algolia
  def full_sync do
    MyApp.Repo.all(MyApp.Catalog.Product)
    |> Enum.chunk_every(1000)
    |> Enum.each(fn batch ->
      docs = Enum.map(batch, fn p ->
        %{objectID: Integer.to_string(p.id), name: p.name, description: p.description,
          category: p.category, price: Decimal.to_float(p.price)}
      end)
      ExAlgolia.save_objects(client(), @index, docs)
    end)
  end
end
```

---

## Step 655: Search GenServer with Caching

```elixir
defmodule MyApp.Search.Server do
  use GenServer

  @cache_ttl 300_000  # 5 minutes

  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)

  def search(query, opts \\ []) do
    cache_key = {query, opts}
    
    case :ets.lookup(:search_cache, cache_key) do
      [{_, result, expires_at}] when expires_at > System.monotonic_time(:millisecond) ->
        {:ok, result}
      _ ->
        GenServer.call(__MODULE__, {:search, query, opts})
    end
  end

  def init(_) do
    :ets.new(:search_cache, [:set, :public, :named_table, read_concurrency: true])
    schedule_cleanup()
    {:ok, %{}}
  end

  def handle_call({:search, query, opts}, _from, state) do
    result = do_search(query, opts)
    cache_key = {query, opts}
    expires_at = System.monotonic_time(:millisecond) + @cache_ttl
    :ets.insert(:search_cache, {cache_key, result, expires_at})
    {:reply, {:ok, result}, state}
  end

  def handle_info(:cleanup_cache, state) do
    now = System.monotonic_time(:millisecond)
    :ets.select_delete(:search_cache, [{{:_, :_, :"$1"}, [{:<, :"$1", now}], [true]}])
    schedule_cleanup()
    {:noreply, state}
  end

  defp do_search(query, opts) do
    case Application.get_env(:my_app, :search_backend, :postgres) do
      :postgres      -> MyApp.Search.PostgresSearch.search_products(query, opts)
      :meilisearch   -> MyApp.Search.Meilisearch.search(query, opts)
      :elasticsearch -> MyApp.Search.Elasticsearch.search(query, opts)
    end
  end

  defp schedule_cleanup do
    Process.send_after(self(), :cleanup_cache, 60_000)
  end
end
```

---

## Step 656: Autocomplete / Typeahead

```elixir
defmodule MyApp.Search.Autocomplete do
  import Ecto.Query

  def suggest(prefix, limit \\ 8) do
    # Fast prefix search using trigram index
    from(p in MyApp.Catalog.Product,
      where: fragment("? ILIKE ?", p.name, ^"#{prefix}%"),
      order_by: [desc: p.view_count],
      select: %{id: p.id, name: p.name, category: p.category},
      limit: ^limit
    )
    |> MyApp.Repo.all()
  end

  # Using Redis for autocomplete (sorted set)
  def redis_suggest(prefix, limit \\ 8) do
    # Store all product names in sorted set by score
    # On each keystroke, ZRANGEBYLEX to get suggestions

    normalized = String.downcase(prefix)
    Redix.command(:redix, [
      "ZRANGEBYLEX",
      "autocomplete:products",
      "[#{normalized}",
      "[#{normalized}\xFF",
      "LIMIT", "0", Integer.to_string(limit)
    ])
  end

  def build_redis_index do
    MyApp.Repo.all(
      from p in MyApp.Catalog.Product,
      select: %{id: p.id, name: p.name}
    )
    |> Enum.each(fn %{id: id, name: name} ->
      normalized = String.downcase(name)
      Redix.command(:redix, ["ZADD", "autocomplete:products", "0", "#{normalized}:#{id}"])
    end)
  end
end

# LiveView autocomplete component
defmodule MyAppWeb.SearchLive do
  use MyAppWeb, :live_view

  def handle_event("search_input", %{"query" => q}, socket) when byte_size(q) >= 2 do
    suggestions = MyApp.Search.Autocomplete.suggest(q)
    {:noreply, assign(socket, :suggestions, suggestions)}
  end

  def handle_event("search_input", _params, socket) do
    {:noreply, assign(socket, :suggestions, [])}
  end

  def handle_event("select_suggestion", %{"id" => id}, socket) do
    {:noreply, push_navigate(socket, to: ~p"/products/#{id}")}
  end
end
```

---

## Step 657: Faceted Search

```elixir
defmodule MyApp.Search.Facets do
  import Ecto.Query

  def search_with_facets(query, filters \\ %{}) do
    base_query = build_base_query(query, filters)

    # Run search and aggregations in parallel
    {results, facets} = Task.await_many([
      Task.async(fn -> MyApp.Repo.all(paginate(base_query, filters)) end),
      Task.async(fn -> build_facets(query) end)
    ])

    %{
      results:  results,
      facets:   facets,
      total:    MyApp.Repo.aggregate(base_query, :count)
    }
  end

  defp build_facets(query) do
    base = from(p in MyApp.Catalog.Product,
      where: fragment(
        "to_tsvector('english', ? || ' ' || ?) @@ plainto_tsquery('english', ?)",
        p.name, p.description, ^query
      )
    )

    %{
      categories: aggregate_categories(base),
      price_ranges: aggregate_price_ranges(base)
    }
  end

  defp aggregate_categories(base_query) do
    from(p in base_query,
      group_by: p.category,
      select: %{name: p.category, count: count(p.id)},
      order_by: [desc: count(p.id)]
    )
    |> MyApp.Repo.all()
  end

  defp aggregate_price_ranges(base_query) do
    [
      %{label: "Under $25",    min: 0,    max: 25},
      %{label: "$25-$50",      min: 25,   max: 50},
      %{label: "$50-$100",     min: 50,   max: 100},
      %{label: "$100 and up",  min: 100,  max: nil}
    ]
    |> Enum.map(fn range ->
      query = base_query
        |> where([p], p.price >= ^range.min)
        |> then(fn q ->
          if range.max, do: where(q, [p], p.price < ^range.max), else: q
        end)
      
      count = MyApp.Repo.aggregate(query, :count)
      Map.put(range, :count, count)
    end)
    |> Enum.filter(& &1.count > 0)
  end

  defp build_base_query(query, filters) do
    from(p in MyApp.Catalog.Product,
      where: fragment(
        "to_tsvector('english', ? || ' ' || ?) @@ plainto_tsquery('english', ?)",
        p.name, p.description, ^query
      )
    )
    |> apply_filters(filters)
  end

  defp apply_filters(q, %{"category" => cat}), do: where(q, [p], p.category == ^cat)
  defp apply_filters(q, %{"min_price" => min}), do: where(q, [p], p.price >= ^min)
  defp apply_filters(q, %{"max_price" => max}), do: where(q, [p], p.price <= ^max)
  defp apply_filters(q, _), do: q

  defp paginate(q, filters) do
    page  = Map.get(filters, "page", 1) |> to_integer()
    limit = 20
    q |> limit(^limit) |> offset(^((page - 1) * limit))
  end

  defp to_integer(n) when is_integer(n), do: n
  defp to_integer(s) when is_binary(s), do: String.to_integer(s)
end
```

---

## Step 658: Search Analytics

```elixir
defmodule MyApp.Search.Analytics do
  def track_search(query, results_count, user_id \\ nil) do
    %SearchEvent{}
    |> SearchEvent.changeset(%{
      query:          query,
      results_count:  results_count,
      user_id:        user_id,
      searched_at:    DateTime.utc_now()
    })
    |> MyApp.Repo.insert()
  end

  def track_click(query, product_id, position, user_id \\ nil) do
    %SearchClick{}
    |> SearchClick.changeset(%{
      query:       query,
      product_id:  product_id,
      position:    position,
      user_id:     user_id,
      clicked_at:  DateTime.utc_now()
    })
    |> MyApp.Repo.insert()
  end

  def popular_queries(limit \\ 10, days \\ 7) do
    since = DateTime.add(DateTime.utc_now(), -days * 86400)

    from(s in SearchEvent,
      where: s.searched_at >= ^since and s.results_count > 0,
      group_by: s.query,
      order_by: [desc: count(s.id)],
      limit: ^limit,
      select: %{query: s.query, count: count(s.id), avg_results: avg(s.results_count)}
    )
    |> MyApp.Repo.all()
  end

  def zero_result_queries(limit \\ 10, days \\ 7) do
    since = DateTime.add(DateTime.utc_now(), -days * 86400)

    from(s in SearchEvent,
      where: s.searched_at >= ^since and s.results_count == 0,
      group_by: s.query,
      order_by: [desc: count(s.id)],
      limit: ^limit,
      select: %{query: s.query, count: count(s.id)}
    )
    |> MyApp.Repo.all()
  end
end
```

---

## Step 659: Vector Search (AI-Powered)

```elixir
defmodule MyApp.Search.VectorSearch do
  # Requires pgvector PostgreSQL extension
  # CREATE EXTENSION vector;
  # ALTER TABLE products ADD COLUMN embedding vector(1536);

  def generate_embedding(text) do
    # Call OpenAI embeddings API
    Req.post!("https://api.openai.com/v1/embeddings",
      json: %{model: "text-embedding-ada-002", input: text},
      headers: [authorization: "Bearer #{System.get_env("OPENAI_API_KEY")}"]
    ).body["data"]
    |> List.first()
    |> Map.get("embedding")
  end

  def index_product(product) do
    text = "#{product.name}. #{product.description}"
    embedding = generate_embedding(text)

    from(p in MyApp.Catalog.Product, where: p.id == ^product.id)
    |> MyApp.Repo.update_all(set: [embedding: embedding])
  end

  def semantic_search(query, limit \\ 10) do
    query_embedding = generate_embedding(query)
    embedding_str   = "[#{Enum.join(query_embedding, ",")}]"

    from(p in MyApp.Catalog.Product,
      where: not is_nil(p.embedding),
      order_by: fragment("embedding <=> ?::vector", ^embedding_str),
      limit:    ^limit,
      select:   %{
        id:          p.id,
        name:        p.name,
        description: p.description,
        similarity:  fragment("1 - (embedding <=> ?::vector)", ^embedding_str)
      }
    )
    |> MyApp.Repo.all()
  end

  # Hybrid search: combine BM25 + vector similarity
  def hybrid_search(query, limit \\ 10) do
    query_embedding = generate_embedding(query)
    embedding_str   = "[#{Enum.join(query_embedding, ",")}]"

    from(p in MyApp.Catalog.Product,
      where: fragment(
        "to_tsvector('english', ? || ' ' || ?) @@ plainto_tsquery('english', ?) OR embedding IS NOT NULL",
        p.name, p.description, ^query
      ),
      order_by: fragment(
        "(0.7 * (1 - (embedding <=> ?::vector))) + (0.3 * ts_rank(to_tsvector('english', ? || ' ' || ?), plainto_tsquery('english', ?))) DESC",
        ^embedding_str, p.name, p.description, ^query
      ),
      limit: ^limit
    )
    |> MyApp.Repo.all()
  end
end
```

---

## Step 660: Search Index Sync with Oban

```elixir
defmodule MyApp.Search.SyncWorker do
  use Oban.Worker, queue: :search_sync, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"action" => "index", "product_id" => id}}) do
    product = MyApp.Catalog.get_product!(id)
    
    with :ok <- MyApp.Search.Elasticsearch.index_product(product),
         :ok <- MyApp.Search.Meilisearch.index_product(product) do
      :ok
    end
  end

  def perform(%Oban.Job{args: %{"action" => "delete", "product_id" => id}}) do
    HTTPoison.delete!("http://localhost:9200/products/_doc/#{id}")
    :ok
  end

  def schedule_full_sync do
    MyApp.Search.FullSyncWorker.new(%{}) |> Oban.insert()
  end
end

defmodule MyApp.Search.FullSyncWorker do
  use Oban.Worker, queue: :search_sync

  @impl Oban.Worker
  def perform(_job) do
    MyApp.Repo.all(MyApp.Catalog.Product)
    |> Enum.each(fn product ->
      MyApp.Search.SyncWorker.new(%{
        action:     "index",
        product_id: product.id
      })
      |> Oban.insert()
    end)
    :ok
  end
end

# Trigger re-index on product change (via Ecto changesets or triggers)
defmodule MyApp.Catalog do
  def update_product(product, attrs) do
    product
    |> Product.changeset(attrs)
    |> MyApp.Repo.update()
    |> case do
      {:ok, updated} ->
        MyApp.Search.SyncWorker.new(%{action: "index", product_id: updated.id})
        |> Oban.insert()
        {:ok, updated}
      error -> error
    end
  end
end
```

---

## สรุป Part 60

✅ **Step 651** - PostgreSQL full-text search  
✅ **Step 652** - Elasticsearch integration  
✅ **Step 653** - Meilisearch  
✅ **Step 654** - Algolia  
✅ **Step 655** - Search GenServer with cache  
✅ **Step 656** - Autocomplete / typeahead  
✅ **Step 657** - Faceted search  
✅ **Step 658** - Search analytics  
✅ **Step 659** - Vector/semantic search  
✅ **Step 660** - Index sync with Oban  

➡️ [Part 61: Email System](./part-61-email.md)
