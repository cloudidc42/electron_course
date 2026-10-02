# Part 99: Elixir Ecosystem & Libraries (Steps 1051-1060)

## Step 1051: Req - Modern HTTP Client

```elixir
# mix.exs: {:req, "~> 0.5"}

defmodule MyApp.HTTP do
  # Req is the modern HTTP client for Elixir

  # Basic requests
  def get_json(url) do
    case Req.get(url) do
      {:ok, %{status: 200, body: body}} -> {:ok, body}
      {:ok, %{status: status}}          -> {:error, {:http, status}}
      {:error, reason}                  -> {:error, reason}
    end
  end

  # With options and middleware
  def api_client(base_url, token) do
    Req.new(
      base_url: base_url,
      auth: {:bearer, token},
      headers: [{"content-type", "application/json"}],
      retry: :transient,
      max_retries: 3,
      receive_timeout: 30_000,
      connect_options: [timeout: 5_000]
    )
  end

  def list_users(client) do
    Req.get(client, url: "/users")
  end

  def create_resource(client, data) do
    Req.post(client, url: "/resources", json: data)
  end

  # Streaming response
  def stream_large_file(url, output_path) do
    File.open!(output_path, [:write], fn file ->
      Req.get!(url,
        into: fn {:data, chunk}, acc ->
          IO.binwrite(file, chunk)
          {:cont, acc}
        end
      )
    end)
  end

  # Custom middleware (Req plugin)
  def with_logging(req) do
    Req.Request.append_request_steps(req,
      log_request: fn req ->
        Logger.debug("HTTP #{req.method} #{req.url}")
        req
      end
    )
    |> Req.Request.append_response_steps(
      log_response: fn {req, resp} ->
        Logger.debug("HTTP #{resp.status} #{req.url} (#{resp.body |> byte_size()}b)")
        {req, resp}
      end
    )
  end
end
```

---

## Step 1052: Commanded - CQRS/ES Framework

```elixir
# mix.exs: {:commanded, "~> 1.4"}, {:commanded_ecto_projections, "~> 1.3"}

defmodule MyApp.Banking.Application do
  use Commanded.Application, otp_app: :my_app
  router MyApp.Banking.Router
end

# Command
defmodule MyApp.Banking.Commands.DepositMoney do
  @enforce_keys [:account_id, :amount, :currency]
  defstruct [:account_id, :amount, :currency, :reference]
end

# Event
defmodule MyApp.Banking.Events.MoneyDeposited do
  @derive Jason.Encoder
  defstruct [:account_id, :amount, :currency, :balance, :reference, :deposited_at]
end

# Aggregate
defmodule MyApp.Banking.Account do
  @derive Jason.Encoder
  defstruct [:account_id, :balance, :currency, :status]

  alias MyApp.Banking.Commands.DepositMoney
  alias MyApp.Banking.Events.MoneyDeposited

  def execute(%__MODULE__{status: :closed}, %DepositMoney{}) do
    {:error, :account_closed}
  end

  def execute(%__MODULE__{} = account, %DepositMoney{} = cmd) do
    %MoneyDeposited{
      account_id:   cmd.account_id,
      amount:       cmd.amount,
      currency:     cmd.currency,
      balance:      account.balance + cmd.amount,
      reference:    cmd.reference,
      deposited_at: DateTime.utc_now()
    }
  end

  def apply(%__MODULE__{} = account, %MoneyDeposited{} = event) do
    %{account | balance: event.balance}
  end
end

# Router
defmodule MyApp.Banking.Router do
  use Commanded.Commands.Router
  
  dispatch MyApp.Banking.Commands.DepositMoney,
    to: MyApp.Banking.Account,
    identity: :account_id
end
```

---

## Step 1053: Ash Framework

```elixir
# mix.exs: {:ash, "~> 3.0"}

defmodule MyApp.Blog.Post do
  use Ash.Resource, domain: MyApp.Blog, data_layer: AshPostgres.DataLayer

  postgres do
    table "posts"
    repo MyApp.Repo
  end

  attributes do
    uuid_primary_key :id
    attribute :title,   :string, allow_nil?: false
    attribute :body,    :string, allow_nil?: false
    attribute :status,  :atom,   values: [:draft, :published, :archived], default: :draft
    attribute :views,   :integer, default: 0
    timestamps()
  end

  relationships do
    belongs_to :author, MyApp.Accounts.User
    has_many :comments, MyApp.Blog.Comment
  end

  actions do
    defaults [:read, :destroy]

    create :publish do
      accept [:title, :body]
      change set_attribute(:status, :published)
    end

    update :archive do
      change set_attribute(:status, :archived)
    end

    read :published do
      filter expr(status == :published)
    end
  end

  calculations do
    calculate :reading_time, :integer, expr(
      fragment("ceil(length(?) / 200.0)", body)
    )
  end

  aggregates do
    count :comment_count, :comments
  end
end

# Domain
defmodule MyApp.Blog do
  use Ash.Domain
  resources do
    resource MyApp.Blog.Post
    resource MyApp.Blog.Comment
  end
end

# Usage
MyApp.Blog.Post
|> Ash.Query.for_read(:published)
|> Ash.Query.load([:comment_count, :reading_time])
|> Ash.read!()
```

---

## Step 1054: NimbleCSV & File Processing

```elixir
# mix.exs: {:nimble_csv, "~> 1.2"}

defmodule MyApp.CSV do
  NimbleCSV.define(TSVParser, separator: "\t", escape: "\"")
  NimbleCSV.define(CSVParser, separator: ",", escape: "\"")

  def import_products(path) do
    path
    |> File.stream!()
    |> CSVParser.parse_stream(skip_headers: false)
    |> Stream.transform(nil, fn
      headers, nil -> {[], headers}
      row, headers ->
        product = Enum.zip(headers, row) |> Map.new()
        {[product], headers}
    end)
    |> Stream.chunk_every(100)
    |> Stream.each(fn batch ->
      Enum.each(batch, &import_product/1)
    end)
    |> Stream.run()
  end

  def export_orders(orders, path) do
    headers = ["ID", "Customer", "Total", "Status", "Date"]

    rows = Enum.map(orders, fn order ->
      [
        order.id,
        order.customer.name,
        "#{order.total / 100}.#{rem(order.total, 100)}",
        order.status,
        Date.to_iso8601(order.inserted_at)
      ]
    end)

    File.open!(path, [:write], fn file ->
      [headers | rows]
      |> CSVParser.dump_to_iodata()
      |> IO.binwrite(file, ...)
    end)
  end

  defp import_product(attrs) do
    %MyApp.Product{}
    |> MyApp.Product.changeset(attrs)
    |> MyApp.Repo.insert(on_conflict: :replace_all, conflict_target: :sku)
  end
end
```

---

## Step 1055: Cloak - Field Encryption

```elixir
# mix.exs: {:cloak_ecto, "~> 1.3"}

defmodule MyApp.Vault do
  use Cloak.Vault, otp_app: :my_app

  @impl Cloak.Vault
  def init(config) do
    config = Keyword.put_new(config, :ciphers, [
      default: {Cloak.Ciphers.AES.GCM, tag: "AES.GCM.V1", key: decode_key()}
    ])
    {:ok, config}
  end

  defp decode_key do
    Application.fetch_env!(:my_app, :cloak_key) |> Base.decode64!()
  end
end

# Encrypted Ecto types
defmodule MyApp.Encrypted.Binary do
  use Cloak.Ecto.Binary, vault: MyApp.Vault
end

defmodule MyApp.Encrypted.UTF8 do
  use Cloak.Ecto.UTF8, vault: MyApp.Vault
end

# Schema with encrypted fields
defmodule MyApp.Patient do
  use Ecto.Schema

  schema "patients" do
    field :name,         MyApp.Encrypted.UTF8
    field :email,        MyApp.Encrypted.UTF8
    field :ssn,          MyApp.Encrypted.UTF8
    field :medical_notes, MyApp.Encrypted.Binary
    field :ssn_hash,     Cloak.Ecto.SHA256  # for lookups

    timestamps()
  end

  def changeset(patient, attrs) do
    patient
    |> Ecto.Changeset.cast(attrs, [:name, :email, :ssn, :medical_notes])
    |> Ecto.Changeset.validate_required([:name, :ssn])
    |> put_hashed_fields()
  end

  defp put_hashed_fields(changeset) do
    changeset
    |> Ecto.Changeset.put_change(:ssn_hash, Ecto.Changeset.get_field(changeset, :ssn))
  end
end

# Lookup by encrypted field
def find_by_ssn(ssn) do
  # Use hash for lookup, not the encrypted value
  hash = :crypto.hash(:sha256, ssn)
  MyApp.Repo.get_by(MyApp.Patient, ssn_hash: hash)
end
```

---

## Step 1056: Finch - HTTP Connection Pooling

```elixir
# mix.exs: {:finch, "~> 0.18"}

defmodule MyApp.Finch do
  def child_spec do
    {Finch, name: __MODULE__, pools: %{
      "https://api.stripe.com" => [size: 10, count: 2],
      "https://api.sendgrid.com" => [size: 5],
      :default => [size: 10]
    }}
  end
end

# Low-level HTTP with Finch
defmodule MyApp.StripeHTTP do
  def request(method, path, body \\ nil) do
    headers = [
      {"Authorization", "Bearer #{Application.get_env(:stripity_stripe, :api_key)}"},
      {"Content-Type", "application/x-www-form-urlencoded"}
    ]

    body_data = if body, do: URI.encode_query(body), else: ""

    :finch
    |> Finch.build(method, "https://api.stripe.com#{path}", headers, body_data)
    |> Finch.request(MyApp.Finch, receive_timeout: 30_000)
    |> case do
      {:ok, %{status: status, body: body}} when status in 200..299 ->
        {:ok, Jason.decode!(body)}
      {:ok, %{body: body}} ->
        {:error, Jason.decode!(body)}
      {:error, _} = error ->
        error
    end
  end
end
```

---

## Step 1057: Broadway - Data Processing

```elixir
# mix.exs: {:broadway, "~> 1.1"}, {:broadway_sqs, "~> 0.7"}

defmodule MyApp.OrderProcessor do
  use Broadway

  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {BroadwaySQS.Producer, queue_url: System.get_env("ORDER_QUEUE_URL")},
        concurrency: 1
      ],
      processors: [
        default: [concurrency: 10]
      ],
      batchers: [
        db: [concurrency: 2, batch_size: 25, batch_timeout: 2_000],
        notify: [concurrency: 1, batch_size: 50, batch_timeout: 5_000]
      ]
    )
  end

  @impl true
  def handle_message(_, message, _) do
    order = Jason.decode!(message.data)
    
    case validate_order(order) do
      :ok ->
        message
        |> Message.put_data(order)
        |> Message.put_batcher(:db)
      {:error, reason} ->
        Message.failed(message, reason)
    end
  end

  @impl true
  def handle_batch(:db, messages, _, _) do
    orders = Enum.map(messages, & &1.data)
    
    case MyApp.Repo.insert_all("orders", orders, returning: [:id]) do
      {_, inserted} ->
        Enum.map(messages, fn msg ->
          Message.put_data(msg, :ok)
        end)
    end
  rescue
    e ->
      Enum.map(messages, fn msg -> Message.failed(msg, Exception.message(e)) end)
  end

  @impl true
  def handle_batch(:notify, messages, _, _) do
    Enum.each(messages, fn message ->
      MyApp.Notifications.order_confirmed(message.data)
    end)
    messages
  end

  defp validate_order(%{"user_id" => id, "items" => items}) when is_binary(id) and length(items) > 0, do: :ok
  defp validate_order(_), do: {:error, :invalid_order}
end
```

---

## Step 1058: ExMachina - Test Factories

```elixir
# mix.exs: {:ex_machina, "~> 2.8", only: :test}

defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.Accounts.User{
      name:              sequence(:name, &"User #{&1}"),
      email:             sequence(:email, &"user#{&1}@example.com"),
      hashed_password:   Bcrypt.hash_pwd_salt("password123"),
      role:              :user,
      confirmed_at:      DateTime.utc_now()
    }
  end

  def admin_factory do
    struct!(user_factory(), role: :admin)
  end

  def product_factory do
    %MyApp.Products.Product{
      name:        sequence(:product_name, &"Product #{&1}"),
      description: "A test product",
      price:       :rand.uniform(10_000) + 100,
      stock:       :rand.uniform(100),
      sku:         sequence(:sku, &"SKU-#{&1}"),
      status:      :active
    }
  end

  def order_factory do
    user = insert(:user)
    %MyApp.Orders.Order{
      user:        user,
      status:      :pending,
      total:       :rand.uniform(50_000) + 1000,
      items:       build_list(2, :order_item),
      inserted_at: DateTime.utc_now()
    }
  end

  def order_item_factory do
    %MyApp.Orders.OrderItem{
      product:  build(:product),
      quantity: :rand.uniform(5) + 1,
      price:    :rand.uniform(5000) + 100
    }
  end
end

# Usage in tests
defmodule MyApp.OrderTest do
  use MyApp.DataCase
  import MyApp.Factory

  test "canceling an order" do
    user  = insert(:user)
    order = insert(:order, user: user, status: :confirmed)

    assert {:ok, _} = MyApp.Orders.cancel(order.id, user)
    assert MyApp.Repo.get!(MyApp.Orders.Order, order.id).status == :canceled
  end

  test "admin can access all orders" do
    insert_list(5, :order)
    admin = insert(:admin)

    conn = build_conn() |> log_in_user(admin)
    response = get(conn, ~p"/admin/orders")
    assert response.status == 200
  end
end
```

---

## Step 1059: Credo - Code Quality

```elixir
# .credo.exs
%{
  configs: [
    %{
      name: "default",
      files: %{
        included: ["lib/", "src/", "web/", "apps/*/lib/"],
        excluded: ["_build/", "deps/", "test/"]
      },
      strict: true,
      color: true,
      checks: %{
        enabled: [
          # Code readability
          {Credo.Check.Readability.ModuleDoc, []},
          {Credo.Check.Readability.FunctionNames, []},
          {Credo.Check.Readability.MaxLineLength, [max_length: 120]},
          {Credo.Check.Readability.AliasOrder, []},
          
          # Code design
          {Credo.Check.Design.AliasUsage, [if_nested_deeper_than: 2]},
          {Credo.Check.Design.DuplicatedCode, [mass_threshold: 40]},
          
          # Refactoring opportunities
          {Credo.Check.Refactor.Nesting, [max_nesting: 3]},
          {Credo.Check.Refactor.LongQuoteBlocks, []},
          {Credo.Check.Refactor.MatchInCondition, []},
          
          # Warnings
          {Credo.Check.Warning.LazyLogging, []},
          {Credo.Check.Warning.UnusedEnumOperation, []},
          {Credo.Check.Warning.UnusedStringOperation, []},

          # Security
          {Credo.Check.Warning.Dbg, []},
          {Credo.Check.Warning.IoInspect, []}
        ],
        disabled: [
          {Credo.Check.Readability.Specs, false}
        ]
      }
    }
  ]
}

# Ignore specific issues inline
defmodule MyApp.Legacy do
  # credo:disable-for-this-file Credo.Check.Refactor.Nesting

  def complex_legacy_function(data) do
    # credo:disable-for-next-line Credo.Check.Readability.MaxLineLength
    Enum.reduce(data, %{}, fn item, acc -> Map.update(acc, item.category, [item], &[item | &1]) end)
  end
end
```

---

## Step 1060: Dialyxir & Type Specs

```elixir
# mix.exs: {:dialyxir, "~> 1.4", only: [:dev, :test], runtime: false}
# mix dialyzer

defmodule MyApp.TypedAPI do
  @moduledoc "Demonstrates comprehensive typespecs"

  @type user_id    :: pos_integer()
  @type email      :: String.t()
  @type role       :: :admin | :user | :moderator
  @type user       :: %{id: user_id(), email: email(), role: role()}
  @type result(t)  :: {:ok, t} | {:error, term()}
  @type pagination :: %{page: pos_integer(), page_size: pos_integer(), total: non_neg_integer()}

  @spec get_user(user_id()) :: result(user())
  def get_user(id) when is_integer(id) and id > 0 do
    case MyApp.Repo.get(MyApp.Accounts.User, id) do
      nil  -> {:error, :not_found}
      user -> {:ok, %{id: user.id, email: user.email, role: user.role}}
    end
  end

  @spec list_users(Keyword.t()) :: {[user()], pagination()}
  def list_users(opts \\ []) do
    page      = Keyword.get(opts, :page, 1)
    page_size = Keyword.get(opts, :page_size, 20)

    query = from u in MyApp.Accounts.User,
              select: %{id: u.id, email: u.email, role: u.role},
              limit: ^page_size,
              offset: ^((page - 1) * page_size)

    users = MyApp.Repo.all(query)
    total = MyApp.Repo.aggregate(MyApp.Accounts.User, :count)

    {users, %{page: page, page_size: page_size, total: total}}
  end

  @spec update_role(user_id(), role()) :: result(user())
  def update_role(user_id, role) when role in [:admin, :user, :moderator] do
    with {:ok, user} <- get_user(user_id),
         {:ok, updated} <- do_update_role(user.id, role) do
      {:ok, updated}
    end
  end

  @spec do_update_role(user_id(), role()) :: result(user())
  defp do_update_role(id, role) do
    case MyApp.Repo.update_all(
      from(u in MyApp.Accounts.User, where: u.id == ^id),
      set: [role: role]
    ) do
      {1, _} -> get_user(id)
      {0, _} -> {:error, :not_found}
    end
  end
end
```

---

## สรุป Part 99

✅ **Step 1051** - Req (modern HTTP client)  
✅ **Step 1052** - Commanded (CQRS/ES framework)  
✅ **Step 1053** - Ash Framework  
✅ **Step 1054** - NimbleCSV & file processing  
✅ **Step 1055** - Cloak (field encryption)  
✅ **Step 1056** - Finch (HTTP connection pooling)  
✅ **Step 1057** - Broadway (data processing)  
✅ **Step 1058** - ExMachina (test factories)  
✅ **Step 1059** - Credo (code quality)  
✅ **Step 1060** - Dialyxir & type specs  

➡️ [Part 100: Course Capstone](./part-100-capstone.md)
