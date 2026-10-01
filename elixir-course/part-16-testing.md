# Part 16: Testing ด้วย ExUnit (Steps 171-190)

## Step 171: ExUnit Basics

```elixir
# test/my_module_test.exs
defmodule MyModuleTest do
  use ExUnit.Case
  
  # Test ง่ายๆ
  test "addition works" do
    assert 1 + 1 == 2
  end
  
  test "string concatenation" do
    assert "hello " <> "world" == "hello world"
  end
  
  # doctest - test จาก @doc
  doctest MyModule
end

# test/test_helper.exs
ExUnit.start()
```

---

## Step 172: Assertions

```elixir
defmodule AssertionsTest do
  use ExUnit.Case
  
  test "assert" do
    assert true
    assert 1 == 1
    assert "hello" =~ ~r/hell/
  end
  
  test "refute" do
    refute false
    refute 1 == 2
  end
  
  test "assert_raise" do
    assert_raise ArithmeticError, fn ->
      1 / 0
    end
    
    assert_raise ArgumentError, "invalid argument", fn ->
      raise ArgumentError, "invalid argument"
    end
  end
  
  test "assert_receive" do
    send(self(), :hello)
    assert_receive :hello
    assert_receive :hello, 1000  # timeout
  end
  
  test "refute_receive" do
    refute_receive :hello, 100  # should not receive within 100ms
  end
  
  test "assert_in_delta" do
    assert_in_delta 3.14, :math.pi(), 0.01
  end
  
  test "pattern matching in assert" do
    result = {:ok, %{name: "Alice", age: 30}}
    assert {:ok, %{name: name}} = result
    assert name == "Alice"
  end
end
```

---

## Step 173: Setup และ Callbacks

```elixir
defmodule SetupTest do
  use ExUnit.Case
  
  # setup ทำงานก่อน ทุก test
  setup do
    db = start_test_db()
    on_exit(fn -> cleanup_db(db) end)
    {:ok, db: db}  # context
  end
  
  # setup_all ทำงานครั้งเดียวก่อน ทั้ง describe block
  setup_all do
    {:ok, server} = MyServer.start_link()
    {:ok, server: server}
  end
  
  test "reads from db", %{db: db} do
    assert {:ok, _} = DB.query(db, "SELECT 1")
  end
  
  # setup สำหรับ specific test ด้วย function
  setup [:create_user, :create_post]
  
  defp create_user(context) do
    {:ok, user} = Users.create(%{name: "Test User"})
    Map.put(context, :user, user)
  end
  
  defp create_post(%{user: user} = context) do
    {:ok, post} = Posts.create(%{user_id: user.id, title: "Test Post"})
    Map.put(context, :post, post)
  end
  
  test "user has post", %{user: user, post: post} do
    assert post.user_id == user.id
  end
end
```

---

## Step 174: Tags และ Filters

```elixir
defmodule TaggedTest do
  use ExUnit.Case
  
  @moduletag :slow
  @moduletag :integration
  
  @tag :unit
  test "fast test" do
    assert 1 + 1 == 2
  end
  
  @tag :slow
  @tag timeout: 30_000
  test "slow test" do
    :timer.sleep(5000)
    assert true
  end
  
  @tag :skip
  test "not implemented yet" do
    # skipped
  end
  
  @tag :capture_log
  test "test with logs" do
    require Logger
    Logger.info("this is captured")
    assert true
  end
end

# Run specific tags:
# mix test --only unit
# mix test --exclude slow
# mix test --exclude integration
# mix test --only unit --exclude slow
```

---

## Step 175: Mocking กับ Mox

```elixir
# lib/my_app/http_client.ex
defmodule MyApp.HttpClient do
  @callback get(String.t()) :: {:ok, map()} | {:error, term()}
  @callback post(String.t(), map()) :: {:ok, map()} | {:error, term()}
end

# lib/my_app/real_http_client.ex
defmodule MyApp.RealHttpClient do
  @behaviour MyApp.HttpClient
  
  def get(url) do
    case Req.get(url) do
      {:ok, %{status: 200, body: body}} -> {:ok, body}
      {:ok, %{status: status}}          -> {:error, {:status, status}}
      {:error, reason}                  -> {:error, reason}
    end
  end
  
  def post(url, body) do
    case Req.post(url, json: body) do
      {:ok, %{status: 200, body: resp}} -> {:ok, resp}
      {:ok, %{status: status}}          -> {:error, {:status, status}}
      {:error, reason}                  -> {:error, reason}
    end
  end
end

# test/support/mocks.ex
Mox.defmock(MyApp.MockHttpClient, for: MyApp.HttpClient)

# test/test_helper.exs
ExUnit.start()
Mox.defmock(MyApp.MockHttpClient, for: MyApp.HttpClient)

# test/my_service_test.exs
defmodule MyServiceTest do
  use ExUnit.Case, async: true
  import Mox
  
  setup :verify_on_exit!
  
  test "fetches user data" do
    MyApp.MockHttpClient
    |> expect(:get, fn "/api/users/1" ->
         {:ok, %{"id" => 1, "name" => "Alice"}}
       end)
    
    assert {:ok, user} = MyService.get_user(1)
    assert user["name"] == "Alice"
  end
  
  test "handles network error" do
    MyApp.MockHttpClient
    |> expect(:get, fn _ -> {:error, :econnrefused} end)
    
    assert {:error, :network_error} = MyService.get_user(1)
  end
  
  test "retries on error" do
    MyApp.MockHttpClient
    |> expect(:get, fn _ -> {:error, :timeout} end)
    |> expect(:get, fn _ -> {:ok, %{"id" => 1}} end)
    
    assert {:ok, _} = MyService.get_user(1)
  end
end

# config/test.exs
config :my_app, :http_client, MyApp.MockHttpClient
```

---

## Step 176: Property-Based Testing ด้วย StreamData

```elixir
defmodule PropertyTest do
  use ExUnit.Case
  use ExUnitProperties  # from stream_data
  
  property "string length is always non-negative" do
    check all str <- string(:alphanumeric) do
      assert String.length(str) >= 0
    end
  end
  
  property "list reverse twice returns original" do
    check all list <- list_of(integer()) do
      assert list |> Enum.reverse() |> Enum.reverse() == list
    end
  end
  
  property "adding to empty list gives singleton" do
    check all item <- integer() do
      assert [item] == [item | []]
      assert length([item]) == 1
    end
  end
  
  # Custom generators
  property "user email is valid" do
    user_gen = gen all name <- string(:alphanumeric, min_length: 1),
                       domain <- member_of(["gmail.com", "yahoo.com", "outlook.com"]) do
      %{name: name, email: "#{name}@#{domain}"}
    end
    
    check all user <- user_gen do
      assert user.email =~ ~r/@/
      assert String.contains?(user.email, ".")
    end
  end
end
```

---

## Step 177: Test กับ Databases (Ecto)

```elixir
defmodule MyApp.AccountsTest do
  use MyApp.DataCase  # มาจาก Phoenix test helpers
  
  alias MyApp.Accounts
  alias MyApp.Accounts.User
  
  describe "create_user/1" do
    test "with valid data creates user" do
      attrs = %{name: "Alice", email: "alice@example.com", password: "secret123"}
      
      assert {:ok, %User{} = user} = Accounts.create_user(attrs)
      assert user.name == "Alice"
      assert user.email == "alice@example.com"
    end
    
    test "with invalid email returns error" do
      attrs = %{name: "Alice", email: "not-an-email", password: "secret"}
      
      assert {:error, %Ecto.Changeset{} = changeset} = Accounts.create_user(attrs)
      assert %{email: ["has invalid format"]} = errors_on(changeset)
    end
    
    test "with duplicate email returns error" do
      user_fixture(%{email: "taken@example.com"})
      
      assert {:error, changeset} = Accounts.create_user(%{
        name: "Bob",
        email: "taken@example.com",
        password: "secret"
      })
      assert %{email: ["has already been taken"]} = errors_on(changeset)
    end
  end
  
  # Helper
  defp user_fixture(attrs \\ %{}) do
    defaults = %{name: "Test", email: "test@example.com", password: "secret123"}
    {:ok, user} = Accounts.create_user(Map.merge(defaults, attrs))
    user
  end
end

# test/support/data_case.ex
defmodule MyApp.DataCase do
  use ExUnit.CaseTemplate
  
  using do
    quote do
      alias MyApp.Repo
      import Ecto
      import Ecto.Changeset
      import Ecto.Query
      import MyApp.DataCase
    end
  end
  
  setup tags do
    MyApp.DataCase.setup_sandbox(tags)
  end
  
  def setup_sandbox(tags) do
    pid = Ecto.Adapters.SQL.Sandbox.start_owner!(
      MyApp.Repo,
      shared: not tags[:async]
    )
    on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
    :ok
  end
  
  def errors_on(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## Step 178: Testing GenServer

```elixir
defmodule CounterServerTest do
  use ExUnit.Case
  
  alias CounterServer
  
  setup do
    {:ok, server} = start_supervised({CounterServer, initial: 0})
    {:ok, server: server}
  end
  
  test "starts with initial value", %{server: server} do
    assert CounterServer.value(server) == 0
  end
  
  test "increments by 1", %{server: server} do
    :ok = CounterServer.increment(server)
    assert CounterServer.value(server) == 1
  end
  
  test "increments by custom amount", %{server: server} do
    :ok = CounterServer.increment(server, 5)
    assert CounterServer.value(server) == 5
  end
  
  test "resets to zero", %{server: server} do
    CounterServer.increment(server, 10)
    CounterServer.reset(server)
    assert CounterServer.value(server) == 0
  end
  
  test "survives crash and restarts", %{server: server} do
    CounterServer.increment(server, 5)
    
    # สั่งให้ crash
    Process.exit(server, :kill)
    
    # รอ restart
    Process.sleep(50)
    
    # ค่าควร reset (หรือ recover ถ้ามี persistence)
    new_server = Process.whereis(CounterServer)
    assert new_server != nil
    assert CounterServer.value(new_server) == 0
  end
end
```

---

## Step 179: Code Coverage

```elixir
# mix.exs
def project do
  [
    test_coverage: [tool: ExCoveralls],
    preferred_cli_env: [
      "coveralls": :test,
      "coveralls.detail": :test,
      "coveralls.post": :test,
      "coveralls.html": :test
    ]
  ]
end

defp deps do
  [{:excoveralls, "~> 0.18", only: :test}]
end
```

```bash
# Run with coverage
mix coveralls
mix coveralls.html    # generates HTML report
mix coveralls.detail  # detailed line-by-line

# Coverage thresholds (CI)
# coveralls.json
{
  "coverage_options": {
    "minimum_coverage": 80
  }
}
```

---

## Step 180: Complete Test Suite Example

```elixir
defmodule ShoppingCart do
  defstruct items: [], discount: 0
  
  def new, do: %__MODULE__{}
  
  def add_item(%__MODULE__{} = cart, item) do
    %{cart | items: [item | cart.items]}
  end
  
  def remove_item(%__MODULE__{} = cart, item_id) do
    %{cart | items: Enum.reject(cart.items, &(&1.id == item_id))}
  end
  
  def total(%__MODULE__{items: items, discount: disc}) do
    subtotal = Enum.sum(Enum.map(items, &(&1.price * &1.quantity)))
    subtotal * (1 - disc / 100)
  end
  
  def apply_discount(%__MODULE__{} = cart, percent) when percent in 0..100 do
    {:ok, %{cart | discount: percent}}
  end
  def apply_discount(_, _), do: {:error, :invalid_discount}
  
  def item_count(%__MODULE__{items: items}) do
    Enum.sum(Enum.map(items, & &1.quantity))
  end
end

defmodule ShoppingCartTest do
  use ExUnit.Case, async: true
  
  alias ShoppingCart
  
  @item1 %{id: 1, name: "Widget", price: 10.0, quantity: 2}
  @item2 %{id: 2, name: "Gadget", price: 25.0, quantity: 1}
  
  describe "new/0" do
    test "creates empty cart" do
      cart = ShoppingCart.new()
      assert cart.items == []
      assert cart.discount == 0
    end
  end
  
  describe "add_item/2" do
    test "adds item to cart" do
      cart = ShoppingCart.new() |> ShoppingCart.add_item(@item1)
      assert length(cart.items) == 1
      assert hd(cart.items) == @item1
    end
    
    test "can add multiple items" do
      cart = ShoppingCart.new()
             |> ShoppingCart.add_item(@item1)
             |> ShoppingCart.add_item(@item2)
      assert length(cart.items) == 2
    end
  end
  
  describe "remove_item/2" do
    setup do
      cart = ShoppingCart.new()
             |> ShoppingCart.add_item(@item1)
             |> ShoppingCart.add_item(@item2)
      {:ok, cart: cart}
    end
    
    test "removes item by id", %{cart: cart} do
      new_cart = ShoppingCart.remove_item(cart, 1)
      assert length(new_cart.items) == 1
      refute Enum.any?(new_cart.items, &(&1.id == 1))
    end
    
    test "ignores missing item", %{cart: cart} do
      new_cart = ShoppingCart.remove_item(cart, 999)
      assert length(new_cart.items) == 2
    end
  end
  
  describe "total/1" do
    test "calculates total correctly" do
      cart = ShoppingCart.new()
             |> ShoppingCart.add_item(@item1)  # 10 * 2 = 20
             |> ShoppingCart.add_item(@item2)  # 25 * 1 = 25
      
      assert_in_delta ShoppingCart.total(cart), 45.0, 0.001
    end
    
    test "applies discount" do
      cart = ShoppingCart.new()
             |> ShoppingCart.add_item(@item1)  # 20
      
      {:ok, discounted} = ShoppingCart.apply_discount(cart, 10)  # 10% off
      assert_in_delta ShoppingCart.total(discounted), 18.0, 0.001
    end
    
    test "empty cart total is 0" do
      assert ShoppingCart.total(ShoppingCart.new()) == 0
    end
  end
  
  describe "apply_discount/2" do
    test "accepts valid discount" do
      cart = ShoppingCart.new()
      assert {:ok, _} = ShoppingCart.apply_discount(cart, 0)
      assert {:ok, _} = ShoppingCart.apply_discount(cart, 50)
      assert {:ok, _} = ShoppingCart.apply_discount(cart, 100)
    end
    
    test "rejects invalid discount" do
      cart = ShoppingCart.new()
      assert {:error, :invalid_discount} = ShoppingCart.apply_discount(cart, -1)
      assert {:error, :invalid_discount} = ShoppingCart.apply_discount(cart, 101)
    end
  end
end
```

---

## สรุป Part 16

✅ **Step 171** - ExUnit basics  
✅ **Step 172** - Assertions  
✅ **Step 173** - Setup callbacks  
✅ **Step 174** - Tags และ filters  
✅ **Step 175** - Mocking ด้วย Mox  
✅ **Step 176** - Property-based testing  
✅ **Step 177** - Database testing  
✅ **Step 178** - GenServer testing  
✅ **Step 179** - Code coverage  
✅ **Step 180** - Complete test suite  

➡️ [Part 17: Protocols และ Behaviours](./part-17-protocols-behaviours.md)
