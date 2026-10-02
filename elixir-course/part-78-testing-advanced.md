# Part 78: Advanced Testing Strategies (Steps 841-860)

## Step 841: Property-Based Testing with StreamData

```elixir
# mix.exs: {:stream_data, "~> 1.1"}

defmodule MyApp.PropertyTest do
  use ExUnit.Case
  use ExUnitProperties

  # Property: encode/decode should be inverse operations
  property "JSON encode/decode is idempotent for maps with string keys" do
    check all map <- map_of(string(:alphanumeric), integer()) do
      assert Jason.decode!(Jason.encode!(map)) == map
    end
  end

  # Property: sorting should be idempotent
  property "list sorting is idempotent" do
    check all list <- list_of(integer()) do
      sorted = Enum.sort(list)
      assert Enum.sort(sorted) == sorted
    end
  end

  # Property: string length after concatenation
  property "concatenated string length equals sum of parts" do
    check all s1 <- string(:alphanumeric),
              s2 <- string(:alphanumeric) do
      assert String.length(s1 <> s2) == String.length(s1) + String.length(s2)
    end
  end

  # Custom generator
  property "user changeset is valid for generated valid data" do
    check all attrs <- valid_user_attrs() do
      changeset = MyApp.Accounts.User.changeset(%MyApp.Accounts.User{}, attrs)
      assert changeset.valid?
    end
  end

  defp valid_user_attrs do
    gen all name <- string(:alphanumeric, min_length: 2),
             email <- email_gen(),
             password <- string(:alphanumeric, min_length: 8) do
      %{name: name, email: email, password: password}
    end
  end

  defp email_gen do
    gen all local  <- string(:alphanumeric, min_length: 1),
             domain <- string(:alphanumeric, min_length: 1) do
      "#{local}@#{domain}.com"
    end
  end
end
```

---

## Step 842: Contract Testing

```elixir
defmodule MyApp.ContractTest do
  use ExUnit.Case

  # Contract: UserService.get_user/1 must always return {:ok, user} or {:error, :not_found}
  describe "UserService contract" do
    test "get_user returns expected tuple format" do
      existing_user = insert(:user)
      
      result = MyApp.UserService.get_user(existing_user.id)
      assert match?({:ok, %{id: _, name: _, email: _}}, result)
      
      {:ok, user} = result
      assert is_integer(user.id) or is_binary(user.id)
      assert is_binary(user.name)
      assert is_binary(user.email)
    end

    test "get_user returns not_found for unknown id" do
      assert {:error, :not_found} = MyApp.UserService.get_user(99_999_999)
    end

    test "create_user returns user with required fields" do
      attrs = %{name: "Test", email: "test@test.com", password: "password123"}
      
      {:ok, user} = MyApp.UserService.create_user(attrs)
      
      # Contract: created user must have these fields
      assert Map.has_key?(user, :id)
      assert Map.has_key?(user, :name)
      assert Map.has_key?(user, :email)
      assert Map.has_key?(user, :inserted_at)
      refute Map.has_key?(user, :password_hash)  # never expose hash
    end
  end
end
```

---

## Step 843: Mutation Testing

```elixir
# Verify your tests actually catch bugs

# Original function
defmodule MyApp.Calculator do
  def divide(a, _b = 0), do: {:error, :division_by_zero}
  def divide(a, b), do: {:ok, a / b}
  
  def factorial(0), do: 1
  def factorial(n) when n > 0, do: n * factorial(n - 1)
end

# Mutation tests: deliberately introduce bugs and verify tests catch them
defmodule MyApp.CalculatorMutationTest do
  use ExUnit.Case

  # Test that would catch "a + b instead of a / b"
  test "divide computes actual division not addition" do
    assert {:ok, 2.0} = MyApp.Calculator.divide(10, 5)
    refute {:ok, 15.0} == MyApp.Calculator.divide(10, 5)  # would fail if + mutation
  end

  # Test that would catch "factorial(n-1) -> factorial(n)"
  test "factorial terminates" do
    assert 120 = MyApp.Calculator.factorial(5)
    assert 1   = MyApp.Calculator.factorial(0)
    assert 1   = MyApp.Calculator.factorial(1)
  end

  # Test coverage matrix
  test "division by zero is detected for any numerator" do
    for n <- [-5, -1, 0, 1, 5, 100] do
      assert {:error, :division_by_zero} = MyApp.Calculator.divide(n, 0)
    end
  end
end
```

---

## Step 844: Snapshot Testing

```elixir
defmodule MyApp.SnapshotTest do
  use MyAppWeb.ConnCase

  # Snapshot test: verify JSON response doesn't change unexpectedly
  test "GET /api/v1/users returns consistent structure" do
    user = insert(:user, name: "Test User", email: "test@test.com")
    conn = get(build_auth_conn(user), "/api/v1/users/#{user.id}")
    
    response = json_response(conn, 200)
    
    # Compare against snapshot
    snapshot_path = "test/snapshots/api_user_response.json"
    
    if File.exists?(snapshot_path) do
      expected = File.read!(snapshot_path) |> Jason.decode!()
      assert response == expected
    else
      # Create snapshot on first run
      File.write!(snapshot_path, Jason.encode!(response, pretty: true))
    end
  end
end
```

---

## Step 845: Load Testing

```elixir
defmodule MyApp.LoadTest do
  # Mix task for load testing with :gun HTTP client

  def run(url, concurrent \\ 10, total \\ 1000) do
    start_time = System.monotonic_time(:millisecond)
    
    results = 1..total
      |> Enum.chunk_every(div(total, concurrent))
      |> Enum.map(fn batch ->
        Task.async(fn ->
          Enum.map(batch, fn _ ->
            request(url)
          end)
        end)
      end)
      |> Task.await_many(60_000)
      |> List.flatten()

    elapsed = System.monotonic_time(:millisecond) - start_time
    
    successes = Enum.count(results, & &1.status == 200)
    failures  = total - successes
    avg_ms    = Enum.sum(Enum.map(results, & &1.duration)) / total

    %{
      total:      total,
      successes:  successes,
      failures:   failures,
      rps:        total / (elapsed / 1000),
      avg_ms:     avg_ms,
      p95_ms:     percentile(Enum.map(results, & &1.duration), 95),
      p99_ms:     percentile(Enum.map(results, & &1.duration), 99)
    }
  end

  defp request(url) do
    start = System.monotonic_time(:millisecond)
    
    case Req.get(url) do
      {:ok, %{status: status}} ->
        %{status: status, duration: System.monotonic_time(:millisecond) - start}
      {:error, _} ->
        %{status: 0, duration: System.monotonic_time(:millisecond) - start}
    end
  end

  defp percentile(data, p) do
    sorted = Enum.sort(data)
    index  = ceil(length(sorted) * p / 100) - 1
    Enum.at(sorted, max(0, index))
  end
end
```

---

## Step 846: Integration Testing with Docker

```elixir
# test/support/docker_test_helpers.ex
defmodule MyApp.DockerHelpers do
  # Start real PostgreSQL for integration tests
  
  def start_postgres do
    container_id = "my_app_test_#{System.unique_integer([:positive])}"
    
    {_, 0} = System.cmd("docker", [
      "run", "-d",
      "--name", container_id,
      "-e", "POSTGRES_PASSWORD=postgres",
      "-p", "5433:5432",
      "postgres:16-alpine"
    ])

    wait_for_postgres()

    on_exit(fn ->
      System.cmd("docker", ["rm", "-f", container_id])
    end)

    %{container_id: container_id, port: 5433}
  end

  defp wait_for_postgres(attempts \\ 30) do
    case Postgrex.start_link(
      hostname: "localhost",
      port:     5433,
      username: "postgres",
      password: "postgres",
      database: "postgres"
    ) do
      {:ok, conn} ->
        GenServer.stop(conn)
        :ok
      {:error, _} when attempts > 0 ->
        Process.sleep(1000)
        wait_for_postgres(attempts - 1)
      {:error, reason} ->
        raise "PostgreSQL did not start: #{inspect(reason)}"
    end
  end
end
```

---

## Step 847: Chaos Testing

```elixir
defmodule MyApp.ChaosTest do
  use ExUnit.Case

  # Test system behavior under failures
  
  test "system handles database connection failure gracefully" do
    # Temporarily break DB connection
    :ok = disconnect_database()
    
    try do
      result = MyApp.UserService.get_user(1)
      # Should fail gracefully, not crash
      assert match?({:error, _}, result)
    after
      :ok = reconnect_database()
    end
  end

  test "circuit breaker opens after repeated failures" do
    # Simulate external service failure
    Mox.stub(MyApp.PaymentMock, :charge, fn _ ->
      {:error, :service_unavailable}
    end)

    # Should open circuit breaker after threshold
    for _ <- 1..5 do
      MyApp.PaymentService.charge(100, "fake_card")
    end

    # Circuit should be open now
    assert {:error, :circuit_open} = MyApp.PaymentService.charge(100, "fake_card")
  end

  test "graceful degradation when cache unavailable" do
    # Kill Redis connection
    Mox.stub(MyApp.CacheMock, :get, fn _ -> {:error, :connection_refused} end)
    
    # Should fall back to database
    result = MyApp.Products.list()
    
    # Still returns results (from DB fallback)
    assert is_list(result)
    assert length(result) > 0
  end
end
```

---

## Step 848: Test Coverage Analysis

```elixir
# mix.exs - add coveralls
{:excoveralls, "~> 0.18", only: :test}

# .coveralls.json
{
  "coverage_options": {
    "minimum_coverage": 80
  },
  "skip_files": [
    "lib/my_app_web/telemetry.ex",
    "lib/my_app/release.ex"
  ]
}

# mix.exs aliases
aliases: [
  test: "test --cover",
  "test.ci": "coveralls.github"
]

# Run: mix test --cover
# Shows: Module coverage, line-by-line missing

# ExUnit configuration
ExUnit.configure(
  assert_receive_timeout: 500,
  refute_receive_timeout: 100,
  formatters: [ExUnit.CLIFormatter],
  exclude: [:integration, :external]
)
```

---

## Step 849: Fuzz Testing

```elixir
defmodule MyApp.FuzzTest do
  use ExUnit.Case

  # Fuzz test: feed random inputs, ensure no crashes
  
  test "JSON decoder handles any binary input without crashing" do
    for _ <- 1..1000 do
      random_binary = :crypto.strong_rand_bytes(:rand.uniform(100))
      
      # Should never raise, only return {:ok, _} or {:error, _}
      result = Jason.decode(random_binary)
      assert match?({:ok, _}, result) or match?({:error, _}, result)
    end
  end

  test "changeset handles arbitrary map inputs" do
    for _ <- 1..500 do
      random_attrs = %{
        name:     random_string(),
        email:    random_string(),
        password: random_string(),
        age:      :rand.uniform(200) - 100,  # including negatives
        extra:    "ignored_field"
      }

      changeset = MyApp.Accounts.User.changeset(%MyApp.Accounts.User{}, random_attrs)
      
      # Changeset should never crash
      assert %Ecto.Changeset{} = changeset
    end
  end

  defp random_string do
    length = :rand.uniform(50)
    :crypto.strong_rand_bytes(length) |> Base.encode64()
  end
end
```

---

## Step 850: Test Factory Patterns

```elixir
# test/support/factory.ex - using ex_machina
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.Accounts.User{
      name:          sequence(:name, &"User #{&1}"),
      email:         sequence(:email, &"user#{&1}@example.com"),
      password_hash: Bcrypt.hash_pwd_salt("password123"),
      role:          :user,
      confirmed_at:  DateTime.utc_now()
    }
  end

  def admin_factory do
    struct!(user_factory(), role: :admin)
  end

  def product_factory do
    %MyApp.Catalog.Product{
      name:        sequence(:name, &"Product #{&1}"),
      price:       Decimal.new("#{:rand.uniform(10000) / 100}"),
      stock:       :rand.uniform(100),
      description: "A test product"
    }
  end

  def order_factory do
    %MyApp.Orders.Order{
      user:   build(:user),
      status: :pending,
      total:  Decimal.new("99.99"),
      items:  [build(:order_item)]
    }
  end

  def order_item_factory do
    %MyApp.Orders.OrderItem{
      product:  build(:product),
      quantity: :rand.uniform(5),
      price:    Decimal.new("19.99")
    }
  end

  # Trait: make an order that's been paid
  def paid_order_factory do
    struct!(order_factory(),
      status:     :paid,
      paid_at:    DateTime.utc_now(),
      payment_id: "pi_#{random_string(10)}"
    )
  end

  defp random_string(len) do
    :crypto.strong_rand_bytes(len)
    |> Base.encode16(case: :lower)
    |> String.slice(0, len)
  end
end

# Usage
user     = insert(:user)
admin    = insert(:admin)
order    = insert(:order, user: user)
paid     = insert(:paid_order)
products = insert_list(5, :product)
```

---

## สรุป Part 78

✅ **Step 841** - Property-based testing  
✅ **Step 842** - Contract testing  
✅ **Step 843** - Mutation testing  
✅ **Step 844** - Snapshot testing  
✅ **Step 845** - Load testing  
✅ **Step 846** - Integration with Docker  
✅ **Step 847** - Chaos testing  
✅ **Step 848** - Coverage analysis  
✅ **Step 849** - Fuzz testing  
✅ **Step 850** - Factory patterns  

➡️ [Part 79: Advanced OTP Patterns](./part-79-otp-advanced.md)
