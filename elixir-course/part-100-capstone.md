# Part 100: Course Capstone — Full-Stack Elixir Application (Steps 1061-1070)

> ขอแสดงความยินดี! คุณได้มาถึง Part 100 ซึ่งเป็นจุดสิ้นสุดของหลักสูตร Elixir ระดับโลก 🎉

---

## Step 1061: Project Structure — E-Commerce Platform

```
my_shop/
├── apps/
│   ├── my_shop/              # Core business logic (umbrella child)
│   │   ├── lib/
│   │   │   ├── my_shop/
│   │   │   │   ├── accounts/     # Users, auth
│   │   │   │   ├── catalog/      # Products, categories
│   │   │   │   ├── orders/       # Orders, cart
│   │   │   │   ├── payments/     # Stripe integration
│   │   │   │   ├── inventory/    # Stock management
│   │   │   │   ├── notifications/# Email, SMS, push
│   │   │   │   ├── analytics/    # Tracking, reports
│   │   │   │   └── workers/      # Oban background jobs
│   │   └── test/
│   └── my_shop_web/          # Phoenix web layer
│       ├── lib/
│       │   ├── my_shop_web/
│       │   │   ├── controllers/  # REST API
│       │   │   ├── live/         # LiveView pages
│       │   │   ├── channels/     # WebSockets
│       │   │   ├── components/   # Reusable UI components
│       │   │   └── plugs/        # Custom plugs
│       └── test/
├── config/
│   ├── config.exs
│   ├── dev.exs
│   ├── prod.exs
│   └── runtime.exs
├── priv/
│   ├── repo/migrations/
│   └── static/
└── mix.exs
```

---

## Step 1062: Core Domain — Catalog Context

```elixir
defmodule MyShop.Catalog do
  @moduledoc "Product catalog management"

  alias MyShop.Catalog.{Product, Category, Variant}
  alias MyShop.Repo
  import Ecto.Query

  def list_products(opts \\ []) do
    filter = Keyword.get(opts, :filter, %{})
    page   = Keyword.get(opts, :page, 1)
    limit  = Keyword.get(opts, :per_page, 24)

    Product
    |> apply_filters(filter)
    |> preload([:category, :variants, :images])
    |> order_by(desc: :inserted_at)
    |> paginate(page, limit)
    |> Repo.all()
  end

  def get_product!(slug) do
    Product
    |> where(slug: ^slug, status: :active)
    |> preload([:category, variants: :option_values, images: [], reviews: :user])
    |> Repo.one!()
  end

  def create_product(attrs) do
    %Product{}
    |> Product.changeset(attrs)
    |> Repo.insert()
    |> tap(fn
      {:ok, product} -> broadcast_product_update(:created, product)
      _              -> :ok
    end)
  end

  def update_product(%Product{} = product, attrs) do
    product
    |> Product.changeset(attrs)
    |> Repo.update()
    |> tap(fn
      {:ok, product} -> broadcast_product_update(:updated, product)
      _              -> :ok
    end)
  end

  def search(query, opts \\ []) do
    limit = Keyword.get(opts, :limit, 10)

    from(p in Product,
      where: fragment("to_tsvector('english', ? || ' ' || ?) @@ plainto_tsquery('english', ?)",
               p.name, p.description, ^query),
      where: p.status == :active,
      order_by: fragment("ts_rank(to_tsvector('english', ? || ' ' || ?), plainto_tsquery('english', ?)) DESC",
                  p.name, p.description, ^query),
      limit: ^limit,
      preload: [:images]
    ) |> Repo.all()
  end

  defp apply_filters(query, %{"category" => slug} = filter) when is_binary(slug) do
    query
    |> join(:inner, [p], c in Category, on: c.id == p.category_id and c.slug == ^slug)
    |> apply_filters(Map.delete(filter, "category"))
  end
  defp apply_filters(query, %{"min_price" => min} = filter) do
    query |> where([p], p.price >= ^min) |> apply_filters(Map.delete(filter, "min_price"))
  end
  defp apply_filters(query, %{"max_price" => max} = filter) do
    query |> where([p], p.price <= ^max) |> apply_filters(Map.delete(filter, "max_price"))
  end
  defp apply_filters(query, _), do: query

  defp paginate(query, page, limit) do
    query |> limit(^limit) |> offset(^((page - 1) * limit))
  end

  defp broadcast_product_update(event, product) do
    Phoenix.PubSub.broadcast(MyShop.PubSub, "catalog", {event, product})
  end
end
```

---

## Step 1063: Orders Context — Shopping Cart & Checkout

```elixir
defmodule MyShop.Orders do
  alias MyShop.Orders.{Cart, Order, OrderItem, LineItem}
  alias MyShop.{Repo, Catalog, Inventory}
  import Ecto.Query

  # --- Cart ---

  def get_or_create_cart(user_id) do
    case Repo.get_by(Cart, user_id: user_id, status: :active) do
      nil  -> Repo.insert!(%Cart{user_id: user_id})
      cart -> cart
    end
  end

  def add_to_cart(%Cart{} = cart, variant_id, quantity) do
    with {:ok, _} <- Inventory.check_availability(variant_id, quantity) do
      case Repo.get_by(LineItem, cart_id: cart.id, variant_id: variant_id) do
        nil ->
          variant = Catalog.get_variant!(variant_id)
          Repo.insert(%LineItem{
            cart_id: cart.id, variant_id: variant_id,
            quantity: quantity, price: variant.price
          })
        item ->
          item |> Ecto.Changeset.change(quantity: item.quantity + quantity) |> Repo.update()
      end
    end
  end

  # --- Checkout ---

  def checkout(%Cart{} = cart, payment_attrs) do
    Repo.transaction(fn ->
      cart_with_items = Repo.preload(cart, [items: :variant])

      # Reserve inventory
      Enum.each(cart_with_items.items, fn item ->
        case Inventory.reserve(item.variant_id, item.quantity) do
          {:ok, _} -> :ok
          {:error, r} -> Repo.rollback({:inventory, r})
        end
      end)

      # Create order
      total = Enum.sum(Enum.map(cart_with_items.items, &(&1.price * &1.quantity)))
      
      order = Repo.insert!(%Order{
        user_id:    cart.user_id,
        status:     :pending,
        total:      total,
        currency:   "USD",
        items:      build_order_items(cart_with_items.items)
      })

      # Process payment
      case MyShop.Payments.charge(order, payment_attrs) do
        {:ok, payment} ->
          order |> Ecto.Changeset.change(status: :confirmed, payment_id: payment.id) |> Repo.update!()
          Repo.update!(Ecto.Changeset.change(cart, status: :checked_out))
          order
        {:error, reason} ->
          Repo.rollback({:payment, reason})
      end
    end)
  end

  defp build_order_items(line_items) do
    Enum.map(line_items, fn item ->
      %OrderItem{
        variant_id: item.variant_id,
        quantity:   item.quantity,
        price:      item.price,
        total:      item.price * item.quantity
      }
    end)
  end
end
```

---

## Step 1064: LiveView — Real-Time Product Page

```elixir
defmodule MyShopWeb.ProductLive.Show do
  use MyShopWeb, :live_view
  alias MyShop.Catalog

  def mount(%{"slug" => slug}, session, socket) do
    product = Catalog.get_product!(slug)
    
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyShop.PubSub, "product:#{product.id}")
    end

    socket =
      socket
      |> assign(:product, product)
      |> assign(:selected_variant, List.first(product.variants))
      |> assign(:quantity, 1)
      |> assign(:current_user, get_user_from_session(session))

    {:ok, socket}
  end

  def handle_params(%{"variant" => variant_id}, _uri, socket) do
    variant = Enum.find(socket.assigns.product.variants, &(to_string(&1.id) == variant_id))
    {:noreply, assign(socket, :selected_variant, variant || socket.assigns.selected_variant)}
  end
  def handle_params(_params, _uri, socket), do: {:noreply, socket}

  def handle_event("select_variant", %{"id" => id}, socket) do
    {:noreply, push_patch(socket, to: ~p"/products/#{socket.assigns.product.slug}?variant=#{id}")}
  end

  def handle_event("update_quantity", %{"quantity" => qty}, socket) do
    quantity = String.to_integer(qty) |> max(1) |> min(socket.assigns.selected_variant.stock)
    {:noreply, assign(socket, :quantity, quantity)}
  end

  def handle_event("add_to_cart", _params, socket) do
    %{selected_variant: variant, quantity: qty, current_user: user} = socket.assigns

    case MyShop.Orders.add_to_cart_for_user(user.id, variant.id, qty) do
      {:ok, _} ->
        {:noreply, socket |> put_flash(:info, "Added to cart!") |> push_event("cart_updated", %{})}
      {:error, :out_of_stock} ->
        {:noreply, put_flash(socket, :error, "Sorry, not enough stock")}
    end
  end

  # Real-time stock update
  def handle_info({:stock_updated, variant_id, new_stock}, socket) do
    product = update_in(socket.assigns.product.variants, fn variants ->
      Enum.map(variants, fn v ->
        if v.id == variant_id, do: %{v | stock: new_stock}, else: v
      end)
    end)
    {:noreply, assign(socket, :product, product)}
  end

  def render(assigns) do
    ~H"""
    <div class="product-page">
      <div class="product-images">
        <%= for image <- @product.images do %>
          <img src={image.url} alt={@product.name} />
        <% end %>
      </div>
      
      <div class="product-info">
        <h1><%= @product.name %></h1>
        <p class="price">$<%= format_price(@selected_variant.price) %></p>
        
        <div class="variants">
          <%= for variant <- @product.variants do %>
            <button
              phx-click="select_variant"
              phx-value-id={variant.id}
              class={["variant-btn", @selected_variant.id == variant.id && "selected"]}
              disabled={variant.stock == 0}
            >
              <%= variant.name %>
              <%= if variant.stock == 0 do %>
                <span class="out-of-stock">(Out of Stock)</span>
              <% end %>
            </button>
          <% end %>
        </div>

        <div class="quantity">
          <input type="number" value={@quantity} min="1" max={@selected_variant.stock}
                 phx-change="update_quantity" name="quantity" />
          <span>of <%= @selected_variant.stock %> available</span>
        </div>

        <button phx-click="add_to_cart" class="btn-primary"
                disabled={@selected_variant.stock == 0}>
          Add to Cart
        </button>
      </div>
    </div>
    """
  end

  defp format_price(cents), do: :erlang.float_to_binary(cents / 100, decimals: 2)
  defp get_user_from_session(%{"user_token" => token}), do: MyShop.Accounts.get_user_by_session_token(token)
  defp get_user_from_session(_), do: nil
end
```

---

## Step 1065: Admin Dashboard LiveView

```elixir
defmodule MyShopWeb.AdminLive.Dashboard do
  use MyShopWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyShop.PubSub, "orders")
      :timer.send_interval(30_000, :refresh_stats)
    end

    {:ok, assign(socket, stats: load_stats(), recent_orders: load_recent_orders())}
  end

  def handle_info(:refresh_stats, socket) do
    {:noreply, assign(socket, stats: load_stats())}
  end

  def handle_info({:order_created, order}, socket) do
    {:noreply, update(socket, :recent_orders, fn orders -> [order | Enum.take(orders, 9)] end)}
  end

  defp load_stats do
    now   = DateTime.utc_now()
    today = DateTime.to_date(now)

    %{
      daily_revenue:    MyShop.Analytics.revenue_for_period(today, today),
      weekly_revenue:   MyShop.Analytics.revenue_for_period(Date.add(today, -7), today),
      active_users:     MyShop.Analytics.active_users_today(),
      pending_orders:   MyShop.Orders.count_by_status(:pending),
      low_stock_items:  MyShop.Inventory.low_stock_count(threshold: 5),
      conversion_rate:  MyShop.Analytics.conversion_rate(days: 7)
    }
  end

  defp load_recent_orders do
    MyShop.Orders.recent(limit: 10)
  end
end
```

---

## Step 1066: Testing Strategy

```elixir
# Integration test — full checkout flow
defmodule MyShop.CheckoutIntegrationTest do
  use MyShop.DataCase, async: true
  import MyShop.Factory

  setup do
    user     = insert(:user)
    category = insert(:category)
    product  = insert(:product, category: category, status: :active)
    variant  = insert(:variant, product: product, price: 2999, stock: 10)
    %{user: user, variant: variant}
  end

  test "complete checkout flow", %{user: user, variant: variant} do
    # 1. Create cart and add items
    cart = MyShop.Orders.get_or_create_cart(user.id)
    assert {:ok, _} = MyShop.Orders.add_to_cart(cart, variant.id, 2)

    # Verify inventory reserved
    assert MyShop.Inventory.get_reserved(variant.id) == 2

    # 2. Checkout
    assert {:ok, order} = MyShop.Orders.checkout(cart, %{
      payment_method: "pm_card_visa",
      shipping_address: %{
        line1: "123 Main St",
        city: "San Francisco",
        state: "CA",
        zip: "94105",
        country: "US"
      }
    })

    # 3. Verify order state
    assert order.status == :confirmed
    assert order.total == 5998  # 2 × $29.99

    # 4. Verify inventory updated
    updated_variant = MyShop.Repo.get!(MyShop.Catalog.Variant, variant.id)
    assert updated_variant.stock == 8
    assert MyShop.Inventory.get_reserved(variant.id) == 0

    # 5. Verify cart closed
    cart = MyShop.Repo.get!(MyShop.Orders.Cart, cart.id)
    assert cart.status == :checked_out

    # 6. Verify confirmation email queued
    assert_enqueued worker: MyShop.Workers.EmailWorker,
                    args: %{"type" => "order_confirmation", "order_id" => order.id}
  end
end

# LiveView integration test
defmodule MyShopWeb.ProductLiveTest do
  use MyShopWeb.ConnCase, async: true
  import Phoenix.LiveViewTest
  import MyShop.Factory

  test "shows product and allows adding to cart", %{conn: conn} do
    product = insert(:product, status: :active)
    variant = insert(:variant, product: product, stock: 5)
    user    = insert(:user)

    conn    = log_in_user(conn, user)
    {:ok, view, _html} = live(conn, ~p"/products/#{product.slug}")

    assert has_element?(view, "h1", product.name)
    assert has_element?(view, ".variant-btn", variant.name)

    view |> element(".btn-primary") |> render_click()
    assert has_element?(view, ".flash-info", "Added to cart!")
  end
end
```

---

## Step 1067: Performance Optimization

```elixir
defmodule MyShop.Performance do
  # 1. Database query optimization with preloading and N+1 prevention
  def list_products_optimized do
    MyShop.Repo.all(
      from p in MyShop.Catalog.Product,
        where: p.status == :active,
        preload: [
          :category,
          variants: ^from(v in MyShop.Catalog.Variant, where: v.active == true, order_by: v.position),
          images: ^from(i in MyShop.Catalog.Image, where: i.primary == true, limit: 1)
        ]
    )
  end

  # 2. ETS-based caching for hot data
  def cached_categories do
    case :ets.lookup(:category_cache, :all) do
      [{:all, categories, cached_at}] when :erlang.monotonic_time(:second) - cached_at < 300 ->
        categories
      _ ->
        categories = MyShop.Catalog.list_categories()
        :ets.insert(:category_cache, {:all, categories, :erlang.monotonic_time(:second)})
        categories
    end
  end

  # 3. Batch database insertions
  def bulk_import_products(products_data) do
    now = NaiveDateTime.utc_now() |> NaiveDateTime.truncate(:second)
    
    rows = Enum.map(products_data, fn p ->
      %{
        name:        p.name,
        description: p.description,
        price:       p.price,
        slug:        Slug.slugify(p.name),
        status:      :draft,
        inserted_at: now,
        updated_at:  now
      }
    end)

    MyShop.Repo.insert_all(MyShop.Catalog.Product, rows,
      on_conflict: {:replace, [:name, :price, :updated_at]},
      conflict_target: :slug,
      returning: [:id]
    )
  end

  # 4. Connection pooling tuning (config/runtime.exs)
  def pool_config do
    %{
      pool_size:    System.get_env("DB_POOL_SIZE", "20") |> String.to_integer(),
      queue_target: 50,    # ms — start queueing if DB takes longer
      queue_interval: 1000 # ms — check queue depth every 1s
    }
  end
end
```

---

## Step 1068: Production Readiness Checklist

```elixir
defmodule MyShop.HealthCheck do
  @moduledoc "Production readiness health checks"

  def status do
    checks = %{
      database:     check_database(),
      redis:        check_redis(),
      oban:         check_oban(),
      disk_space:   check_disk(),
      memory:       check_memory()
    }

    overall = if Enum.all?(Map.values(checks), &(&1 == :ok)), do: :healthy, else: :degraded
    %{status: overall, checks: checks, timestamp: DateTime.utc_now()}
  end

  defp check_database do
    case MyShop.Repo.query("SELECT 1") do
      {:ok, _}    -> :ok
      {:error, _} -> :error
    end
  end

  defp check_redis do
    case Redix.command(:redix, ["PING"]) do
      {:ok, "PONG"} -> :ok
      _             -> :error
    end
  end

  defp check_oban do
    case Oban.check_queue(:default) do
      {:ok, _} -> :ok
      _        -> :error
    end
  end

  defp check_disk do
    {output, 0} = System.cmd("df", ["-h", "/"])
    lines = String.split(output, "\n")
    case Enum.at(lines, 1) do
      nil -> :error
      line ->
        pct = line |> String.split() |> Enum.at(4) |> String.trim_trailing("%") |> String.to_integer()
        if pct < 90, do: :ok, else: :warning
    end
  end

  defp check_memory do
    memory = :erlang.memory(:total)
    limit  = 2 * 1024 * 1024 * 1024  # 2GB
    if memory < limit, do: :ok, else: :warning
  end
end

# Health endpoint
defmodule MyShopWeb.HealthController do
  use MyShopWeb, :controller

  def live(conn, _params) do
    json(conn, %{status: "ok"})
  end

  def ready(conn, _params) do
    case MyShop.HealthCheck.status() do
      %{status: :healthy} = status ->
        json(conn, status)
      status ->
        conn |> put_status(503) |> json(status)
    end
  end
end
```

---

## Step 1069: Deployment Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    uses: ./.github/workflows/ci.yml

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: |
          docker build \
            --tag myregistry.io/my-shop:${{ github.sha }} \
            --tag myregistry.io/my-shop:latest \
            .

      - name: Push to registry
        run: |
          docker push myregistry.io/my-shop:${{ github.sha }}
          docker push myregistry.io/my-shop:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Run database migrations
        run: |
          kubectl create job migrate-${{ github.sha }} \
            --from=cronjob/migrate \
            --namespace production
          kubectl wait job/migrate-${{ github.sha }} \
            --for=condition=complete --timeout=120s

      - name: Rolling deployment
        run: |
          kubectl set image deployment/my-shop \
            my-shop=myregistry.io/my-shop:${{ github.sha }} \
            --namespace production
          kubectl rollout status deployment/my-shop \
            --namespace production --timeout=300s

      - name: Post-deployment verification
        run: |
          sleep 30
          curl -f https://myshop.com/health/ready || exit 1
```

---

## Step 1070: Course Summary & What's Next

```
╔══════════════════════════════════════════════════════════════╗
║          ELIXIR COURSE COMPLETION SUMMARY                    ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  Total Steps:   1070 (Steps 1-1070)                         ║
║  Total Parts:   100 (Parts 1-100)                           ║
║  Skill Level:   World-Class / Production-Ready              ║
║                                                              ║
╠══════════════════════════════════════════════════════════════╣
║  SKILLS MASTERED                                             ║
╠══════════════════════════════════════════════════════════════╣
║  ✅ Elixir fundamentals & functional programming             ║
║  ✅ OTP (GenServer, Supervisor, Application)                 ║
║  ✅ Phoenix Framework (controllers, LiveView, channels)      ║
║  ✅ Ecto & database patterns                                 ║
║  ✅ Concurrency & distributed systems                        ║
║  ✅ Metaprogramming & macros                                 ║
║  ✅ Testing (ExUnit, property testing, integration)          ║
║  ✅ Performance optimization                                 ║
║  ✅ Security hardening (OWASP, encryption, auth)             ║
║  ✅ API design (REST, GraphQL, WebSockets)                   ║
║  ✅ File storage & media handling                            ║
║  ✅ Email, SMS, push notifications                           ║
║  ✅ Authentication (JWT, OAuth, TOTP, passkeys)              ║
║  ✅ Search (Elasticsearch, pgvector, autocomplete)           ║
║  ✅ Payments (Stripe, subscriptions, fraud)                  ║
║  ✅ Analytics & real-time dashboards                         ║
║  ✅ Nerves (embedded systems)                                ║
║  ✅ LiveBook & data science                                  ║
║  ✅ CI/CD & Kubernetes deployment                            ║
║  ✅ Advanced patterns (Event Sourcing, CQRS, Saga)           ║
║  ✅ Ecosystem libraries (Ash, Commanded, Broadway)           ║
╠══════════════════════════════════════════════════════════════╣
║  WHAT'S NEXT                                                 ║
╠══════════════════════════════════════════════════════════════╣
║  → Contribute to open-source Elixir libraries               ║
║  → Join ElixirForum (elixirforum.com)                       ║
║  → Follow Elixir on GitHub (github.com/elixir-lang)         ║
║  → Explore Nx/Axon for AI/ML workloads                      ║
║  → Build with Membrane (multimedia pipelines)               ║
║  → Explore Livebook Teams for collaborative notebooks       ║
║  → Join local Elixir/Erlang meetups                         ║
║  → Read "Programming Elixir" by Dave Thomas                 ║
║  → Read "Designing Elixir Systems with OTP" by James Gray  ║
╚══════════════════════════════════════════════════════════════╝
```

---

## สรุป Part 100 — Course Complete!

✅ **Step 1061** - Project structure (e-commerce umbrella)  
✅ **Step 1062** - Core domain — catalog context  
✅ **Step 1063** - Orders context — cart & checkout  
✅ **Step 1064** - LiveView — real-time product page  
✅ **Step 1065** - Admin dashboard LiveView  
✅ **Step 1066** - Testing strategy  
✅ **Step 1067** - Performance optimization  
✅ **Step 1068** - Production readiness  
✅ **Step 1069** - Deployment pipeline  
✅ **Step 1070** - Course summary & what's next  

---

## 🎓 จบหลักสูตร Elixir ระดับโลก

คุณได้เรียนรู้ทุกอย่างตั้งแต่พื้นฐานจนถึงระดับ Production-Ready ครบทั้ง **100 Parts** และ **1070 Steps**

**ยินดีด้วย! คุณเป็น Elixir Developer ระดับโลกแล้ว! 🌏**
