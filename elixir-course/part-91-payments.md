# Part 91: Payment Processing (Steps 971-980)

## Step 971: Stripe Integration

```elixir
# mix.exs: {:stripity_stripe, "~> 3.0"}

defmodule MyApp.Payments.Stripe do
  def create_payment_intent(amount_cents, currency, metadata \\ %{}) do
    Stripe.PaymentIntent.create(%{
      amount:   amount_cents,
      currency: currency,
      metadata: metadata,
      automatic_payment_methods: %{enabled: true}
    })
    |> case do
      {:ok, pi}     -> {:ok, %{client_secret: pi.client_secret, id: pi.id}}
      {:error, err} -> {:error, err.message}
    end
  end

  def confirm_payment(payment_intent_id) do
    Stripe.PaymentIntent.retrieve(payment_intent_id)
    |> case do
      {:ok, %{status: "succeeded"} = pi} ->
        {:ok, %{id: pi.id, amount: pi.amount, currency: pi.currency}}
      {:ok, %{status: status}} ->
        {:error, "payment status: #{status}"}
      {:error, err} ->
        {:error, err.message}
    end
  end

  def create_customer(user) do
    Stripe.Customer.create(%{
      email: user.email,
      name:  user.name,
      metadata: %{user_id: user.id}
    })
    |> case do
      {:ok, customer} ->
        MyApp.Accounts.update_user(user, %{stripe_customer_id: customer.id})
        {:ok, customer.id}
      {:error, err} ->
        {:error, err.message}
    end
  end

  def save_card(customer_id, payment_method_id) do
    Stripe.PaymentMethod.attach(%{
      customer:       customer_id,
      payment_method: payment_method_id
    })
  end

  def charge_saved_card(customer_id, pm_id, amount, currency) do
    Stripe.PaymentIntent.create(%{
      amount:         amount,
      currency:       currency,
      customer:       customer_id,
      payment_method: pm_id,
      confirm:        true,
      off_session:    true
    })
  end
end
```

---

## Step 972: Stripe Webhooks

```elixir
defmodule MyAppWeb.StripeWebhookController do
  use MyAppWeb, :controller

  @signing_secret System.get_env("STRIPE_WEBHOOK_SECRET")

  def handle(conn, _params) do
    with {:ok, payload, conn} <- read_body(conn),
         [signature]          <- get_req_header(conn, "stripe-signature"),
         {:ok, event}         <- Stripe.Webhook.construct_event(payload, signature, @signing_secret) do
      
      Task.start(fn -> process_event(event) end)
      send_resp(conn, 200, "ok")
    else
      {:error, %Stripe.SignatureVerificationError{}} ->
        conn |> put_status(400) |> json(%{error: "invalid_signature"}) |> halt()
    end
  end

  defp process_event(%{type: "payment_intent.succeeded", data: %{object: pi}}) do
    order_id = pi.metadata["order_id"]
    
    with {:ok, order} <- MyApp.Orders.get(order_id),
         {:ok, _}     <- MyApp.Orders.mark_paid(order, pi.id) do
      MyApp.Emails.order_confirmation(order) |> MyApp.Mailer.deliver_later()
      MyApp.Analytics.track("order_completed", %{order_id: order_id, amount: pi.amount})
    end
  end

  defp process_event(%{type: "payment_intent.payment_failed", data: %{object: pi}}) do
    order_id = pi.metadata["order_id"]
    reason   = pi.last_payment_error && pi.last_payment_error.message

    MyApp.Orders.mark_payment_failed(order_id, reason)
  end

  defp process_event(%{type: "customer.subscription.deleted", data: %{object: sub}}) do
    MyApp.Subscriptions.cancel(sub.id)
  end

  defp process_event(%{type: "invoice.payment_failed", data: %{object: inv}}) do
    MyApp.Subscriptions.handle_payment_failure(inv.subscription, inv.attempt_count)
  end

  defp process_event(_event), do: :ok
end
```

---

## Step 973: Subscription Management

```elixir
defmodule MyApp.Subscriptions do
  use Ecto.Schema
  import Ecto.Query

  schema "subscriptions" do
    field :user_id,              :integer
    field :stripe_subscription_id, :string
    field :plan,                 :string
    field :status,               :string
    field :current_period_start, :utc_datetime
    field :current_period_end,   :utc_datetime
    field :cancel_at_period_end, :boolean, default: false

    timestamps()
  end

  def create(user, plan) do
    stripe_price_id = price_id_for_plan(plan)
    
    with {:ok, customer_id} <- ensure_customer(user),
         {:ok, sub}         <- Stripe.Subscription.create(%{
           customer: customer_id,
           items:    [%{price: stripe_price_id}],
           metadata: %{user_id: user.id}
         }) do
      %__MODULE__{
        user_id:               user.id,
        stripe_subscription_id: sub.id,
        plan:                  plan,
        status:                sub.status,
        current_period_start:  DateTime.from_unix!(sub.current_period_start),
        current_period_end:    DateTime.from_unix!(sub.current_period_end)
      }
      |> MyApp.Repo.insert()
    end
  end

  def cancel(stripe_sub_id, opts \\ []) do
    immediate = Keyword.get(opts, :immediate, false)
    
    if immediate do
      Stripe.Subscription.cancel(stripe_sub_id)
      from(s in __MODULE__, where: s.stripe_subscription_id == ^stripe_sub_id)
      |> MyApp.Repo.update_all(set: [status: "canceled"])
    else
      Stripe.Subscription.update(stripe_sub_id, %{cancel_at_period_end: true})
      from(s in __MODULE__, where: s.stripe_subscription_id == ^stripe_sub_id)
      |> MyApp.Repo.update_all(set: [cancel_at_period_end: true])
    end
  end

  def active?(user_id) do
    from(s in __MODULE__,
      where: s.user_id == ^user_id and s.status == "active",
      where: s.current_period_end > ^DateTime.utc_now()
    )
    |> MyApp.Repo.exists?()
  end

  defp price_id_for_plan("basic"),  do: System.get_env("STRIPE_BASIC_PRICE_ID")
  defp price_id_for_plan("pro"),    do: System.get_env("STRIPE_PRO_PRICE_ID")

  defp ensure_customer(user) do
    if user.stripe_customer_id do
      {:ok, user.stripe_customer_id}
    else
      MyApp.Payments.Stripe.create_customer(user)
    end
  end
end
```

---

## Step 974: Refunds & Disputes

```elixir
defmodule MyApp.Payments.Refunds do
  def issue_refund(order_id, amount_cents \\ nil, reason \\ "requested_by_customer") do
    order = MyApp.Repo.get!(MyApp.Order, order_id)
    
    params = %{
      payment_intent: order.stripe_payment_intent_id,
      reason:         reason
    }
    params = if amount_cents, do: Map.put(params, :amount, amount_cents), else: params
    
    case Stripe.Refund.create(params) do
      {:ok, refund} ->
        MyApp.Repo.insert!(%MyApp.Refund{
          order_id:         order_id,
          stripe_refund_id: refund.id,
          amount:           refund.amount,
          reason:           reason,
          status:           refund.status
        })
        
        MyApp.Orders.update_status(order, :refunded)
        {:ok, refund}
      {:error, err} ->
        {:error, err.message}
    end
  end

  def handle_dispute(dispute) do
    order = MyApp.Repo.get_by(MyApp.Order, stripe_payment_intent_id: dispute.payment_intent)
    
    # Automatically gather evidence
    evidence = %{
      customer_name:  order.customer.name,
      customer_email: order.customer.email,
      shipping_documentation: get_shipping_proof(order),
      product_description:    build_product_description(order)
    }
    
    Stripe.Dispute.update(dispute.id, %{evidence: evidence})
    
    Logger.warning("Dispute #{dispute.id} for order #{order.id}: #{dispute.reason}")
  end

  defp get_shipping_proof(order) do
    if order.tracking_number do
      "Order shipped via #{order.carrier} tracking: #{order.tracking_number}"
    end
  end

  defp build_product_description(order) do
    order.items
    |> Enum.map(&"#{&1.name} x#{&1.quantity}")
    |> Enum.join(", ")
  end
end
```

---

## Step 975: Invoice Generation

```elixir
defmodule MyApp.Invoices do
  use Ecto.Schema

  schema "invoices" do
    field :number,       :string
    field :user_id,      :integer
    field :order_id,     :integer
    field :status,       :string, default: "draft"
    field :subtotal,     :integer
    field :tax,          :integer
    field :total,        :integer
    field :currency,     :string, default: "usd"
    field :due_date,     :date
    field :pdf_url,      :string

    timestamps()
  end

  def generate_pdf(invoice) do
    html = render_html(invoice)
    
    case System.cmd("wkhtmltopdf", ["-", "/tmp/invoice_#{invoice.id}.pdf"],
           input: html) do
      {_, 0} ->
        pdf_path = "/tmp/invoice_#{invoice.id}.pdf"
        {:ok, url} = MyApp.Storage.upload(%Plug.Upload{
          path:         pdf_path,
          filename:     "invoice_#{invoice.number}.pdf",
          content_type: "application/pdf"
        })
        
        MyApp.Repo.update_all(
          from(i in __MODULE__, where: i.id == ^invoice.id),
          set: [pdf_url: url, status: "issued"]
        )
        {:ok, url}
      {error, _} ->
        {:error, error}
    end
  end

  def next_number do
    year  = Date.utc_today().year
    count = from(i in __MODULE__,
              where: fragment("EXTRACT(YEAR FROM inserted_at) = ?", ^year),
              select: count(i.id)
            ) |> MyApp.Repo.one()
    "INV-#{year}-#{String.pad_leading(to_string(count + 1), 5, "0")}"
  end

  defp render_html(invoice) do
    # Render invoice HTML (simplified)
    """
    <!DOCTYPE html>
    <html>
    <body>
      <h1>Invoice #{invoice.number}</h1>
      <p>Total: #{invoice.currency |> String.upcase} #{invoice.total / 100}</p>
    </body>
    </html>
    """
  end
end
```

---

## Step 976: Multi-Currency Support

```elixir
defmodule MyApp.Currency do
  # Exchange rates (cache for 1 hour)
  def exchange_rate(from, to) do
    cache_key = "fx:#{from}:#{to}"
    
    case Redix.command(:redix, ["GET", cache_key]) do
      {:ok, nil} ->
        rate = fetch_rate(from, to)
        Redix.command(:redix, ["SETEX", cache_key, 3600, to_string(rate)])
        rate
      {:ok, rate} ->
        String.to_float(rate)
    end
  end

  def convert(amount, from, to) when from == to, do: amount
  def convert(amount, from, to) do
    rate = exchange_rate(from, to)
    round(amount * rate)
  end

  def format(amount_cents, currency) do
    amount = amount_cents / 100
    symbol = currency_symbol(currency)
    "#{symbol}#{:erlang.float_to_binary(amount, decimals: 2)}"
  end

  defp fetch_rate(from, to) do
    api_key = System.get_env("EXCHANGE_RATE_API_KEY")
    case Req.get("https://v6.exchangerate-api.com/v6/#{api_key}/pair/#{from}/#{to}") do
      {:ok, %{status: 200, body: %{"conversion_rate" => rate}}} -> rate
      _ -> raise "Cannot fetch exchange rate"
    end
  end

  defp currency_symbol("usd"), do: "$"
  defp currency_symbol("eur"), do: "€"
  defp currency_symbol("gbp"), do: "£"
  defp currency_symbol("thb"), do: "฿"
  defp currency_symbol(code),  do: "#{String.upcase(code)} "
end
```

---

## Step 977: Tax Calculation

```elixir
defmodule MyApp.Tax do
  # US Sales Tax with TaxJar API

  def calculate(order) do
    params = %{
      from_country: "US",
      from_zip:     System.get_env("BUSINESS_ZIP"),
      to_country:   order.shipping_address.country,
      to_zip:       order.shipping_address.zip,
      to_state:     order.shipping_address.state,
      amount:       order.subtotal / 100,
      shipping:     order.shipping_cost / 100,
      line_items:   format_line_items(order.items)
    }

    case Req.post("https://api.taxjar.com/v2/taxes",
           json: params,
           headers: [{"authorization", "Token #{System.get_env("TAXJAR_API_KEY")}"}]) do
      {:ok, %{status: 200, body: %{"tax" => tax}}} ->
        {:ok, round(tax["amount_to_collect"] * 100)}
      {:error, reason} ->
        {:error, reason}
    end
  end

  defp format_line_items(items) do
    Enum.map(items, fn item ->
      %{
        id:          item.id,
        quantity:    item.quantity,
        unit_price:  item.price / 100,
        product_tax_code: item.product.tax_code || "general"
      }
    end)
  end

  # VAT for European customers
  def calculate_vat(country, amount_cents) do
    rate = vat_rate(country)
    vat  = round(amount_cents * rate / 100)
    {vat, amount_cents + vat}
  end

  defp vat_rate("DE"), do: 19
  defp vat_rate("FR"), do: 20
  defp vat_rate("GB"), do: 20
  defp vat_rate("NL"), do: 21
  defp vat_rate(_),    do: 0
end
```

---

## Step 978: Fraud Detection

```elixir
defmodule MyApp.Payments.FraudDetection do
  @high_risk_score 80

  def analyze(order) do
    score = calculate_risk_score(order)
    
    result = %{
      score:    score,
      level:    risk_level(score),
      signals:  collect_signals(order)
    }

    if score >= @high_risk_score do
      Logger.warning("High-risk order #{order.id}: score=#{score}")
      {:review_required, result}
    else
      {:ok, result}
    end
  end

  defp calculate_risk_score(order) do
    signals = collect_signals(order)
    
    Enum.reduce(signals, 0, fn {_name, weight}, acc ->
      acc + weight
    end)
    |> min(100)
  end

  defp collect_signals(order) do
    signals = []
    
    signals = if new_account?(order.user), do: [{:new_account, 20} | signals], else: signals
    signals = if high_value_order?(order),  do: [{:high_value, 15} | signals], else: signals
    signals = if billing_shipping_mismatch?(order), do: [{:address_mismatch, 25} | signals], else: signals
    signals = if multiple_cards_used?(order.user), do: [{:multiple_cards, 20} | signals], else: signals
    signals = if vpn_detected?(order.ip_address),  do: [{:vpn_detected, 20} | signals], else: signals
    signals = if velocity_exceeded?(order.user),   do: [{:high_velocity, 30} | signals], else: signals
    
    signals
  end

  defp new_account?(user) do
    DateTime.diff(DateTime.utc_now(), user.inserted_at, :day) < 7
  end

  defp high_value_order?(order), do: order.total > 50_000  # $500

  defp billing_shipping_mismatch?(order) do
    order.billing_zip != order.shipping_zip
  end

  defp multiple_cards_used?(user) do
    distinct_cards = from(o in MyApp.Order,
      where: o.user_id == ^user.id,
      select: count(fragment("DISTINCT stripe_payment_method_id"))
    ) |> MyApp.Repo.one()
    distinct_cards > 3
  end

  defp vpn_detected?(_ip), do: false  # Would call IP intelligence API

  defp velocity_exceeded?(user) do
    recent = from(o in MyApp.Order,
      where: o.user_id == ^user.id and o.inserted_at > ^DateTime.add(DateTime.utc_now(), -3600),
      select: count(o.id)
    ) |> MyApp.Repo.one()
    recent > 5
  end

  defp risk_level(score) when score < 30, do: :low
  defp risk_level(score) when score < 60, do: :medium
  defp risk_level(_score),                do: :high
end
```

---

## Step 979: Payment Methods

```elixir
defmodule MyApp.PaymentMethods do
  use Ecto.Schema
  import Ecto.Query

  schema "payment_methods" do
    field :user_id,        :integer
    field :type,           :string
    field :stripe_pm_id,   :string
    field :brand,          :string
    field :last4,          :string
    field :exp_month,      :integer
    field :exp_year,       :integer
    field :is_default,     :boolean, default: false

    timestamps()
  end

  def add_card(user, payment_method_id) do
    with {:ok, pm} <- Stripe.PaymentMethod.retrieve(payment_method_id),
         :ok       <- attach_to_customer(user, pm) do
      
      # If first card, set as default
      is_first = not from(p in __MODULE__, where: p.user_id == ^user.id) |> MyApp.Repo.exists?()
      
      %__MODULE__{
        user_id:      user.id,
        type:         pm.type,
        stripe_pm_id: pm.id,
        brand:        pm.card.brand,
        last4:        pm.card.last4,
        exp_month:    pm.card.exp_month,
        exp_year:     pm.card.exp_year,
        is_default:   is_first
      }
      |> MyApp.Repo.insert()
    end
  end

  def set_default(user_id, pm_id) do
    MyApp.Repo.transaction(fn ->
      from(p in __MODULE__, where: p.user_id == ^user_id)
      |> MyApp.Repo.update_all(set: [is_default: false])
      
      from(p in __MODULE__, where: p.id == ^pm_id and p.user_id == ^user_id)
      |> MyApp.Repo.update_all(set: [is_default: true])
    end)
  end

  defp attach_to_customer(user, pm) do
    customer_id = user.stripe_customer_id
    Stripe.PaymentMethod.attach(%{customer: customer_id, payment_method: pm.id})
    :ok
  end
end
```

---

## Step 980: Payment Analytics

```elixir
defmodule MyApp.Payments.Analytics do
  import Ecto.Query

  def revenue_summary(period \\ :month) do
    cutoff = period_cutoff(period)
    
    from(o in MyApp.Order,
      where: o.status == "paid" and o.paid_at > ^cutoff,
      select: %{
        total_revenue: sum(o.total),
        order_count:   count(o.id),
        avg_order:     fragment("ROUND(AVG(?)::numeric, 2)", o.total),
        refund_total:  coalesce(
          fragment("(SELECT SUM(amount) FROM refunds WHERE order_id = ANY(array_agg(?)))", o.id),
          0
        )
      }
    )
    |> MyApp.Repo.one()
  end

  def mrr do
    # Monthly Recurring Revenue from active subscriptions
    from(s in MyApp.Subscription,
      join: p in assoc(s, :plan),
      where: s.status == "active",
      select: sum(p.monthly_price)
    )
    |> MyApp.Repo.one() || 0
  end

  def churn_rate(period \\ :month) do
    cutoff = period_cutoff(period)
    
    canceled = from(s in MyApp.Subscription,
      where: s.canceled_at > ^cutoff,
      select: count(s.id)
    ) |> MyApp.Repo.one()
    
    total = from(s in MyApp.Subscription,
      where: s.inserted_at < ^cutoff,
      select: count(s.id)
    ) |> MyApp.Repo.one()
    
    if total > 0, do: canceled / total * 100, else: 0.0
  end

  defp period_cutoff(:day),   do: DateTime.add(DateTime.utc_now(), -86_400)
  defp period_cutoff(:week),  do: DateTime.add(DateTime.utc_now(), -7 * 86_400)
  defp period_cutoff(:month), do: DateTime.add(DateTime.utc_now(), -30 * 86_400)
  defp period_cutoff(:year),  do: DateTime.add(DateTime.utc_now(), -365 * 86_400)
end
```

---

## สรุป Part 91

✅ **Step 971** - Stripe integration  
✅ **Step 972** - Stripe webhooks  
✅ **Step 973** - Subscription management  
✅ **Step 974** - Refunds & disputes  
✅ **Step 975** - Invoice generation  
✅ **Step 976** - Multi-currency support  
✅ **Step 977** - Tax calculation  
✅ **Step 978** - Fraud detection  
✅ **Step 979** - Payment methods  
✅ **Step 980** - Payment analytics  

➡️ [Part 92: GraphQL API](./part-92-graphql.md)
