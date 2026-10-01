# Part 45: Capstone Project - Production E-Commerce Platform (Steps 501-550)

## Step 501: Project Overview

```
Capstone: ShopEx - Production E-Commerce Platform
- Full Phoenix + LiveView frontend
- Microservices backend (Umbrella)
- Real-time features (live inventory, notifications)
- ML recommendations (Nx/Axon)
- Background jobs (Oban)
- Event Sourcing for orders
- Docker + Kubernetes deployment
- CI/CD with GitHub Actions
- Full monitoring (Prometheus, Grafana, Jaeger)

Architecture:
                        ┌─────────────┐
                        │  API Gateway │ (Phoenix)
                        └──────┬──────┘
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
      ┌───────────┐    ┌───────────────┐  ┌──────────────┐
      │ Accounts  │    │    Orders     │  │  Catalog     │
      │ Service   │    │   Service     │  │  Service     │
      └─────┬─────┘    └──────┬────────┘  └──────┬───────┘
            │                 │                   │
      ┌─────▼─────────────────▼───────────────────▼──────┐
      │              PostgreSQL + Redis                    │
      └────────────────────────────────────────────────────┘
```

---

## Step 502: Umbrella Structure

```bash
mix new shopex --umbrella
cd shopex/apps

mix phx.new api_gateway --no-ecto
mix new accounts
mix new catalog
mix new orders
mix new notifications
mix new recommendations
mix new payments

# shopex/apps/ tree
# api_gateway/     ← Phoenix router, authentication
# accounts/        ← User management
# catalog/         ← Products, categories, search
# orders/          ← Order lifecycle, event sourcing
# notifications/   ← Email, push, in-app notifications
# recommendations/ ← ML-powered product recommendations
# payments/        ← Payment processing integration
```

---

## Step 503: Accounts Service

```elixir
defmodule Accounts.Users do
  alias Accounts.{Repo, User, Token}
  
  def register(attrs) do
    Multi.new()
    |> Multi.insert(:user, User.registration_changeset(%User{}, attrs))
    |> Multi.run(:welcome_email, fn _repo, %{user: user} ->
      Notifications.send_welcome_email(user)
      {:ok, :sent}
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{user: user}} -> {:ok, user}
      {:error, :user, cs, _} -> {:error, cs}
    end
  end
  
  def authenticate(email, password) do
    with {:ok, user} <- get_by_email(email),
         true        <- User.valid_password?(user, password),
         false       <- user.locked do
      BruteForce.reset(email)
      {:ok, user}
    else
      false when is_map(get_by_email(email)) ->
        BruteForce.record_attempt(email)
        {:error, :invalid_credentials}
      _ ->
        User.dummy_check()
        {:error, :invalid_credentials}
    end
  end
  
  def generate_tokens(user) do
    access_token  = Token.generate_access(user, ttl: {15, :minute})
    refresh_token = Token.generate_refresh(user, ttl: {30, :day})
    {:ok, %{access: access_token, refresh: refresh_token}}
  end
  
  defp get_by_email(email) do
    case Repo.get_by(User, email: String.downcase(email)) do
      nil  -> {:error, :not_found}
      user -> {:ok, user}
    end
  end
end
```

---

## Step 504: Catalog Service

```elixir
defmodule Catalog.Products do
  import Ecto.Query
  alias Catalog.{Repo, Product, Category}
  
  def list(opts \\ []) do
    base_query()
    |> apply_filters(opts)
    |> apply_sorting(opts)
    |> paginate(opts)
    |> Repo.all()
  end
  
  def search(term, opts \\ []) do
    sanitized = String.replace(term, ~r/[^a-zA-Z0-9 ]/, "")
    tsquery   = "#{sanitized}:*"
    
    base_query()
    |> where([p], fragment("to_tsvector('english', ? || ' ' || ?) @@ to_tsquery(?)",
        p.name, p.description, ^tsquery))
    |> order_by([p], fragment("ts_rank(to_tsvector('english', ? || ' ' || ?), to_tsquery(?)) DESC",
        p.name, p.description, ^tsquery))
    |> paginate(opts)
    |> Repo.all()
  end
  
  def update_stock(product_id, quantity_delta) do
    Repo.transaction(fn ->
      product = Repo.get!(Product, product_id, lock: "FOR UPDATE")
      
      new_stock = product.stock + quantity_delta
      
      if new_stock < 0 do
        Repo.rollback(:insufficient_stock)
      else
        product
        |> Product.changeset(%{stock: new_stock})
        |> Repo.update!()
      end
    end)
  end
  
  defp base_query do
    from p in Product,
      join: c in Category, on: p.category_id == c.id,
      where: p.active == true,
      preload: [:category, :images]
  end
  
  defp apply_filters(query, opts) do
    query
    |> maybe_filter_category(Keyword.get(opts, :category))
    |> maybe_filter_price(Keyword.get(opts, :min_price), Keyword.get(opts, :max_price))
    |> maybe_filter_in_stock(Keyword.get(opts, :in_stock))
  end
  
  defp maybe_filter_category(q, nil), do: q
  defp maybe_filter_category(q, cat_id), do: where(q, [p], p.category_id == ^cat_id)
  
  defp maybe_filter_price(q, nil, nil), do: q
  defp maybe_filter_price(q, min, nil), do: where(q, [p], p.price >= ^min)
  defp maybe_filter_price(q, nil, max), do: where(q, [p], p.price <= ^max)
  defp maybe_filter_price(q, min, max), do: where(q, [p], p.price >= ^min and p.price <= ^max)
  
  defp maybe_filter_in_stock(q, true), do: where(q, [p], p.stock > 0)
  defp maybe_filter_in_stock(q, _), do: q
  
  defp apply_sorting(query, opts) do
    case Keyword.get(opts, :sort, :inserted_at) do
      :price_asc   -> order_by(query, [p], asc:  p.price)
      :price_desc  -> order_by(query, [p], desc: p.price)
      :name        -> order_by(query, [p], asc:  p.name)
      :newest      -> order_by(query, [p], desc: p.inserted_at)
      _default     -> order_by(query, [p], desc: p.inserted_at)
    end
  end
  
  defp paginate(query, opts) do
    page  = Keyword.get(opts, :page, 1)
    limit = Keyword.get(opts, :limit, 20)
    query |> limit(^limit) |> offset(^((page - 1) * limit))
  end
end
```

---

## Step 505: Orders Service with Event Sourcing

```elixir
defmodule Orders.Order do
  @enforce_keys [:id]
  defstruct [
    id:           nil,
    user_id:      nil,
    items:        [],
    status:       :pending,
    total:        Decimal.new("0"),
    payment_id:   nil,
    shipped_at:   nil,
    delivered_at: nil,
    events:       []
  ]
  
  # Command handling
  def place(user_id, items) do
    order = %__MODULE__{id: Ecto.UUID.generate(), user_id: user_id}
    apply_event(order, %Events.OrderPlaced{
      order_id: order.id,
      user_id:  user_id,
      items:    items,
      total:    calculate_total(items)
    })
  end
  
  def confirm(%__MODULE__{status: :pending} = order, payment_id) do
    apply_event(order, %Events.OrderConfirmed{
      order_id:   order.id,
      payment_id: payment_id,
      confirmed_at: DateTime.utc_now()
    })
  end
  def confirm(_, _), do: {:error, :invalid_state}
  
  def ship(%__MODULE__{status: :confirmed} = order, tracking_number) do
    apply_event(order, %Events.OrderShipped{
      order_id:        order.id,
      tracking_number: tracking_number,
      shipped_at:      DateTime.utc_now()
    })
  end
  def ship(_, _), do: {:error, :invalid_state}
  
  # Event application
  defp apply_event(order, event) do
    new_order = handle(order, event)
    {:ok, %{new_order | events: [event | order.events]}}
  end
  
  defp handle(order, %Events.OrderPlaced{} = e) do
    %{order | user_id: e.user_id, items: e.items, total: e.total, status: :pending}
  end
  defp handle(order, %Events.OrderConfirmed{} = e) do
    %{order | payment_id: e.payment_id, status: :confirmed}
  end
  defp handle(order, %Events.OrderShipped{} = e) do
    %{order | shipped_at: e.shipped_at, status: :shipped}
  end
  
  defp calculate_total(items) do
    Enum.reduce(items, Decimal.new("0"), fn item, acc ->
      Decimal.add(acc, Decimal.mult(item.price, Decimal.new(item.quantity)))
    end)
  end
end
```

---

## Step 506: ML Recommendations

```elixir
defmodule Recommendations.Engine do
  use GenServer
  
  @retrain_interval :timer.hours(24)
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def get_recommendations(user_id, limit \\ 10) do
    GenServer.call(__MODULE__, {:recommend, user_id, limit})
  end
  
  def init(_) do
    model = load_or_train_model()
    schedule_retrain()
    {:ok, %{model: model}}
  end
  
  def handle_call({:recommend, user_id, limit}, _from, %{model: model} = state) do
    recommendations = case get_user_history(user_id) do
      [] ->
        get_popular_products(limit)
      history ->
        collaborative_filter(model, user_id, history, limit)
    end
    
    {:reply, recommendations, state}
  end
  
  def handle_info(:retrain, state) do
    new_model = train_model()
    schedule_retrain()
    {:noreply, %{state | model: new_model}}
  end
  
  defp train_model do
    interactions = load_interactions()
    
    # Build user-item matrix
    user_ids    = interactions |> Enum.map(& &1.user_id) |> Enum.uniq() |> Enum.sort()
    product_ids = interactions |> Enum.map(& &1.product_id) |> Enum.uniq() |> Enum.sort()
    
    n_users    = length(user_ids)
    n_products = length(product_ids)
    
    user_index    = Enum.with_index(user_ids) |> Map.new()
    product_index = Enum.with_index(product_ids) |> Map.new()
    
    matrix = Nx.broadcast(Nx.tensor(0, type: {:f, 32}), {n_users, n_products})
    
    matrix = Enum.reduce(interactions, matrix, fn inter, mat ->
      u = user_index[inter.user_id]
      p = product_index[inter.product_id]
      Nx.put_slice(mat, [u, p], Nx.tensor([[1.0]]))
    end)
    
    %{
      matrix:        matrix,
      user_index:    user_index,
      product_index: product_index,
      product_ids:   product_ids
    }
  end
  
  defp collaborative_filter(%{matrix: matrix} = model, user_id, _history, limit) do
    u = model.user_index[user_id]
    
    if u do
      user_vector = matrix[u]
      
      # Find similar users using cosine similarity
      similarity = Nx.divide(
        Nx.dot(matrix, Nx.transpose(user_vector)),
        Nx.multiply(
          Nx.sum(Nx.power(matrix, 2), axes: [1]) |> Nx.sqrt(),
          Nx.sum(Nx.power(user_vector, 2)) |> Nx.sqrt()
        )
      )
      
      # Weighted sum of items
      scores = Nx.dot(similarity, matrix)
      
      # Mask already seen
      unseen_scores = Nx.subtract(scores, Nx.multiply(user_vector, 1_000_000))
      
      # Top N product indices
      top_indices = unseen_scores
        |> Nx.argsort(direction: :desc)
        |> Nx.slice([0], [limit])
        |> Nx.to_flat_list()
      
      Enum.map(top_indices, fn i -> Enum.at(model.product_ids, i) end)
    else
      get_popular_products(limit)
    end
  end
  
  defp get_popular_products(limit) do
    Catalog.Products.list(sort: :popular, limit: limit)
    |> Enum.map(& &1.id)
  end
  
  defp schedule_retrain do
    Process.send_after(self(), :retrain, @retrain_interval)
  end
end
```

---

## Step 507: Phoenix LiveView Frontend

```elixir
defmodule ApiGatewayWeb.ShopLive do
  use ApiGatewayWeb, :live_view
  
  alias ApiGateway.{Cart, Catalog, Recommendations}
  
  def mount(_params, session, socket) do
    user = get_user(session)
    
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "inventory:updates")
    end
    
    {:ok, assign(socket,
      products:        Catalog.list_featured(),
      recommendations: Recommendations.for_user(user && user.id),
      cart:            Cart.get(user && user.id),
      page:            1,
      loading:         false
    )}
  end
  
  def handle_event("add_to_cart", %{"product_id" => product_id}, socket) do
    user = socket.assigns.current_user
    
    case Cart.add_item(user.id, product_id, 1) do
      {:ok, cart} ->
        {:noreply,
          socket
          |> assign(:cart, cart)
          |> put_flash(:info, "Added to cart!")
        }
      
      {:error, :out_of_stock} ->
        {:noreply, put_flash(socket, :error, "Out of stock!")}
    end
  end
  
  def handle_event("load_more", _params, socket) do
    page     = socket.assigns.page + 1
    products = Catalog.list(page: page)
    
    {:noreply,
      socket
      |> assign(page: page)
      |> stream(:products, products)
    }
  end
  
  def handle_info({:inventory_update, %{product_id: id, stock: stock}}, socket) do
    {:noreply, push_event(socket, "inventory_update", %{product_id: id, stock: stock})}
  end
  
  def render(assigns) do
    ~H"""
    <div id="shop">
      <.live_component module={CartWidget} id="cart" cart={@cart} />
      
      <div class="product-grid" id="products" phx-update="stream">
        <.product_card
          :for={{dom_id, product} <- @streams.products}
          id={dom_id}
          product={product}
        />
      </div>
      
      <div id="recommendations" class="mt-8">
        <h2>Recommended for You</h2>
        <.product_card :for={p <- @recommendations} id={"rec-#{p.id}"} product={p} />
      </div>
      
      <button phx-click="load_more" class="load-more-btn">
        Load More
      </button>
    </div>
    """
  end
end
```

---

## Step 508: Background Jobs

```elixir
# Order processing pipeline
defmodule Orders.Workers.ProcessOrder do
  use Oban.Worker, queue: :orders, max_attempts: 3
  
  @impl Oban.Worker
  def perform(%{args: %{"order_id" => order_id}}) do
    with {:ok, order} <- Orders.get(order_id),
         {:ok, _}     <- reserve_inventory(order),
         {:ok, _}     <- charge_payment(order),
         {:ok, _}     <- confirm_order(order),
         :ok          <- send_confirmation(order) do
      :ok
    else
      {:error, :out_of_stock} ->
        Orders.cancel(order_id, "Out of stock")
        {:cancel, "Inventory unavailable"}
      
      {:error, :payment_failed} ->
        Orders.cancel(order_id, "Payment failed")
        {:cancel, "Payment declined"}
      
      {:error, reason} ->
        {:error, reason}
    end
  end
end

# Email notifications
defmodule Notifications.Workers.SendEmail do
  use Oban.Worker, queue: :emails, max_attempts: 5
  
  @impl Oban.Worker
  def perform(%{args: %{"type" => type, "user_id" => user_id, "data" => data}}) do
    user = Accounts.get_user!(user_id)
    
    case type do
      "welcome"           -> Notifications.Email.welcome(user)
      "order_confirmed"   -> Notifications.Email.order_confirmed(user, data)
      "order_shipped"     -> Notifications.Email.order_shipped(user, data)
      "password_reset"    -> Notifications.Email.password_reset(user, data)
    end
    |> Swoosh.Mailer.deliver()
    |> then(fn
      {:ok, _}    -> :ok
      {:error, r} -> {:error, r}
    end)
  end
end
```

---

## Step 509: Docker + Kubernetes

```yaml
# docker-compose.yml
version: "3.8"

services:
  api_gateway:
    build:
      context: .
      target: prod
    ports:
      - "4000:4000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@postgres/shopex
      - REDIS_URL=redis://redis:6379
      - SECRET_KEY_BASE=${SECRET_KEY_BASE}
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: shopex
      POSTGRES_PASSWORD: postgres
    volumes:
      - pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

volumes:
  pg_data:
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopex-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shopex-api
  template:
    metadata:
      labels:
        app: shopex-api
    spec:
      containers:
      - name: shopex
        image: ghcr.io/myorg/shopex:latest
        ports:
        - containerPort: 4000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: shopex-secrets
              key: database-url
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /health
            port: 4000
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 4000
          initialDelaySeconds: 30
          periodSeconds: 10
```

---

## Step 510: CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
    
    steps:
    - uses: actions/checkout@v4
    - uses: erlef/setup-beam@v1
      with:
        otp-version: "26.2"
        elixir-version: "1.16"
    - run: mix deps.get
    - run: mix test --cover
  
  build:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Build Docker image
      run: |
        docker build -t ghcr.io/myorg/shopex:${{ github.sha }} .
        docker push ghcr.io/myorg/shopex:${{ github.sha }}
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/shopex-api \
          shopex=ghcr.io/myorg/shopex:${{ github.sha }}
        kubectl rollout status deployment/shopex-api --timeout=300s
    
    - name: Run database migrations
      run: |
        kubectl exec deploy/shopex-api -- bin/shopex eval "Shopex.Release.migrate()"
    
    - name: Smoke test
      run: |
        curl -f https://shopex.example.com/health
```

---

## สรุป Part 45

✅ **Step 501** - Project architecture  
✅ **Step 502** - Umbrella structure  
✅ **Step 503** - Accounts service  
✅ **Step 504** - Catalog service  
✅ **Step 505** - Orders + event sourcing  
✅ **Step 506** - ML recommendations  
✅ **Step 507** - LiveView frontend  
✅ **Step 508** - Background jobs  
✅ **Step 509** - Docker + Kubernetes  
✅ **Step 510** - CI/CD pipeline  

➡️ [Part 46: Advanced OTP Patterns](./part-46-advanced-otp.md)
