# Part 63: AI/LLM Integration (Steps 681-700)

## Step 681: Claude API Integration

```elixir
# mix.exs: {:req, "~> 0.4"} or {:anthropic, "~> 0.1"}

defmodule MyApp.AI.Claude do
  @api_url "https://api.anthropic.com/v1"
  @model   "claude-opus-5-5"

  def client do
    Req.new(
      base_url: @api_url,
      headers:  [
        {"x-api-key", System.get_env("ANTHROPIC_API_KEY")},
        {"anthropic-version", "2023-06-01"},
        {"content-type", "application/json"}
      ]
    )
  end

  def message(prompt, opts \\ []) do
    body = %{
      model:      Keyword.get(opts, :model, @model),
      max_tokens: Keyword.get(opts, :max_tokens, 1024),
      messages:   [%{role: "user", content: prompt}],
      system:     Keyword.get(opts, :system, "")
    }

    case Req.post(client(), url: "/messages", json: body) do
      {:ok, %{status: 200, body: resp}} ->
        text = resp["content"] |> List.first() |> Map.get("text")
        {:ok, text}
      {:ok, %{status: status, body: body}} ->
        {:error, %{status: status, message: body["error"]["message"]}}
      {:error, reason} ->
        {:error, reason}
    end
  end

  def chat(messages, opts \\ []) do
    body = %{
      model:      Keyword.get(opts, :model, @model),
      max_tokens: Keyword.get(opts, :max_tokens, 2048),
      messages:   messages,
      system:     Keyword.get(opts, :system, "")
    }

    case Req.post(client(), url: "/messages", json: body) do
      {:ok, %{status: 200, body: resp}} ->
        text = resp["content"] |> List.first() |> Map.get("text")
        input_tokens  = resp["usage"]["input_tokens"]
        output_tokens = resp["usage"]["output_tokens"]
        {:ok, %{text: text, input_tokens: input_tokens, output_tokens: output_tokens}}
      {:ok, %{body: %{"error" => %{"message" => msg}}}} ->
        {:error, msg}
      {:error, reason} ->
        {:error, reason}
    end
  end
end
```

---

## Step 682: Streaming Responses

```elixir
defmodule MyApp.AI.StreamingClaude do
  def stream(prompt, pid \\ self()) do
    body = %{
      model:      "claude-opus-5-5",
      max_tokens: 2048,
      messages:   [%{role: "user", content: prompt}],
      stream:     true
    }

    Req.post!(
      "https://api.anthropic.com/v1/messages",
      json: body,
      headers: [
        {"x-api-key", System.get_env("ANTHROPIC_API_KEY")},
        {"anthropic-version", "2023-06-01"}
      ],
      into: fn {:data, chunk}, acc ->
        # Parse SSE events
        chunk
        |> String.split("\n")
        |> Enum.each(fn line ->
          case line do
            "data: " <> json ->
              case Jason.decode(json) do
                {:ok, %{"type" => "content_block_delta", "delta" => %{"text" => text}}} ->
                  send(pid, {:stream_chunk, text})
                {:ok, %{"type" => "message_stop"}} ->
                  send(pid, :stream_done)
                _ -> :ok
              end
            _ -> :ok
          end
        end)
        {:cont, acc}
      end
    )
  end

  # Use in LiveView
  defmodule MyAppWeb.AILive do
    use MyAppWeb, :live_view

    def handle_event("ask", %{"question" => q}, socket) do
      Task.start(fn -> MyApp.AI.StreamingClaude.stream(q, self()) end)
      {:noreply, assign(socket, :response, "", :streaming, true)}
    end

    def handle_info({:stream_chunk, chunk}, socket) do
      {:noreply, update(socket, :response, &(&1 <> chunk))}
    end

    def handle_info(:stream_done, socket) do
      {:noreply, assign(socket, :streaming, false)}
    end
  end
end
```

---

## Step 683: Tool Use (Function Calling)

```elixir
defmodule MyApp.AI.ToolUse do
  @tools [
    %{
      name:        "get_product",
      description: "Get product information by ID",
      input_schema: %{
        type:       "object",
        properties: %{
          product_id: %{type: "integer", description: "The product ID"}
        },
        required: ["product_id"]
      }
    },
    %{
      name:        "search_products",
      description: "Search for products by name or description",
      input_schema: %{
        type:       "object",
        properties: %{
          query: %{type: "string", description: "Search query"},
          limit: %{type: "integer", description: "Max results (default 5)"}
        },
        required: ["query"]
      }
    },
    %{
      name:        "create_order",
      description: "Create a new order for a user",
      input_schema: %{
        type:       "object",
        properties: %{
          user_id:    %{type: "integer"},
          product_id: %{type: "integer"},
          quantity:   %{type: "integer"}
        },
        required: ["user_id", "product_id", "quantity"]
      }
    }
  ]

  def run(user_message) do
    messages = [%{role: "user", content: user_message}]
    loop(messages)
  end

  defp loop(messages) do
    body = %{
      model:      "claude-opus-5-5",
      max_tokens: 2048,
      tools:      @tools,
      messages:   messages
    }

    {:ok, %{status: 200, body: resp}} = Req.post(
      MyApp.AI.Claude.client(),
      url: "/messages",
      json: body
    )

    case resp["stop_reason"] do
      "tool_use" ->
        # Process tool calls
        tool_results = process_tool_calls(resp["content"])
        
        new_messages = messages ++ [
          %{role: "assistant", content: resp["content"]},
          %{role: "user", content: tool_results}
        ]
        
        loop(new_messages)

      "end_turn" ->
        text = resp["content"]
          |> Enum.find(&(&1["type"] == "text"))
          |> Map.get("text")
        {:ok, text}
    end
  end

  defp process_tool_calls(content) do
    content
    |> Enum.filter(&(&1["type"] == "tool_use"))
    |> Enum.map(fn tool_use ->
      result = execute_tool(tool_use["name"], tool_use["input"])
      %{
        type:      "tool_result",
        tool_use_id: tool_use["id"],
        content:   Jason.encode!(result)
      }
    end)
  end

  defp execute_tool("get_product", %{"product_id" => id}) do
    case MyApp.Catalog.get_product(id) do
      nil  -> %{error: "Product not found"}
      prod -> Map.take(prod, [:id, :name, :description, :price, :stock])
    end
  end

  defp execute_tool("search_products", %{"query" => query} = params) do
    limit = Map.get(params, "limit", 5)
    MyApp.Search.Server.search(query, limit: limit)
  end

  defp execute_tool("create_order", params) do
    case MyApp.Orders.create(params["user_id"], params["product_id"], params["quantity"]) do
      {:ok, order}  -> %{success: true, order_id: order.id}
      {:error, msg} -> %{error: msg}
    end
  end
end
```

---

## Step 684: RAG (Retrieval Augmented Generation)

```elixir
defmodule MyApp.AI.RAG do
  @model   "claude-opus-5-5"
  @context_docs 5

  def ask(question, namespace \\ "default") do
    # 1. Generate embedding for the question
    embedding = MyApp.AI.Embeddings.generate(question)
    
    # 2. Retrieve relevant documents
    docs = retrieve_relevant(embedding, namespace, @context_docs)
    
    # 3. Build context from docs
    context = build_context(docs)
    
    # 4. Ask Claude with context
    system = """
    You are a helpful assistant. Answer the question based on the provided context.
    If the answer is not in the context, say "I don't have information about that."
    Always cite the source document when referencing information.
    
    Context:
    #{context}
    """

    MyApp.AI.Claude.message(question, system: system)
  end

  defp retrieve_relevant(embedding, namespace, limit) do
    embedding_str = "[#{Enum.join(embedding, ",")}]"

    from(d in Document,
      where: d.namespace == ^namespace,
      order_by: fragment("embedding <=> ?::vector", ^embedding_str),
      limit: ^limit,
      select: %{id: d.id, title: d.title, content: d.content, source: d.source}
    )
    |> MyApp.Repo.all()
  end

  defp build_context(docs) do
    docs
    |> Enum.with_index(1)
    |> Enum.map_join("\n\n", fn {doc, i} ->
      """
      [Document #{i}: #{doc.title}]
      Source: #{doc.source}
      #{doc.content}
      """
    end)
  end

  def index_document(title, content, source, namespace \\ "default") do
    chunks = chunk_text(content, max_tokens: 500)
    
    Enum.each(chunks, fn chunk ->
      embedding = MyApp.AI.Embeddings.generate(chunk)
      
      %Document{}
      |> Document.changeset(%{
        title:     title,
        content:   chunk,
        source:    source,
        namespace: namespace,
        embedding: embedding
      })
      |> MyApp.Repo.insert!()
    end)
  end

  defp chunk_text(text, opts) do
    max_chars = Keyword.get(opts, :max_tokens, 500) * 4  # rough chars per token
    
    text
    |> String.split(~r/\n\n+/)
    |> Enum.reduce([[]], fn paragraph, [current | rest] ->
      joined = Enum.join(current ++ [paragraph], "\n\n")
      if String.length(joined) <= max_chars do
        [current ++ [paragraph] | rest]
      else
        [[paragraph] | [current | rest]]
      end
    end)
    |> Enum.reverse()
    |> Enum.map(&Enum.join(&1, "\n\n"))
    |> Enum.filter(&(String.length(&1) > 0))
  end
end
```

---

## Step 685: Embeddings Service

```elixir
defmodule MyApp.AI.Embeddings do
  # Use OpenAI or a local model
  @model "text-embedding-ada-002"

  def generate(text) when is_binary(text) do
    # Check cache first
    cache_key = :crypto.hash(:sha256, text) |> Base.encode16()
    
    case :ets.lookup(:embedding_cache, cache_key) do
      [{_, embedding}] -> embedding
      [] ->
        embedding = fetch_embedding(text)
        :ets.insert(:embedding_cache, {cache_key, embedding})
        embedding
    end
  end

  defp fetch_embedding(text) do
    response = Req.post!(
      "https://api.openai.com/v1/embeddings",
      json: %{model: @model, input: text},
      headers: [{"Authorization", "Bearer #{System.get_env("OPENAI_API_KEY")}"}]
    )

    response.body["data"]
    |> List.first()
    |> Map.get("embedding")
  end

  # Batch embeddings (more efficient)
  def generate_batch(texts) do
    response = Req.post!(
      "https://api.openai.com/v1/embeddings",
      json: %{model: @model, input: texts},
      headers: [{"Authorization", "Bearer #{System.get_env("OPENAI_API_KEY")}"}]
    )

    response.body["data"]
    |> Enum.sort_by(& &1["index"])
    |> Enum.map(& &1["embedding"])
  end
end
```

---

## Step 686: AI Content Moderation

```elixir
defmodule MyApp.AI.Moderation do
  def check_content(text) do
    system = """
    You are a content moderation system. Analyze the given text and respond with a JSON object:
    {
      "safe": true/false,
      "categories": ["spam", "harassment", "adult", "violence", "hate"],
      "confidence": 0.0-1.0,
      "reason": "Brief explanation if unsafe"
    }
    Only respond with valid JSON, no other text.
    """

    with {:ok, response} <- MyApp.AI.Claude.message(text, system: system, max_tokens: 200),
         {:ok, result}   <- Jason.decode(response) do
      {:ok, %{
        safe:       result["safe"],
        categories: result["categories"] || [],
        confidence: result["confidence"],
        reason:     result["reason"]
      }}
    end
  end

  def moderate_comment(comment_id) do
    comment = MyApp.get_comment!(comment_id)
    
    case check_content(comment.body) do
      {:ok, %{safe: false} = result} ->
        comment
        |> MyApp.Comment.changeset(%{
          flagged:     true,
          flag_reason: result.reason,
          hidden:      result.confidence > 0.8
        })
        |> MyApp.Repo.update!()
        
        notify_moderators(comment, result)

      {:ok, %{safe: true}} ->
        :ok

      {:error, _} ->
        :ok
    end
  end

  defp notify_moderators(comment, result) do
    MyApp.Workers.ModerationAlertWorker.new(%{
      comment_id: comment.id,
      reason:     result.reason,
      confidence: result.confidence
    })
    |> Oban.insert()
  end
end
```

---

## Step 687: AI-Powered Recommendations

```elixir
defmodule MyApp.AI.Recommendations do
  def for_user(user_id, opts \\ []) do
    limit = Keyword.get(opts, :limit, 5)
    
    # Get user's purchase history
    history = MyApp.Orders.user_product_history(user_id)
    
    if length(history) < 3 do
      # Cold start: use popular items
      MyApp.Catalog.popular_products(limit: limit)
    else
      ai_recommend(user_id, history, limit)
    end
  end

  defp ai_recommend(user_id, history, limit) do
    history_str = Enum.map_join(history, ", ", & "\"#{&1.name}\"")
    
    prompt = """
    A user has purchased these products: #{history_str}
    
    From our catalog, suggest #{limit} products they would likely enjoy.
    Respond with a JSON array of product names only.
    """

    categories = Enum.map(history, & &1.category) |> Enum.uniq()
    
    with {:ok, response} <- MyApp.AI.Claude.message(prompt),
         {:ok, names}    <- Jason.decode(response),
         products        <- MyApp.Catalog.find_by_names(names) do
      if length(products) >= limit do
        products
      else
        # Fallback: similar categories
        MyApp.Catalog.by_categories(categories, limit: limit)
      end
    else
      _ -> MyApp.Catalog.popular_products(limit: limit)
    end
  end
end
```

---

## Step 688: AI Rate Limiting

```elixir
defmodule MyApp.AI.RateLimiter do
  # Control AI API costs with per-user limits

  @daily_token_limit  50_000   # per user
  @monthly_token_limit 500_000

  def check_and_consume(user_id, estimated_tokens) do
    key_daily   = "ai_tokens:#{user_id}:#{Date.utc_today()}"
    key_monthly = "ai_tokens:#{user_id}:#{Date.utc_today().month}"

    {:ok, daily_used}   = Redix.command(:redix, ["GET", key_daily])
    {:ok, monthly_used} = Redix.command(:redix, ["GET", key_monthly])

    daily_used   = if daily_used,   do: String.to_integer(daily_used),   else: 0
    monthly_used = if monthly_used, do: String.to_integer(monthly_used), else: 0

    cond do
      daily_used + estimated_tokens > @daily_token_limit ->
        {:error, :daily_limit_exceeded}
      monthly_used + estimated_tokens > @monthly_token_limit ->
        {:error, :monthly_limit_exceeded}
      true ->
        Redix.pipeline(:redix, [
          ["INCRBY", key_daily,   estimated_tokens],
          ["EXPIRE", key_daily,   86400],
          ["INCRBY", key_monthly, estimated_tokens],
          ["EXPIRE", key_monthly, 2_592_000]
        ])
        :ok
    end
  end

  def with_limit(user_id, estimated_tokens, fun) do
    case check_and_consume(user_id, estimated_tokens) do
      :ok ->
        result = fun.()
        # Adjust based on actual usage if available
        result
      {:error, :daily_limit_exceeded} ->
        {:error, "Daily AI usage limit reached. Resets tomorrow."}
      {:error, :monthly_limit_exceeded} ->
        {:error, "Monthly AI usage limit reached."}
    end
  end
end
```

---

## Step 689: Prompt Management

```elixir
defmodule MyApp.AI.PromptLibrary do
  # Centralized prompt management

  @prompts %{
    product_description: """
    You are a copywriter for an e-commerce site. Generate a compelling product description.
    Product name: {name}
    Category: {category}
    Key features: {features}
    
    Write a 2-3 sentence description that is engaging and SEO-friendly.
    """,

    customer_reply: """
    You are a customer support agent for MyApp. Reply helpfully and professionally.
    Customer message: {message}
    Order details: {order_details}
    Company policy: 30-day returns, free shipping on orders over $50.
    """,

    seo_meta: """
    Generate SEO meta title and description for:
    Page: {page_title}
    Content: {content}
    
    Respond with JSON: {"title": "...", "description": "..."}
    Max title: 60 chars, max description: 160 chars.
    """
  }

  def get(name, vars \\ %{}) do
    template = Map.fetch!(@prompts, name)
    
    Enum.reduce(vars, template, fn {key, value}, acc ->
      String.replace(acc, "{#{key}}", to_string(value))
    end)
  end

  def generate_product_description(product) do
    prompt = get(:product_description, %{
      name:     product.name,
      category: product.category,
      features: Enum.join(product.features || [], ", ")
    })
    
    MyApp.AI.Claude.message(prompt)
  end
end
```

---

## Step 690: AI Observability

```elixir
defmodule MyApp.AI.Telemetry do
  def setup do
    :telemetry.attach_many("ai-metrics", [
      [:my_app, :ai, :request, :start],
      [:my_app, :ai, :request, :stop],
      [:my_app, :ai, :tokens, :used]
    ], &handle_event/4, nil)
  end

  def instrument(user_id, model, fun) do
    start = System.monotonic_time()
    :telemetry.execute([:my_app, :ai, :request, :start], %{}, %{user_id: user_id, model: model})

    result = fun.()

    duration = System.monotonic_time() - start
    :telemetry.execute([:my_app, :ai, :request, :stop],
      %{duration: duration},
      %{user_id: user_id, model: model, status: status_of(result)}
    )

    case result do
      {:ok, %{input_tokens: i, output_tokens: o}} ->
        :telemetry.execute([:my_app, :ai, :tokens, :used],
          %{input: i, output: o, total: i + o},
          %{user_id: user_id, model: model}
        )
      _ -> :ok
    end

    result
  end

  defp handle_event([:my_app, :ai, :request, :stop], %{duration: d}, meta, _) do
    ms = System.convert_time_unit(d, :native, :millisecond)
    Logger.info("AI request #{meta.model} #{meta.status}: #{ms}ms")
  end

  defp handle_event([:my_app, :ai, :tokens, :used], measurements, meta, _) do
    cost = calculate_cost(meta.model, measurements)
    Logger.info("AI tokens: #{measurements.total} (~$#{:erlang.float_to_binary(cost, [decimals: 4])})")
  end

  defp handle_event(_, _, _, _), do: :ok

  defp calculate_cost("claude-opus-5-5", %{input: i, output: o}) do
    (i * 15 + o * 75) / 1_000_000  # $/token approximation
  end
  defp calculate_cost(_, %{input: i, output: o}), do: (i + o) / 1_000_000

  defp status_of({:ok, _}),    do: :success
  defp status_of({:error, _}), do: :error
  defp status_of(_),           do: :unknown
end
```

---

## สรุป Part 63

✅ **Step 681** - Claude API integration  
✅ **Step 682** - Streaming responses  
✅ **Step 683** - Tool use (function calling)  
✅ **Step 684** - RAG (retrieval augmented generation)  
✅ **Step 685** - Embeddings service  
✅ **Step 686** - Content moderation  
✅ **Step 687** - AI recommendations  
✅ **Step 688** - AI rate limiting  
✅ **Step 689** - Prompt management  
✅ **Step 690** - AI observability  

➡️ [Part 64: WebRTC & Real-Time Media](./part-64-webrtc.md)
