# Part 55: Advanced Testing Strategies (Steps 601-620)

## Step 601: Contract Testing

```elixir
# Contract testing: verify API contracts between services
# Using Pact or custom contract tests

defmodule MyApp.ContractTest do
  use ExUnit.Case

  # Define the contract
  @user_contract %{
    required_fields: [:id, :name, :email],
    types: %{
      id:    :integer,
      name:  :string,
      email: :string,
      role:  {:enum, [:user, :admin]}
    }
  }

  def validate_contract(data, contract) do
    # Check required fields
    missing = contract.required_fields
      |> Enum.filter(fn f -> not Map.has_key?(data, f) end)
    
    # Check types
    type_errors = contract.types
      |> Enum.filter(fn {field, type} ->
        Map.has_key?(data, field) and not valid_type?(data[field], type)
      end)

    if missing == [] and type_errors == [] do
      :ok
    else
      {:error, %{missing: missing, type_errors: type_errors}}
    end
  end

  defp valid_type?(value, :integer), do: is_integer(value)
  defp valid_type?(value, :string),  do: is_binary(value)
  defp valid_type?(value, {:enum, vals}), do: value in vals

  test "UserService response matches contract" do
    response = MyApp.UserService.get_user(1)
    assert :ok == validate_contract(response, @user_contract)
  end
end
```

---

## Step 602: Mutation Testing

```elixir
# Mutation testing with ExUnit - verify tests catch bugs
# Install: mix archive.install hex mix_test_watch

defmodule MyApp.Calculator do
  def add(a, b), do: a + b
  def multiply(a, b), do: a * b
  def divide(_a, 0), do: {:error, :division_by_zero}
  def divide(a, b), do: {:ok, a / b}
end

defmodule MyApp.CalculatorTest do
  use ExUnit.Case

  # Good tests catch mutations like a + b → a - b, a * b → a / b
  test "add returns correct sum" do
    assert MyApp.Calculator.add(2, 3) == 5
    assert MyApp.Calculator.add(-1, 1) == 0
    assert MyApp.Calculator.add(0, 0) == 0
  end

  test "multiply returns correct product" do
    assert MyApp.Calculator.multiply(3, 4) == 12
    assert MyApp.Calculator.multiply(0, 5) == 0
    assert MyApp.Calculator.multiply(-2, 3) == -6
  end

  test "divide by zero returns error" do
    assert {:error, :division_by_zero} = MyApp.Calculator.divide(5, 0)
  end

  test "divide returns result" do
    assert {:ok, 2.5} = MyApp.Calculator.divide(5, 2)
  end
end

# Run mutation analysis manually by temporarily changing operators and checking tests fail
# mix test --cover to see coverage
```

---

## Step 603: Load Testing with Locust/k6

```javascript
// k6/load_test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m',  target: 100 },  // ramp up
    { duration: '5m',  target: 100 },  // hold
    { duration: '2m',  target: 200 },  // spike
    { duration: '2m',  target: 100 },  // back down
    { duration: '2m',  target: 0   },  // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],   // 99% under 500ms
    http_req_failed:   ['rate<0.01'],   // less than 1% errors
    errors:            ['rate<0.05']
  }
};

export default function () {
  const BASE_URL = 'https://staging.myapp.com';
  
  // Test products listing
  const products = http.get(`${BASE_URL}/api/products`);
  check(products, {
    'products status 200': (r) => r.status === 200,
    'products latency < 200ms': (r) => r.timings.duration < 200
  });
  
  // Test search
  const search = http.get(`${BASE_URL}/api/products?q=elixir`);
  errorRate.add(search.status !== 200);
  
  sleep(1);
}
```

```elixir
# Elixir side: simulate load with Task
defmodule MyApp.LoadSimulator do
  def run(concurrency, duration_ms, fun) do
    start = System.monotonic_time(:millisecond)
    
    tasks = for _ <- 1..concurrency do
      Task.async(fn ->
        loop_until(start + duration_ms, fun, [])
      end)
    end
    
    results = Task.await_many(tasks, duration_ms + 5_000)
    
    all_results = List.flatten(results)
    successes = Enum.count(all_results, & &1 == :ok)
    
    %{
      total:         length(all_results),
      success_rate:  successes / max(length(all_results), 1) * 100,
      concurrency:   concurrency
    }
  end

  defp loop_until(deadline, fun, acc) do
    if System.monotonic_time(:millisecond) < deadline do
      result = fun.()
      loop_until(deadline, fun, [result | acc])
    else
      acc
    end
  end
end
```

---

## Step 604: Chaos Testing with ExUnit

```elixir
defmodule MyApp.ChaosTest do
  use ExUnit.Case, async: false

  @tag :chaos
  test "system recovers from database connection loss" do
    # Simulate connection pool exhaustion
    tasks = for _ <- 1..100 do
      Task.async(fn ->
        MyApp.Repo.query("SELECT pg_sleep(2)")
      end)
    end
    
    # During pool exhaustion, requests should get queued error, not crash
    result = MyApp.Products.list()
    assert {:error, _} = result or match?([_ | _], result)
    
    Task.await_many(tasks, 5_000)
    
    # System should recover
    assert [_ | _] = MyApp.Products.list()
  end

  @tag :chaos
  test "system handles GenServer crash and restart" do
    pid_before = GenServer.whereis(MyApp.Cache)
    
    Process.exit(pid_before, :kill)
    Process.sleep(100)  # Allow supervisor to restart
    
    pid_after = GenServer.whereis(MyApp.Cache)
    assert pid_after != nil
    assert pid_after != pid_before
    
    # Cache should work after restart
    assert :ok = MyApp.Cache.put("key", "value")
    assert "value" = MyApp.Cache.get("key")
  end

  @tag :chaos
  test "circuit breaker opens after repeated failures" do
    # Simulate repeated failures
    for _ <- 1..5 do
      assert {:error, _} = MyApp.ExternalService.call(%{fail: true})
    end
    
    # Circuit should be open now
    assert {:error, :circuit_open} = MyApp.ExternalService.call(%{})
    
    # Wait for circuit to half-open
    Process.sleep(1_000)
    
    # Should try again (half-open state)
    result = MyApp.ExternalService.call(%{})
    assert result != {:error, :circuit_open}
  end
end
```

---

## Step 605: Visual Regression Testing

```elixir
# Visual screenshot testing for LiveView
defmodule MyAppWeb.VisualTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest

  @screenshot_dir "test/screenshots"

  setup do
    File.mkdir_p!(@screenshot_dir)
    :ok
  end

  test "product page renders correctly", %{conn: conn} do
    insert(:product, name: "Test Product", price: Decimal.new("29.99"))
    
    {:ok, view, html} = live(conn, "/products")
    
    # Save baseline screenshot (HTML snapshot for comparison)
    baseline_file = "#{@screenshot_dir}/products_baseline.html"
    
    if File.exists?(baseline_file) do
      baseline = File.read!(baseline_file)
      # Compare normalized HTML
      assert normalize_html(html) == normalize_html(baseline),
        "Visual regression detected in products page"
    else
      File.write!(baseline_file, html)
      IO.puts("Saved baseline screenshot: #{baseline_file}")
    end
  end

  defp normalize_html(html) do
    # Remove dynamic parts (timestamps, ids, etc.)
    html
    |> String.replace(~r/id="[^"]*phx-[^"]*"/, "id=\"phx-normalized\"")
    |> String.replace(~r/\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/, "TIMESTAMP")
  end
end
```

---

## Step 606: Test Data Factories

```elixir
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.Accounts.User{
      name:            sequence(:name, &"User #{&1}"),
      email:           sequence(:email, &"user#{&1}@example.com"),
      hashed_password: Bcrypt.hash_pwd_salt("password123"),
      role:            :user,
      confirmed_at:    ~N[2024-01-01 00:00:00]
    }
  end

  def admin_user_factory do
    struct!(user_factory(), %{role: :admin})
  end

  def product_factory do
    %MyApp.Catalog.Product{
      name:        sequence(:product_name, &"Product #{&1}"),
      description: "A great product description",
      price:       Decimal.new("#{:rand.uniform(10_000) / 100}"),
      stock:       :rand.uniform(100),
      category:    Enum.random(["books", "electronics", "clothing"]),
      user:        build(:user)
    }
  end

  def order_factory do
    %MyApp.Orders.Order{
      user:   build(:user),
      status: :pending,
      total:  Decimal.new("99.99"),
      items:  build_list(2, :order_item)
    }
  end

  def order_item_factory do
    %MyApp.Orders.OrderItem{
      product:  build(:product),
      quantity: :rand.uniform(5),
      price:    Decimal.new("29.99")
    }
  end

  # Traits
  def with_admin_user(factory) do
    Map.put(factory, :user, build(:admin_user))
  end

  def out_of_stock(factory) do
    Map.put(factory, :stock, 0)
  end
end
```

---

## Step 607: Integration Test Helpers

```elixir
defmodule MyAppWeb.ConnCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      use Phoenix.ConnTest
      import Plug.Conn
      import Phoenix.LiveViewTest
      import MyApp.Factory
      alias MyApp.Repo
      
      @endpoint MyAppWeb.Endpoint

      def register_and_log_in_user(%{conn: conn}) do
        user = insert(:user)
        conn = log_in_user(conn, user)
        %{conn: conn, user: user}
      end

      def log_in_user(conn, user) do
        token = MyApp.Accounts.generate_user_session_token(user)
        conn
        |> Phoenix.ConnTest.init_test_session(%{})
        |> Plug.Conn.put_session(:user_token, token)
      end

      def api_auth_headers(user) do
        {:ok, token, _} = MyApp.Auth.Guardian.encode_and_sign(user)
        [{"authorization", "Bearer #{token}"}]
      end

      def assert_email_sent(to: to, subject: subject) do
        assert_receive {:email, %Swoosh.Email{to: [{_, ^to}], subject: ^subject}}
      end
    end
  end

  setup tags do
    pid = Ecto.Adapters.SQL.Sandbox.start_owner!(MyApp.Repo, shared: not tags[:async])
    on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
    {:ok, conn: Phoenix.ConnTest.build_conn()}
  end
end
```

---

## Step 608: Testing Oban Jobs

```elixir
defmodule MyApp.Workers.WelcomeEmailWorkerTest do
  use MyApp.DataCase, async: true
  use Oban.Testing, repo: MyApp.Repo

  test "sends welcome email" do
    user = insert(:user)
    
    assert :ok = perform_job(MyApp.Workers.WelcomeEmailWorker, %{user_id: user.id})
    
    assert_email_sent(to: user.email, subject: "Welcome to MyApp!")
  end

  test "handles missing user gracefully" do
    result = perform_job(MyApp.Workers.WelcomeEmailWorker, %{user_id: 99999})
    assert {:discard, :user_not_found} = result
  end

  test "retries on email service failure" do
    user = insert(:user)
    
    # Mock email service to fail
    Mox.expect(MyApp.EmailMock, :send, fn _ -> {:error, :service_unavailable} end)
    
    result = perform_job(MyApp.Workers.WelcomeEmailWorker, %{user_id: user.id})
    assert {:error, _} = result
  end

  test "job is enqueued on user creation" do
    {:ok, user} = MyApp.Accounts.create_user(%{name: "New", email: "new@test.com", password: "pass123"})
    
    assert_enqueued(worker: MyApp.Workers.WelcomeEmailWorker, args: %{user_id: user.id})
  end
end
```

---

## Step 609: Security Testing

```elixir
defmodule MyAppWeb.SecurityTest do
  use MyAppWeb.ConnCase, async: true

  describe "SQL injection prevention" do
    test "parameterized queries prevent injection" do
      malicious_input = "'; DROP TABLE users; --"
      
      # This should safely return empty results, not execute SQL
      result = MyApp.Accounts.search_users(malicious_input)
      assert result == []
      
      # Users table should still exist
      assert MyApp.Repo.aggregate(MyApp.Accounts.User, :count) >= 0
    end
  end

  describe "XSS prevention" do
    test "HTML is escaped in responses" do
      xss_payload = "<script>alert('xss')</script>"
      user = insert(:user, name: xss_payload)
      
      conn = get(build_conn(), "/users/#{user.id}")
      
      assert response(conn, 200) =~ "&lt;script&gt;"
      refute response(conn, 200) =~ "<script>"
    end
  end

  describe "CSRF protection" do
    test "POST without CSRF token is rejected" do
      conn = build_conn()
        |> post("/users", %{user: %{name: "test"}})
      
      assert response(conn, 403)
    end
  end

  describe "rate limiting" do
    test "login is rate limited after 5 attempts" do
      for _ <- 1..5 do
        post(build_conn(), "/users/log_in", %{user: %{email: "x@test.com", password: "wrong"}})
      end
      
      conn = post(build_conn(), "/users/log_in", %{user: %{email: "x@test.com", password: "wrong"}})
      assert response(conn, 429)
    end
  end
end
```

---

## Step 610: Test Configuration

```elixir
# config/test.exs
config :my_app, MyApp.Repo,
  pool: Ecto.Adapters.SQL.Sandbox,
  pool_size: System.schedulers_online() * 2

config :my_app, :email_adapter, MyApp.EmailMock  # Use mock in tests
config :bcrypt_elixir, :log_rounds, 1             # Fast hashing in tests
config :my_app, :rate_limiter, disabled: true     # Disable rate limiting

# test/test_helper.exs
ExUnit.configure(
  exclude: [:pending, :chaos, :integration],  # Skip slow tests by default
  formatters: [ExUnit.CLIFormatter],
  timeout: 30_000
)

Mox.defmock(MyApp.EmailMock, for: MyApp.EmailBehaviour)
Mox.defmock(MyApp.PaymentMock, for: MyApp.PaymentBehaviour)

ExUnit.start()
Ecto.Adapters.SQL.Sandbox.mode(MyApp.Repo, :manual)

# Run specific test suites:
# mix test                           # Unit tests only
# mix test --include integration     # Include integration tests
# mix test --include chaos           # Include chaos tests
# mix test --only pending            # Only pending tests
```

---

## สรุป Part 55

✅ **Step 601** - Contract testing  
✅ **Step 602** - Mutation testing  
✅ **Step 603** - Load testing  
✅ **Step 604** - Chaos testing  
✅ **Step 605** - Visual regression  
✅ **Step 606** - Test factories  
✅ **Step 607** - Integration helpers  
✅ **Step 608** - Oban job testing  
✅ **Step 609** - Security testing  
✅ **Step 610** - Test configuration  

➡️ [Part 56: SaaS Architecture](./part-56-saas.md)
