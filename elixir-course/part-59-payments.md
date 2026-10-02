# Part 59: Payment & Commerce Systems (Steps 641-660)

## Step 641: Payment Gateway Integration

```elixir
# Behaviour for swappable payment providers
defmodule MyApp.Payments.Gateway do
  @callback charge(amount :: integer, currency :: String.t(), source :: String.t(), opts :: keyword()) ::
              {:ok, map()} | {:error, term()}

  @callback refund(charge_id :: String.t(), amount :: integer | nil) ::
              {:ok, map()} | {:error, term()}

  @callback capture(intent_id :: String.t()) ::
              {:ok, map()} | {:error, term()}
end

# Stripe adapter
defmodule MyApp.Payments.StripeGateway do
  @behaviour MyApp.Payments.Gateway

  @impl true
  def charge(amount, currency, source, opts) do
    Stripe.Charge.create(%{
      amount:      amount,
      currency:    currency,
      source:      source,
      description: Keyword.get(opts, :description, ""),
      metadata:    Keyword.get(opts, :metadata, %{})
    })
  end

  @impl true
  def refund(charge_id, amount) do
    params = if amount, do: %{charge: charge_id, amount: amount}, else: %{charge: charge_id}
    Stripe.Refund.create(params)
  end

  @impl true
  def capture(intent_id) do
    Stripe.PaymentIntent.capture(intent_id)
  end
end

# Usage
defmodule MyApp.Payments do
  def gateway, do: Application.get_env(:my_app, :payment_gateway, MyApp.Payments.StripeGateway)

  def charge(amount, currency, source, opts \\ []) do
    gateway().charge(amount, currency, source, opts)
  end
end
```

---

## Step 642: Idempotent Payments

```elixir
defmodule MyApp.Payments.Idempotent do
  # Prevent double-charging by using idempotency keys

  def charge_order(order_id, payment_source) do
    idempotency_key = "order_charge:#{order_id}"

    case get_existing_charge(idempotency_key) do
      {:ok, existing} ->
        # Already charged, return existing result
        {:ok, existing}

      :not_found ->
        with {:ok, charge} <- Stripe.Charge.create(
               %{amount: get_amount(order_id), currency: "thb"},
               [idempotency_key: idempotency_key]
             ),
             {:ok, _} <- save_charge(idempotency_key, charge) do
          {:ok, charge}
        end
    end
  end

  defp get_existing_charge(key) do
    case MyApp.Repo.get_by(PaymentRecord, idempotency_key: key) do
      nil    -> :not_found
      record -> {:ok, record.charge_data}
    end
  end

  defp save_charge(key, charge) do
    %PaymentRecord{}
    |> PaymentRecord.changeset(%{
      idempotency_key: key,
      charge_id:       charge.id,
      charge_data:     charge
    })
    |> MyApp.Repo.insert(on_conflict: :nothing)
  end
end
```

---

## Step 643: Webhook Processing

```elixir
defmodule MyAppWeb.WebhooksController do
  use MyAppWeb, :controller

  # Verify Stripe webhook signature
  def stripe(conn, _params) do
    signature = get_req_header(conn, "stripe-signature") |> List.first()
    secret    = System.get_env("STRIPE_WEBHOOK_SECRET")

    {:ok, body, conn} = Plug.Conn.read_body(conn)

    case Stripe.WebhookPlug.verify_signature(body, signature, secret) do
      {:ok, event} ->
        # Process asynchronously to return 200 quickly
        MyApp.Workers.WebhookWorker.new(%{
          event_type: event.type,
          event_data: Jason.encode!(event.data)
        })
        |> Oban.insert()

        send_resp(conn, 200, "ok")

      {:error, _} ->
        send_resp(conn, 400, "Invalid signature")
    end
  end
end

defmodule MyApp.Workers.WebhookWorker do
  use Oban.Worker, queue: :webhooks, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"event_type" => type, "event_data" => data_json}}) do
    event = Jason.decode!(data_json)
    MyApp.Billing.handle_webhook(type, event)
  end
end
```

---

## Step 644: Shopping Cart

```elixir
defmodule MyApp.Cart do
  use Ecto.Schema
  import Ecto.Query

  schema "carts" do
    field :session_id, :string
    belongs_to :user,  MyApp.Accounts.User
    has_many :items,   MyApp.Cart.Item
    timestamps()
  end

  def get_or_create(session_id, user_id \\ nil) do
    query = from c in __MODULE__,
      where: c.session_id == ^session_id,
      preload: [items: :product]

    case MyApp.Repo.one(query) do
      nil  -> create_cart(session_id, user_id)
      cart -> merge_guest_cart(cart, user_id)
    end
  end

  def add_item(cart, product_id, quantity) do
    existing = Enum.find(cart.items, & &1.product_id == product_id)

    if existing do
      update_item_quantity(existing, existing.quantity + quantity)
    else
      %MyApp.Cart.Item{}
      |> MyApp.Cart.Item.changeset(%{
        cart_id:    cart.id,
        product_id: product_id,
        quantity:   quantity,
        price:      get_product_price(product_id)
      })
      |> MyApp.Repo.insert()
    end
  end

  def total(cart) do
    cart.items
    |> Enum.reduce(Decimal.new("0"), fn item, acc ->
      Decimal.add(acc, Decimal.mult(item.price, Decimal.new(item.quantity)))
    end)
  end

  # Merge guest cart when user logs in
  defp merge_guest_cart(cart, nil), do: cart
  defp merge_guest_cart(cart, user_id) do
    guest_query = from c in __MODULE__,
      where: c.user_id == ^user_id and c.id != ^cart.id,
      preload: :items

    case MyApp.Repo.one(guest_query) do
      nil       -> cart
      user_cart ->
        Enum.each(user_cart.items, fn item ->
          add_item(cart, item.product_id, item.quantity)
        end)
        MyApp.Repo.delete(user_cart)
        MyApp.Repo.reload(cart) |> MyApp.Repo.preload(:items)
    end
  end
end
```

---

## Step 645: Order State Machine

```elixir
defmodule MyApp.Orders.StateMachine do
  # Valid state transitions
  @transitions %{
    :pending     => [:processing, :cancelled],
    :processing  => [:shipped, :cancelled, :refunded],
    :shipped     => [:delivered, :returned],
    :delivered   => [:returned],
    :cancelled   => [],
    :refunded    => [],
    :returned    => [:refunded]
  }

  def can_transition?(from, to) do
    to in Map.get(@transitions, from, [])
  end

  def transition(order, new_status) do
    if can_transition?(order.status, new_status) do
      order
      |> MyApp.Orders.Order.changeset(%{status: new_status})
      |> MyApp.Repo.update()
      |> case do
        {:ok, updated} ->
          after_transition(updated, order.status, new_status)
          {:ok, updated}
        error -> error
      end
    else
      {:error, :invalid_transition}
    end
  end

  defp after_transition(order, _from, :shipped) do
    MyApp.Workers.ShipmentEmailWorker.new(%{order_id: order.id}) |> Oban.insert()
  end

  defp after_transition(order, _from, :delivered) do
    MyApp.Workers.DeliveryEmailWorker.new(%{order_id: order.id}) |> Oban.insert()
    schedule_review_request(order)
  end

  defp after_transition(order, _from, :refunded) do
    MyApp.Payments.issue_refund(order)
  end

  defp after_transition(_order, _from, _to), do: :ok

  defp schedule_review_request(order) do
    MyApp.Workers.ReviewRequestWorker.new(
      %{order_id: order.id},
      scheduled_at: DateTime.add(DateTime.utc_now(), 7 * 86400)  # 7 days later
    )
    |> Oban.insert()
  end
end
```

---

## Step 646: Inventory Management

```elixir
defmodule MyApp.Inventory do
  import Ecto.Query
  alias MyApp.{Repo, Catalog.Product}

  # Atomic stock reservation
  def reserve(product_id, quantity) do
    result = Repo.update_all(
      from(p in Product,
        where: p.id == ^product_id and p.stock >= ^quantity
      ),
      [inc: [stock: -quantity, reserved: quantity]],
      returning: true
    )

    case result do
      {0, []}       -> {:error, :insufficient_stock}
      {1, [product]} -> {:ok, product}
    end
  end

  def release(product_id, quantity) do
    Repo.update_all(
      from(p in Product, where: p.id == ^product_id),
      inc: [reserved: -quantity, stock: quantity]
    )
    :ok
  end

  def confirm_sale(product_id, quantity) do
    Repo.update_all(
      from(p in Product, where: p.id == ^product_id),
      inc: [reserved: -quantity, sold: quantity]
    )
    :ok
  end

  # Low stock alerts
  def check_low_stock do
    from(p in Product,
      where: p.stock <= p.low_stock_threshold and p.active == true
    )
    |> Repo.all()
    |> Enum.each(fn product ->
      MyApp.Workers.LowStockAlertWorker.new(%{product_id: product.id})
      |> Oban.insert()
    end)
  end
end
```

---

## Step 647: Discount & Coupon System

```elixir
defmodule MyApp.Discounts do
  use Ecto.Schema
  import Ecto.Changeset, import Ecto.Query

  schema "coupons" do
    field :code,          :string
    field :type,          Ecto.Enum, values: [:percentage, :fixed_amount, :free_shipping]
    field :value,         :decimal
    field :min_order,     :decimal
    field :max_uses,      :integer
    field :uses_count,    :integer, default: 0
    field :expires_at,    :utc_datetime
    field :active,        :boolean, default: true
    has_many :conditions, MyApp.Discounts.Condition
  end

  def apply(cart, coupon_code) do
    with {:ok, coupon} <- get_valid_coupon(coupon_code),
         :ok           <- check_eligibility(cart, coupon),
         {:ok, _}      <- increment_uses(coupon) do
      discount = calculate_discount(cart, coupon)
      {:ok, discount}
    end
  end

  defp get_valid_coupon(code) do
    now = DateTime.utc_now()
    case Repo.get_by(__MODULE__, code: String.upcase(code)) do
      nil -> {:error, :invalid_coupon}
      %{active: false} -> {:error, :coupon_inactive}
      %{expires_at: exp} when not is_nil(exp) and exp < now -> {:error, :coupon_expired}
      coupon -> {:ok, coupon}
    end
  end

  defp check_eligibility(cart, coupon) do
    total = MyApp.Cart.total(cart)
    if coupon.min_order && Decimal.compare(total, coupon.min_order) == :lt do
      {:error, {:minimum_order, coupon.min_order}}
    else
      :ok
    end
  end

  defp calculate_discount(cart, %{type: :percentage, value: pct}) do
    Decimal.mult(MyApp.Cart.total(cart), Decimal.div(pct, 100))
  end

  defp calculate_discount(_cart, %{type: :fixed_amount, value: amount}) do
    amount
  end

  defp increment_uses(coupon) do
    {1, _} = Repo.update_all(
      from(c in __MODULE__, where: c.id == ^coupon.id and
           (is_nil(c.max_uses) or c.uses_count < c.max_uses)),
      inc: [uses_count: 1]
    )
    {:ok, :incremented}
  rescue
    _ -> {:error, :coupon_exhausted}
  end
end
```

---

## Step 648: Email Receipts

```elixir
defmodule MyApp.Emails.OrderReceipt do
  use Swoosh.Email

  def order_confirmation(order) do
    new()
    |> to({order.user.name, order.user.email})
    |> from({"MyApp Shop", "noreply@myapp.com"})
    |> subject("Order Confirmed ##{order.id}")
    |> render_body("order_confirmation.html", order: order)
  end

  def shipping_notification(order, tracking) do
    new()
    |> to({order.user.name, order.user.email})
    |> from({"MyApp Shop", "noreply@myapp.com"})
    |> subject("Your order has shipped! ##{order.id}")
    |> render_body("shipping_notification.html",
         order:    order,
         tracking: tracking
       )
  end

  defp render_body(email, template, assigns) do
    html = Phoenix.View.render_to_string(
      MyAppWeb.EmailView,
      template,
      assigns
    )

    text = html_to_text(html)

    email
    |> html_body(html)
    |> text_body(text)
  end

  defp html_to_text(html) do
    html
    |> String.replace(~r/<[^>]*>/, "")
    |> String.replace(~r/\s+/, " ")
    |> String.trim()
  end
end

defmodule MyApp.Workers.OrderReceiptWorker do
  use Oban.Worker, queue: :email

  def perform(%Oban.Job{args: %{"order_id" => id}}) do
    order = MyApp.Orders.get!(id) |> MyApp.Repo.preload([:user, items: :product])
    email = MyApp.Emails.OrderReceipt.order_confirmation(order)
    MyApp.Mailer.deliver(email)
  end
end
```

---

## Step 649: Reporting & Analytics

```elixir
defmodule MyApp.Analytics.Sales do
  import Ecto.Query

  def revenue_by_period(period, from, to) do
    truncate = case period do
      :day   -> "day"
      :week  -> "week"
      :month -> "month"
    end

    from(o in MyApp.Orders.Order,
      where: o.status in [:delivered] and
             o.completed_at >= ^from and
             o.completed_at <= ^to,
      group_by: fragment("date_trunc(?, ?)", ^truncate, o.completed_at),
      order_by: fragment("date_trunc(?, ?)", ^truncate, o.completed_at),
      select: %{
        period:  fragment("date_trunc(?, ?)", ^truncate, o.completed_at),
        revenue: sum(o.total),
        count:   count(o.id),
        avg:     avg(o.total)
      }
    )
    |> MyApp.Repo.all()
  end

  def top_products(limit \\ 10, days \\ 30) do
    since = DateTime.add(DateTime.utc_now(), -days * 86400)

    from(oi in MyApp.Orders.OrderItem,
      join: p  in assoc(oi, :product),
      join: o  in assoc(oi, :order),
      where: o.status == :delivered and o.completed_at >= ^since,
      group_by: [p.id, p.name],
      order_by: [desc: sum(oi.quantity)],
      limit: ^limit,
      select: %{
        product_id: p.id,
        name:       p.name,
        sold:       sum(oi.quantity),
        revenue:    sum(fragment("? * ?", oi.quantity, oi.price))
      }
    )
    |> MyApp.Repo.all()
  end

  def cohort_retention(cohort_month) do
    # Analyze user retention by cohort
    first_order = from(o in MyApp.Orders.Order,
      group_by: o.user_id,
      select: %{user_id: o.user_id, first_order: min(o.inserted_at)}
    )

    from(u in subquery(first_order),
      where: fragment("date_trunc('month', ?)", u.first_order) == ^cohort_month,
      join: o in MyApp.Orders.Order, on: o.user_id == u.user_id,
      group_by: fragment("date_trunc('month', ?)", o.inserted_at),
      select: %{
        month: fragment("date_trunc('month', ?)", o.inserted_at),
        users: count(fragment("DISTINCT ?", o.user_id))
      }
    )
    |> MyApp.Repo.all()
  end
end
```

---

## Step 650: Return & Refund Flow

```elixir
defmodule MyApp.Returns do
  alias MyApp.{Repo, Orders, Payments}
  alias Ecto.Multi

  def initiate_return(order_id, items, reason) do
    order = Orders.get!(order_id)

    unless order.status == :delivered do
      {:error, :order_not_delivered}
    else
      Multi.new()
      |> Multi.insert(:return_request, fn _ ->
        %ReturnRequest{}
        |> ReturnRequest.changeset(%{
          order_id:  order_id,
          items:     items,
          reason:    reason,
          status:    :pending
        })
      end)
      |> Multi.run(:notify_warehouse, fn _, %{return_request: req} ->
        MyApp.Workers.ReturnNotificationWorker.new(%{return_id: req.id})
        |> Oban.insert()
      end)
      |> Repo.transaction()
    end
  end

  def approve_return(return_id) do
    return = get_return!(return_id)
    order  = Orders.get!(return.order_id)

    Multi.new()
    |> Multi.update(:approve, ReturnRequest.changeset(return, %{status: :approved}))
    |> Multi.run(:refund, fn _, _ ->
      refund_amount = calculate_refund(return)
      Payments.refund(order.payment_id, refund_amount)
    end)
    |> Multi.run(:update_inventory, fn _, _ ->
      MyApp.Inventory.restore_items(return.items)
    end)
    |> Multi.run(:email, fn _, %{refund: refund} ->
      MyApp.Emails.RefundEmail.send(order.user, refund)
    end)
    |> Repo.transaction()
  end

  defp calculate_refund(return) do
    return.items
    |> Enum.reduce(Decimal.new("0"), fn item, acc ->
      Decimal.add(acc, Decimal.mult(item.price, Decimal.new(item.quantity)))
    end)
  end
end
```

---

## สรุป Part 59

✅ **Step 641** - Gateway integration  
✅ **Step 642** - Idempotent payments  
✅ **Step 643** - Webhook processing  
✅ **Step 644** - Shopping cart  
✅ **Step 645** - Order state machine  
✅ **Step 646** - Inventory management  
✅ **Step 647** - Discounts & coupons  
✅ **Step 648** - Email receipts  
✅ **Step 649** - Analytics & reporting  
✅ **Step 650** - Return & refund  

➡️ [Part 60: Search Engine Integration](./part-60-search.md)
