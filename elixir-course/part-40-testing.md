# Part 40: Testing Mastery (Steps 441-460)

## Step 441: ExUnit Advanced

```elixir
# Test tags and filtering
defmodule MyApp.SlowTest do
  use ExUnit.Case
  
  @moduletag :slow  # tag whole module
  
  @tag :slow
  @tag :integration
  test "expensive operation" do
    # ...
  end
end

# mix test --exclude slow
# mix test --only integration
# mix test --include slow

# Custom assertions
defmodule MyApp.Assertions do
  import ExUnit.Assertions
  
  def assert_changeset_valid(cs) do
    assert cs.valid?, "Expected changeset to be valid, got errors: #{inspect(cs.errors)}"
    cs
  end
  
  def assert_changeset_error(cs, field, message) do
    errors = Ecto.Changeset.traverse_errors(cs, fn {msg, opts} ->
      Enum.reduce(opts, msg, fn {key, value}, acc ->
        String.replace(acc, "%{#{key}}", to_string(value))
      end)
    end)
    
    actual_messages = Map.get(errors, field, [])
    assert message in actual_messages,
      "Expected #{field} to have error #{inspect(message)}, got: #{inspect(actual_messages)}"
  end
  
  def assert_ok({:ok, value}), do: value
  def assert_ok({:error, reason}) do
    flunk("Expected {:ok, _}, got {:error, #{inspect(reason)}}")
  end
  
  def assert_error({:error, reason}), do: reason
  def assert_error({:ok, value}) do
    flunk("Expected {:error, _}, got {:ok, #{inspect(value)}}")
  end
end
```

---

## Step 442: Factory ด้วย ExMachina

```elixir
# mix.exs
{:ex_machina, "~> 2.7", only: :test}

defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo
  
  def user_factory do
    %MyApp.Accounts.User{
      name:           sequence(:name, &"User #{&1}"),
      email:          sequence(:email, &"user#{&1}@example.com"),
      password_hash:  Bcrypt.hash_pwd_salt("password123!"),
      role:           :user,
      confirmed_at:   DateTime.utc_now()
    }
  end
  
  def admin_factory do
    struct!(user_factory(), role: :admin)
  end
  
  def product_factory do
    %MyApp.Catalog.Product{
      name:        sequence(:product_name, &"Product #{&1}"),
      description: "A great product",
      price:       Decimal.new("29.99"),
      stock:       100,
      category:    build(:category)
    }
  end
  
  def category_factory do
    %MyApp.Catalog.Category{
      name: sequence(:category_name, &"Category #{&1}")
    }
  end
  
  def order_factory do
    user = build(:user)
    
    %MyApp.Orders.Order{
      user:         user,
      status:       :pending,
      total:        Decimal.new("99.99"),
      items:        build_list(2, :order_item)
    }
  end
  
  def order_item_factory do
    %MyApp.Orders.OrderItem{
      product:  build(:product),
      quantity: 1,
      price:    Decimal.new("29.99")
    }
  end
end

# Usage in tests
use MyApp.DataCase
import MyApp.Factory

test "creates order" do
  user    = insert(:user)
  product = insert(:product, price: Decimal.new("50.00"))
  
  {:ok, order} = Orders.create_order(%{
    user_id: user.id,
    items: [%{product_id: product.id, quantity: 2}]
  })
  
  assert order.total == Decimal.new("100.00")
end
```

---

## Step 443: Mox (Mocking)

```elixir
# mix.exs
{:mox, "~> 1.0", only: :test}

# Define behaviour
defmodule MyApp.PaymentGateway do
  @callback charge(amount :: Decimal.t(), card :: map()) ::
    {:ok, transaction_id :: String.t()} | {:error, reason :: String.t()}
  
  @callback refund(transaction_id :: String.t()) ::
    {:ok, refund_id :: String.t()} | {:error, reason :: String.t()}
end

# Real implementation
defmodule MyApp.StripeGateway do
  @behaviour MyApp.PaymentGateway
  
  def charge(amount, card) do
    # Real Stripe API call
  end
end

# test/support/mocks.ex
Mox.defmock(MyApp.MockPaymentGateway, for: MyApp.PaymentGateway)

# config/test.exs
config :my_app, payment_gateway: MyApp.MockPaymentGateway

# In tests
defmodule MyApp.OrdersTest do
  use MyApp.DataCase
  import Mox
  
  # Allow mocks to be called from any process
  setup :verify_on_exit!
  
  test "processes payment on order creation" do
    user    = insert(:user)
    product = insert(:product, price: Decimal.new("50.00"))
    
    expect(MyApp.MockPaymentGateway, :charge, fn amount, _card ->
      assert amount == Decimal.new("50.00")
      {:ok, "txn_123"}
    end)
    
    {:ok, order} = Orders.create_order(%{
      user_id: user.id,
      items: [%{product_id: product.id, quantity: 1}],
      payment: %{card: "tok_visa"}
    })
    
    assert order.payment_id == "txn_123"
  end
  
  test "handles payment failure" do
    stub(MyApp.MockPaymentGateway, :charge, fn _amount, _card ->
      {:error, "Card declined"}
    end)
    
    assert {:error, :payment_failed} = Orders.create_order(%{...})
  end
end
```

---

## Step 444: Property-Based Testing (StreamData)

```elixir
# mix.exs
{:stream_data, "~> 0.6", only: [:dev, :test]}

defmodule MyApp.PropertyTest do
  use ExUnit.Case
  use ExUnitProperties
  
  property "encoding and decoding is identity" do
    check all string <- string(:alphanumeric, min_length: 1, max_length: 100) do
      assert {:ok, decoded} = Base.decode64(Base.encode64(string))
      assert decoded == string
    end
  end
  
  property "pagination always returns correct subset" do
    check all items <- list_of(integer(), min_length: 0, max_length: 1000),
              page  <- integer(1..100),
              limit <- integer(1..50) do
      
      result = paginate(items, page, limit)
      
      expected_start = (page - 1) * limit
      expected_end   = min(page * limit, length(items))
      expected        = Enum.slice(items, expected_start, limit)
      
      assert result == expected
      assert length(result) <= limit
    end
  end
  
  property "sorted list is always sorted" do
    check all list <- list_of(integer()) do
      sorted = Enum.sort(list)
      
      assert sorted == Enum.sort(sorted)  # idempotent
      assert length(sorted) == length(list)
      
      Enum.each(Enum.zip(sorted, tl(sorted)), fn {a, b} ->
        assert a <= b
      end)
    end
  end
  
  # Custom generators
  def user_params do
    gen all name  <- string(:alphanumeric, min_length: 2, max_length: 50),
             email <- email_generator(),
             age  <- integer(18..120) do
      %{name: name, email: email, age: age}
    end
  end
  
  defp email_generator do
    gen all local  <- string(:alphanumeric, min_length: 1, max_length: 20),
             domain <- string(:alphanumeric, min_length: 2, max_length: 10) do
      "#{local}@#{domain}.com"
    end
  end
end
```

---

## Step 445: Integration Testing

```elixir
defmodule MyAppWeb.OrderFlowTest do
  use MyAppWeb.ConnCase
  import MyApp.Factory
  
  setup do
    user = insert(:user)
    {:ok, token, _claims} = MyAppWeb.Auth.encode_and_sign(user)
    
    %{user: user, token: token}
  end
  
  test "complete order flow", %{user: user, token: token} do
    # 1. Browse products
    conn = get(build_conn(), ~p"/api/v1/products")
    products = json_response(conn, 200)["data"]
    product = List.first(products)
    
    # 2. Add to cart
    conn = build_conn()
      |> put_req_header("authorization", "Bearer #{token}")
      |> post(~p"/api/v1/cart/items", %{
          product_id: product["id"],
          quantity: 2
        })
    
    assert json_response(conn, 201)
    
    # 3. Checkout
    conn = build_conn()
      |> put_req_header("authorization", "Bearer #{token}")
      |> post(~p"/api/v1/orders", %{
          payment_method: "card",
          card_token: "tok_visa"
        })
    
    %{"id" => order_id} = json_response(conn, 201)
    
    # 4. Verify order created
    conn = build_conn()
      |> put_req_header("authorization", "Bearer #{token}")
      |> get(~p"/api/v1/orders/#{order_id}")
    
    order = json_response(conn, 200)["data"]
    
    assert order["status"] == "pending"
    assert order["user_id"] == user.id
    
    # 5. Verify side effects
    assert_enqueued(worker: MyApp.Workers.OrderConfirmationEmail)
    assert_enqueued(worker: MyApp.Workers.InventoryReserve)
  end
end
```

---

## Step 446: Testing LiveView

```elixir
defmodule MyAppWeb.ProductLiveTest do
  use MyAppWeb.ConnCase, async: true
  import Phoenix.LiveViewTest
  import MyApp.Factory
  
  test "renders product list", %{conn: conn} do
    insert_list(3, :product)
    
    {:ok, view, html} = live(conn, ~p"/products")
    
    assert html =~ "Products"
    assert has_element?(view, ".product-card", "Product")
  end
  
  test "filters products by category", %{conn: conn} do
    cat1 = insert(:category, name: "Electronics")
    cat2 = insert(:category, name: "Books")
    
    insert(:product, category: cat1, name: "Laptop")
    insert(:product, category: cat2, name: "Novel")
    
    {:ok, view, _html} = live(conn, ~p"/products")
    
    # Apply filter
    view
    |> element("#category-filter")
    |> render_change(%{category: cat1.id})
    
    assert has_element?(view, ".product-card", "Laptop")
    refute has_element?(view, ".product-card", "Novel")
  end
  
  test "adds product to cart via form submit", %{conn: conn} do
    user    = insert(:user)
    product = insert(:product)
    
    {:ok, view, _html} = conn
      |> log_in_user(user)
      |> live(~p"/products/#{product.id}")
    
    view
    |> form("#add-to-cart-form", %{quantity: 2})
    |> render_submit()
    
    assert_patched(view, ~p"/cart")
    flash = get_flash(view)
    assert flash["info"] =~ "Added to cart"
  end
  
  test "receives real-time price update via PubSub", %{conn: conn} do
    product = insert(:product, price: Decimal.new("100"))
    
    {:ok, view, _html} = live(conn, ~p"/products/#{product.id}")
    
    assert has_element?(view, ".price", "$100.00")
    
    # Simulate price update from another process
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "product:#{product.id}",
      {:price_updated, Decimal.new("90.00")}
    )
    
    assert has_element?(view, ".price", "$90.00")
  end
end
```

---

## Step 447: Testing Channels

```elixir
defmodule MyAppWeb.ChatChannelTest do
  use MyAppWeb.ChannelCase
  import MyApp.Factory
  
  setup do
    user = insert(:user)
    room = insert(:room)
    
    {:ok, socket} = connect(MyAppWeb.UserSocket, %{"token" => generate_token(user)})
    {:ok, _reply, socket} = subscribe_and_join(socket, "room:#{room.id}", %{})
    
    %{socket: socket, user: user, room: room}
  end
  
  test "sends message to channel", %{socket: socket, user: user} do
    push(socket, "new_message", %{body: "Hello!"})
    
    assert_broadcast "new_message", %{
      body:    "Hello!",
      user_id: ^(user.id),
      user:    %{name: _}
    }
  end
  
  test "receives typing indicator", %{socket: socket} do
    push(socket, "typing", %{})
    assert_broadcast "user_typing", %{user_id: _}
  end
  
  test "disconnects unauthorized user" do
    {:error, reason} = connect(MyAppWeb.UserSocket, %{"token" => "invalid"})
    assert reason == :unauthorized
  end
end
```

---

## Step 448: Testing GenServer

```elixir
defmodule MyApp.CacheTest do
  use ExUnit.Case, async: true
  
  alias MyApp.Cache
  
  setup do
    # Start a fresh cache for each test
    {:ok, cache} = Cache.start_link(name: nil, ttl: 100)
    {:ok, cache: cache}
  end
  
  test "stores and retrieves values", %{cache: cache} do
    Cache.put(cache, "key", "value")
    assert Cache.get(cache, "key") == "value"
  end
  
  test "returns nil for missing key", %{cache: cache} do
    assert Cache.get(cache, "missing") == nil
  end
  
  test "expires entries after TTL", %{cache: cache} do
    Cache.put(cache, "key", "value")
    
    # Wait past TTL (100ms in setup)
    Process.sleep(150)
    
    assert Cache.get(cache, "key") == nil
  end
  
  test "evicts old entries when full", %{cache: cache} do
    # Fill to capacity
    for i <- 1..100 do
      Cache.put(cache, "key#{i}", "value#{i}")
    end
    
    # Adding more should not crash
    Cache.put(cache, "new_key", "new_value")
    
    # New key accessible
    assert Cache.get(cache, "new_key") == "new_value"
  end
  
  test "handles concurrent access", %{cache: cache} do
    tasks = for i <- 1..100 do
      Task.async(fn ->
        Cache.put(cache, "key#{i}", i)
        Cache.get(cache, "key#{i}")
      end)
    end
    
    results = Task.await_many(tasks)
    
    # All values should be either nil (race) or correct integer
    Enum.each(Enum.with_index(results), fn {result, i} ->
      assert result == nil || result == i + 1
    end)
  end
end
```

---

## Step 449: Coverage และ Quality

```bash
# mix.exs
defp aliases do
  [
    test: ["ecto.create --quiet", "ecto.migrate --quiet", "test"],
    "test.coverage": ["test --cover"],
    "ci": ["format --check-formatted", "credo --strict", "dialyzer", "test --cover"]
  ]
end

# .credo.exs
%{
  configs: [
    %{
      name: "default",
      checks: %{
        enabled: [
          {Credo.Check.Consistency.TabsOrSpaces},
          {Credo.Check.Design.AliasUsage},
          {Credo.Check.Readability.FunctionNames},
          {Credo.Check.Refactor.FunctionArity, max_arity: 5},
          {Credo.Check.Refactor.LongQuoteBlocks},
          {Credo.Check.Warning.UnusedEnumOperation},
          {Credo.Check.Warning.LazyLogging}
        ]
      }
    }
  ]
}

# .dialyzer_ignore.exs - suppress known false positives
[
  {:warn_contract_supertype, :_, :_}
]
```

---

## Step 450: CI Testing Pipeline

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: my_app_test
        ports: ["5432:5432"]
      
      redis:
        image: redis:7
        ports: ["6379:6379"]
    
    env:
      MIX_ENV: test
      DATABASE_URL: postgresql://postgres:postgres@localhost/my_app_test
      REDIS_URL: redis://localhost:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Elixir
        uses: erlef/setup-beam@v1
        with:
          otp-version: "26.2"
          elixir-version: "1.16"
      
      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: deps
          key: ${{ runner.os }}-mix-${{ hashFiles('**/mix.lock') }}
      
      - run: mix deps.get
      - run: mix compile --warnings-as-errors
      - run: mix format --check-formatted
      - run: mix credo --strict
      - run: mix ecto.create && mix ecto.migrate
      - run: mix test --cover
      - run: mix dialyzer
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

---

## สรุป Part 40

✅ **Step 441** - ExUnit advanced  
✅ **Step 442** - ExMachina factories  
✅ **Step 443** - Mox mocking  
✅ **Step 444** - StreamData property tests  
✅ **Step 445** - Integration testing  
✅ **Step 446** - LiveView testing  
✅ **Step 447** - Channel testing  
✅ **Step 448** - GenServer testing  
✅ **Step 449** - Coverage + quality tools  
✅ **Step 450** - CI testing pipeline  

➡️ [Part 41: Open Source Library Development](./part-41-library-dev.md)
