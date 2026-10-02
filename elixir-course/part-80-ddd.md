# Part 80: Domain-Driven Design (Steps 861-880)

## Step 861: Bounded Contexts

```elixir
# In DDD, a Bounded Context is an explicit boundary within which a model applies
# Each context has its own schema, language, and rules

defmodule MyApp.Sales do
  # Sales context: deals with orders, cart, pricing
  
  defmodule Order do
    use Ecto.Schema
    
    schema "sales_orders" do
      field :status,      Ecto.Enum, values: [:draft, :confirmed, :shipped, :delivered]
      field :total,       :decimal
      field :currency,    :string
      field :customer_id, :integer  # Reference to Accounts context
      
      has_many :line_items, MyApp.Sales.LineItem
      timestamps()
    end
  end
end

defmodule MyApp.Accounts do
  # Accounts context: customers, authentication, profiles
  
  defmodule Customer do
    use Ecto.Schema
    
    schema "accounts_customers" do
      field :name,  :string
      field :email, :string
      field :tier,  Ecto.Enum, values: [:standard, :premium, :enterprise]
      timestamps()
    end
  end
end

# Anti-corruption layer: translate between contexts
defmodule MyApp.Sales.CustomerInfo do
  # Sales context's view of a customer (only what Sales needs)
  defstruct [:id, :name, :email, :tier, :discount_pct]
  
  def from_account(%MyApp.Accounts.Customer{} = customer) do
    %__MODULE__{
      id:           customer.id,
      name:         customer.name,
      email:        customer.email,
      tier:         customer.tier,
      discount_pct: discount_for_tier(customer.tier)
    }
  end
  
  defp discount_for_tier(:standard),   do: 0
  defp discount_for_tier(:premium),    do: 10
  defp discount_for_tier(:enterprise), do: 20
end
```

---

## Step 862: Aggregates

```elixir
defmodule MyApp.Sales.Order do
  # Aggregate Root: enforces business invariants
  
  use Ecto.Schema
  import Ecto.Changeset

  schema "orders" do
    field :status, Ecto.Enum, values: [:cart, :placed, :paid, :shipped, :delivered, :cancelled]
    field :total,  :decimal,  default: Decimal.new(0)
    
    has_many :items, MyApp.Sales.OrderItem, preload_order: [inserted_at: :asc]
    timestamps()
  end

  # Business operations (commands) -- pure Ecto.Multi
  def place_order(order) when order.status == :cart do
    Ecto.Multi.new()
    |> Ecto.Multi.run(:validate, fn _repo, _changes ->
      cond do
        Enum.empty?(order.items) -> {:error, :no_items}
        order.total <= 0         -> {:error, :invalid_total}
        true                     -> {:ok, :valid}
      end
    end)
    |> Ecto.Multi.update(:order, change(order, status: :placed))
    |> Ecto.Multi.run(:reserve_stock, fn _repo, %{order: o} ->
      MyApp.Inventory.reserve(o.items)
    end)
  end

  def cancel(order) when order.status in [:placed, :paid] do
    Ecto.Multi.new()
    |> Ecto.Multi.update(:order, change(order, status: :cancelled))
    |> Ecto.Multi.run(:release_stock, fn _repo, %{order: o} ->
      MyApp.Inventory.release(o.items)
    end)
    |> Ecto.Multi.run(:refund, fn _repo, %{order: o} ->
      if o.status == :paid, do: MyApp.Payments.refund(o), else: {:ok, :not_paid}
    end)
  end
end
```

---

## Step 863: Value Objects

```elixir
defmodule MyApp.Money do
  # Value object: equality by value, immutable
  @enforce_keys [:amount, :currency]
  defstruct [:amount, :currency]

  def new(amount, currency) when is_binary(currency) do
    %__MODULE__{
      amount:   Decimal.new(amount),
      currency: String.upcase(currency)
    }
  end

  def add(%__MODULE__{currency: c} = a, %__MODULE__{currency: c} = b) do
    %{a | amount: Decimal.add(a.amount, b.amount)}
  end
  def add(_, _), do: raise ArgumentError, "cannot add different currencies"

  def multiply(%__MODULE__{} = money, factor) do
    %{money | amount: Decimal.mult(money.amount, Decimal.new(factor))}
  end

  def compare(%__MODULE__{amount: a}, %__MODULE__{amount: b}) do
    Decimal.compare(a, b)
  end

  def zero?(money), do: Decimal.eq?(money.amount, 0)

  def to_string(%__MODULE__{amount: amount, currency: currency}) do
    "#{currency} #{Decimal.round(amount, 2)}"
  end
end

defmodule MyApp.Address do
  @enforce_keys [:street, :city, :country]
  defstruct [:street, :city, :state, :postal_code, :country]

  def valid?(%__MODULE__{} = addr) do
    addr.street != "" and addr.city != "" and addr.country != ""
  end
end
```

---

## Step 864: Domain Events

```elixir
defmodule MyApp.DomainEvent do
  defmacro __using__(_opts) do
    quote do
      @enforce_keys [:id, :occurred_at, :aggregate_id]
      defstruct @enforce_keys ++ [:payload]

      def new(aggregate_id, payload \\ %{}) do
        %__MODULE__{
          id:           Ecto.UUID.generate(),
          occurred_at:  DateTime.utc_now(),
          aggregate_id: aggregate_id,
          payload:      payload
        }
      end
    end
  end
end

defmodule MyApp.Events.OrderPlaced do
  use MyApp.DomainEvent
  # aggregate_id = order_id
  # payload = %{customer_id: _, total: _, items: [...]}
end

defmodule MyApp.Events.PaymentReceived do
  use MyApp.DomainEvent
  # payload = %{amount: _, payment_method: _}
end

defmodule MyApp.Events.OrderShipped do
  use MyApp.DomainEvent
  # payload = %{tracking_number: _, carrier: _}
end

# Domain event publisher
defmodule MyApp.DomainEvents do
  def publish(event) do
    # Save to event log
    MyApp.Repo.insert!(%MyApp.EventLog{
      event_type:  event.__struct__ |> Module.split() |> List.last(),
      aggregate_id: event.aggregate_id,
      payload:      event.payload,
      occurred_at:  event.occurred_at
    })

    # Dispatch to handlers
    Phoenix.PubSub.broadcast(MyApp.PubSub, "domain_events", {:domain_event, event})
  end
end
```

---

## Step 865: Repository Pattern

```elixir
defmodule MyApp.OrderRepository do
  # Encapsulates data access for the Order aggregate
  
  import Ecto.Query
  alias MyApp.{Repo, Sales.Order}

  def find(id) do
    case Repo.get(Order, id) |> Repo.preload([:items]) do
      nil   -> {:error, :not_found}
      order -> {:ok, order}
    end
  end

  def find_by_customer(customer_id, opts \\ []) do
    page = Keyword.get(opts, :page, 1)
    per  = Keyword.get(opts, :per_page, 20)

    Order
    |> where(customer_id: ^customer_id)
    |> order_by(desc: :inserted_at)
    |> preload(:items)
    |> Repo.paginate(page: page, page_size: per)
  end

  def find_pending do
    Order
    |> where(status: :placed)
    |> where([o], o.inserted_at < ago(7, "day"))
    |> Repo.all()
  end

  def save(%Order{id: nil} = order) do
    Repo.insert(order)
  end

  def save(%Order{} = order) do
    Repo.update(order)
  end

  def delete(%Order{status: status}) when status in [:shipped, :delivered] do
    {:error, :cannot_delete_fulfilled_order}
  end

  def delete(%Order{} = order) do
    Repo.delete(order)
  end
end
```

---

## Step 866: Domain Services

```elixir
defmodule MyApp.PricingService do
  # Domain service: logic that doesn't naturally fit in one entity

  def calculate_order_total(items, customer) do
    subtotal = items
      |> Enum.map(&line_item_total/1)
      |> Enum.reduce(MyApp.Money.new(0, "USD"), &MyApp.Money.add/2)

    discount = apply_discount(subtotal, customer.tier)
    tax      = calculate_tax(subtotal, customer.address)

    subtotal
    |> MyApp.Money.add(tax)
    |> MyApp.Money.add(discount)  # discount is negative
  end

  defp line_item_total(%{quantity: qty, unit_price: price}) do
    MyApp.Money.multiply(price, qty)
  end

  defp apply_discount(subtotal, :premium) do
    subtotal |> MyApp.Money.multiply(-0.10)
  end
  defp apply_discount(subtotal, :enterprise) do
    subtotal |> MyApp.Money.multiply(-0.20)
  end
  defp apply_discount(_, :standard), do: MyApp.Money.new(0, "USD")

  defp calculate_tax(subtotal, %{country: "US", state: state}) do
    rate = US.TaxRates.rate_for(state)
    MyApp.Money.multiply(subtotal, rate)
  end
  defp calculate_tax(_, _), do: MyApp.Money.new(0, "USD")
end

defmodule MyApp.ShippingService do
  def calculate_shipping(items, destination) do
    total_weight = Enum.sum(Enum.map(items, & &1.weight_kg))
    
    cond do
      destination.country == "US" and total_weight < 1.0 -> MyApp.Money.new("4.99", "USD")
      destination.country == "US"                         -> MyApp.Money.new("9.99", "USD")
      true                                                 -> MyApp.Money.new("24.99", "USD")
    end
  end
end
```

---

## Step 867: Specification Pattern

```elixir
defmodule MyApp.Specification do
  # Composable business rules

  defmacro __using__(_) do
    quote do
      @behaviour MyApp.Specification
      
      def and_spec(other), do: MyApp.Specification.And.new(this: __MODULE__, other: other)
      def or_spec(other),  do: MyApp.Specification.Or.new(this: __MODULE__, other: other)
      def not_spec,        do: MyApp.Specification.Not.new(inner: __MODULE__)
    end
  end

  @callback satisfied_by?(any()) :: boolean()
end

defmodule MyApp.Specifications.PremiumCustomer do
  use MyApp.Specification

  def satisfied_by?(%{tier: tier}), do: tier in [:premium, :enterprise]
end

defmodule MyApp.Specifications.HasAddress do
  use MyApp.Specification
  
  def satisfied_by?(%{address: address}), do: not is_nil(address)
end

defmodule MyApp.Specification.And do
  defstruct [:this, :other]

  def new(opts), do: struct(__MODULE__, opts)

  def satisfied_by?(%{this: this, other: other}, candidate) do
    this.satisfied_by?(candidate) and other.satisfied_by?(candidate)
  end
end

# Usage
eligible_for_express = MyApp.Specifications.PremiumCustomer
  |> struct([])
  |> Map.put(:__struct__, MyApp.Specifications.PremiumCustomer)

MyApp.Specifications.PremiumCustomer.satisfied_by?(customer)
# or with composition
spec = MyApp.Specification.And.new(
  this:  MyApp.Specifications.PremiumCustomer,
  other: MyApp.Specifications.HasAddress
)
MyApp.Specification.And.satisfied_by?(spec, customer)
```

---

## Step 868: Saga Pattern for Long Transactions

```elixir
defmodule MyApp.CheckoutSaga do
  # Coordinates multi-step checkout with compensation

  def execute(order_id, payment_info) do
    saga = new_saga(order_id, payment_info)

    with {:ok, saga} <- step_reserve_inventory(saga),
         {:ok, saga} <- step_process_payment(saga),
         {:ok, saga} <- step_confirm_order(saga),
         {:ok, saga} <- step_send_confirmation(saga) do
      {:ok, saga.order}
    else
      {:error, step, reason, saga} ->
        compensate(saga, step, reason)
    end
  end

  defp step_reserve_inventory(%{order_id: order_id} = saga) do
    case MyApp.Inventory.reserve(order_id) do
      {:ok, reservation} ->
        {:ok, Map.put(saga, :reservation_id, reservation.id)}
      {:error, reason} ->
        {:error, :reserve_inventory, reason, saga}
    end
  end

  defp step_process_payment(%{payment_info: payment, order_id: order_id} = saga) do
    case MyApp.Payments.charge(payment, order_id) do
      {:ok, charge} ->
        {:ok, Map.put(saga, :charge_id, charge.id)}
      {:error, reason} ->
        {:error, :process_payment, reason, saga}
    end
  end

  # Compensation: undo completed steps in reverse order
  defp compensate(saga, failed_at, reason) do
    Logger.error("Checkout saga failed at #{failed_at}: #{inspect(reason)}")

    if Map.has_key?(saga, :charge_id) do
      MyApp.Payments.refund(saga.charge_id)
    end

    if Map.has_key?(saga, :reservation_id) do
      MyApp.Inventory.release(saga.reservation_id)
    end

    {:error, reason}
  end

  defp new_saga(order_id, payment_info) do
    %{order_id: order_id, payment_info: payment_info}
  end
end
```

---

## Step 869: Anti-Corruption Layer

```elixir
defmodule MyApp.Integrations.Stripe do
  # Anti-corruption layer for Stripe API
  # Translates external concepts to our domain

  def charge_customer(customer, amount) do
    # Translate our domain Money to Stripe's cents
    params = %{
      amount:   to_cents(amount),
      currency: String.downcase(amount.currency),
      customer: customer.stripe_customer_id
    }

    case Stripity.Stripe.Charge.create(params) do
      {:ok, stripe_charge} ->
        # Translate Stripe response to our domain Payment
        {:ok, to_domain_payment(stripe_charge)}
      {:error, %Stripe.Error{code: :card_declined}} ->
        {:error, :payment_declined}
      {:error, %Stripe.Error{code: :insufficient_funds}} ->
        {:error, :insufficient_funds}
      {:error, error} ->
        Logger.error("Stripe error: #{inspect(error)}")
        {:error, :payment_failed}
    end
  end

  defp to_cents(%MyApp.Money{amount: amount}) do
    amount |> Decimal.mult(100) |> Decimal.round(0) |> Decimal.to_integer()
  end

  defp to_domain_payment(stripe_charge) do
    %MyApp.Payment{
      external_id:    stripe_charge.id,
      amount:         MyApp.Money.new(stripe_charge.amount / 100, stripe_charge.currency),
      status:         translate_status(stripe_charge.status),
      payment_method: :credit_card,
      processed_at:   DateTime.from_unix!(stripe_charge.created)
    }
  end

  defp translate_status("succeeded"), do: :completed
  defp translate_status("pending"),   do: :pending
  defp translate_status("failed"),    do: :failed
end
```

---

## Step 870: Ubiquitous Language

```elixir
# Enforcing ubiquitous language through naming

defmodule MyApp.Catalog do
  # Use domain expert's language: "publish" not "set_visible"
  
  def publish_product(product) do
    product
    |> Product.changeset(%{status: :published, published_at: DateTime.utc_now()})
    |> Repo.update()
  end

  def discontinue_product(product, reason) do
    product
    |> Product.changeset(%{status: :discontinued, discontinuation_reason: reason})
    |> Repo.update()
  end

  # "markdown" not "apply_discount"
  def markdown(product, percentage) when percentage in 1..99 do
    new_price = Decimal.mult(product.price, Decimal.new((100 - percentage) / 100))
    product
    |> Product.changeset(%{
      sale_price:        Decimal.round(new_price, 2),
      markdown_percent:  percentage,
      on_sale:           true
    })
    |> Repo.update()
  end

  # "restock" not "increase_inventory"
  def restock(product, quantity, supplier_order_id) do
    Ecto.Multi.new()
    |> Ecto.Multi.update(:product, Product.changeset(product, %{stock: product.stock + quantity}))
    |> Ecto.Multi.insert(:movement, %StockMovement{
      product_id: product.id,
      quantity:   quantity,
      type:       :restock,
      reference:  supplier_order_id
    })
    |> Repo.transaction()
  end
end
```

---

## สรุป Part 80

✅ **Step 861** - Bounded contexts  
✅ **Step 862** - Aggregates  
✅ **Step 863** - Value objects  
✅ **Step 864** - Domain events  
✅ **Step 865** - Repository pattern  
✅ **Step 866** - Domain services  
✅ **Step 867** - Specification pattern  
✅ **Step 868** - Saga pattern  
✅ **Step 869** - Anti-corruption layer  
✅ **Step 870** - Ubiquitous language  

➡️ [Part 81: Microservices Architecture](./part-81-microservices.md)
