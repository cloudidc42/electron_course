# Part 61: Email System (Steps 661-680)

## Step 661: Swoosh Email Library

```elixir
# mix.exs: {:swoosh, "~> 1.14"}, {:gen_smtp, "~> 1.2"}

# config/config.exs
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.SMTP

# config/prod.exs
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.Sendgrid,
  api_key: System.get_env("SENDGRID_API_KEY")

# lib/my_app/mailer.ex
defmodule MyApp.Mailer do
  use Swoosh.Mailer, otp_app: :my_app
end

# Basic email
defmodule MyApp.Emails.Welcome do
  use Swoosh.Email

  def new(user) do
    new()
    |> to({user.name, user.email})
    |> from({"MyApp", "hello@myapp.com"})
    |> reply_to("support@myapp.com")
    |> subject("Welcome to MyApp, #{user.name}!")
    |> html_body(render_html(user))
    |> text_body(render_text(user))
  end

  defp render_html(user) do
    """
    <!DOCTYPE html>
    <html>
    <body>
      <h1>Welcome, #{user.name}!</h1>
      <p>Thanks for joining MyApp.</p>
      <a href="https://myapp.com/dashboard">Get Started</a>
    </body>
    </html>
    """
  end

  defp render_text(user) do
    """
    Welcome, #{user.name}!
    
    Thanks for joining MyApp.
    Get started: https://myapp.com/dashboard
    """
  end
end

# Send email
MyApp.Emails.Welcome.new(user) |> MyApp.Mailer.deliver()
```

---

## Step 662: Email Templates with HEEX

```elixir
defmodule MyApp.Emails do
  use Phoenix.Component

  # Base layout component
  def email_layout(assigns) do
    ~H"""
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title><%= @subject %></title>
      <style>
        body { font-family: Arial, sans-serif; background: #f4f4f4; margin: 0; }
        .container { max-width: 600px; margin: 0 auto; background: white; padding: 40px; }
        .button { background: #4f46e5; color: white; padding: 12px 24px; border-radius: 6px;
                  text-decoration: none; display: inline-block; }
        .footer { color: #999; font-size: 12px; margin-top: 40px; }
      </style>
    </head>
    <body>
      <div class="container">
        <%= render_slot(@inner_block) %>
        <div class="footer">
          <p>You received this email because you signed up for MyApp.</p>
          <p><a href="<%= @unsubscribe_url %>">Unsubscribe</a></p>
        </div>
      </div>
    </body>
    </html>
    """
  end

  def order_confirmation_email(assigns) do
    ~H"""
    <.email_layout subject="Order Confirmed" unsubscribe_url={@unsubscribe_url}>
      <h1>Order Confirmed! 🎉</h1>
      <p>Hi <%= @user.name %>, your order #<%= @order.id %> has been confirmed.</p>
      
      <table style="width: 100%; border-collapse: collapse;">
        <thead>
          <tr style="border-bottom: 1px solid #eee;">
            <th>Item</th><th>Qty</th><th>Price</th>
          </tr>
        </thead>
        <tbody>
          <tr :for={item <- @order.items}>
            <td><%= item.product.name %></td>
            <td><%= item.quantity %></td>
            <td>$<%= item.price %></td>
          </tr>
        </tbody>
        <tfoot>
          <tr>
            <td colspan="2"><strong>Total</strong></td>
            <td><strong>$<%= @order.total %></strong></td>
          </tr>
        </tfoot>
      </table>
      
      <p><a href={@track_url} class="button">Track Order</a></p>
    </.email_layout>
    """
  end

  def render_order_confirmation(user, order) do
    html = Phoenix.LiveView.HTMLEngine.render(
      fn assigns -> order_confirmation_email(assigns) end,
      %{user: user, order: order, unsubscribe_url: unsubscribe_url(user), track_url: track_url(order)},
      nil
    )
    html
  end

  defp unsubscribe_url(user), do: "https://myapp.com/unsubscribe/#{user.id}"
  defp track_url(order),      do: "https://myapp.com/orders/#{order.id}/track"
end
```

---

## Step 663: Email Queue with Oban

```elixir
defmodule MyApp.Workers.EmailWorker do
  use Oban.Worker, queue: :email, max_attempts: 5

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"email_type" => type, "data" => data}}) do
    data = for {k, v} <- data, into: %{}, do: {String.to_atom(k), v}

    email = build_email(type, data)
    
    case MyApp.Mailer.deliver(email) do
      {:ok, _}        -> :ok
      {:error, reason} -> {:error, reason}
    end
  end

  defp build_email("welcome", %{user_id: uid}) do
    user = MyApp.Accounts.get_user!(uid)
    MyApp.Emails.Welcome.new(user)
  end

  defp build_email("order_confirmation", %{order_id: oid}) do
    order = MyApp.Orders.get!(oid) |> MyApp.Repo.preload([:user, items: :product])
    MyApp.Emails.OrderConfirmation.new(order)
  end

  defp build_email("password_reset", %{user_id: uid, token: token}) do
    user = MyApp.Accounts.get_user!(uid)
    MyApp.Emails.PasswordReset.new(user, token)
  end
end

# Helper to enqueue emails
defmodule MyApp.Emails.Dispatcher do
  def send_welcome(user) do
    MyApp.Workers.EmailWorker.new(%{
      email_type: "welcome",
      data: %{user_id: user.id}
    })
    |> Oban.insert()
  end

  def send_order_confirmation(order) do
    MyApp.Workers.EmailWorker.new(%{
      email_type: "order_confirmation",
      data: %{order_id: order.id}
    })
    |> Oban.insert()
  end
end
```

---

## Step 664: Email Tracking

```elixir
defmodule MyApp.EmailTracking do
  # Track opens with 1x1 pixel
  # Track clicks with redirect links

  def track_open_pixel(email_id) do
    "<img src=\"https://myapp.com/email/track/open/#{email_id}\" width=\"1\" height=\"1\" />"
  end

  def tracked_link(url, email_id, link_id) do
    "https://myapp.com/email/track/click/#{email_id}/#{link_id}?url=#{URI.encode(url)}"
  end

  # Controller to handle tracking
  defmodule MyAppWeb.EmailTrackingController do
    use MyAppWeb, :controller

    def open(conn, %{"email_id" => email_id}) do
      MyApp.EmailTracking.record_open(email_id)
      
      # Return 1x1 transparent GIF
      conn
      |> put_resp_content_type("image/gif")
      |> send_resp(200, Base.decode64!("R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7"))
    end

    def click(conn, %{"email_id" => email_id, "link_id" => link_id, "url" => url}) do
      MyApp.EmailTracking.record_click(email_id, link_id)
      redirect(conn, external: url)
    end
  end

  def record_open(email_id) do
    from(e in EmailSend, where: e.id == ^email_id)
    |> MyApp.Repo.update_all(
      set: [opened_at: DateTime.utc_now()],
      inc: [open_count: 1]
    )
  end

  def record_click(email_id, link_id) do
    %EmailClick{}
    |> EmailClick.changeset(%{email_id: email_id, link_id: link_id})
    |> MyApp.Repo.insert(on_conflict: {:replace, [:click_count, :last_clicked_at]},
                         conflict_target: [:email_id, :link_id])
  end
end
```

---

## Step 665: Email Preferences & Unsubscribe

```elixir
defmodule MyApp.EmailPreferences do
  use Ecto.Schema

  schema "email_preferences" do
    belongs_to :user, MyApp.Accounts.User
    
    field :marketing,     :boolean, default: true
    field :transactional, :boolean, default: true
    field :weekly_digest, :boolean, default: true
    field :product_news,  :boolean, default: true
    
    timestamps()
  end

  def get_or_create(user_id) do
    case MyApp.Repo.get_by(__MODULE__, user_id: user_id) do
      nil  ->
        %__MODULE__{}
        |> changeset(%{user_id: user_id})
        |> MyApp.Repo.insert!()
      pref -> pref
    end
  end

  def can_send?(user_id, type) do
    prefs = get_or_create(user_id)
    Map.get(prefs, type, true)
  end

  def unsubscribe_all(user_id) do
    __MODULE__
    |> where(user_id: ^user_id)
    |> MyApp.Repo.update_all(
      set: [marketing: false, weekly_digest: false, product_news: false]
    )
  end

  # Generate unique unsubscribe token
  def generate_token(user_id, email_type) do
    :crypto.mac(:hmac, :sha256, Application.get_env(:my_app, :secret_key), "#{user_id}:#{email_type}")
    |> Base.url_encode64()
  end

  def verify_token(token, user_id, email_type) do
    expected = generate_token(user_id, email_type)
    Plug.Crypto.secure_compare(token, expected)
  end
end
```

---

## Step 666: Bulk Email Campaigns

```elixir
defmodule MyApp.Campaigns do
  use Ecto.Schema

  schema "campaigns" do
    field :name,       :string
    field :subject,    :string
    field :html_body,  :string
    field :text_body,  :string
    field :status,     Ecto.Enum, values: [:draft, :scheduled, :sending, :sent, :cancelled]
    field :scheduled_at, :utc_datetime
    field :sent_count, :integer, default: 0
    field :audience,   :map  # filters for target audience
    timestamps()
  end

  def send_campaign(campaign_id) do
    campaign = MyApp.Repo.get!(__MODULE__, campaign_id)
    
    campaign
    |> changeset(%{status: :sending})
    |> MyApp.Repo.update!()

    users = get_audience(campaign.audience)
    total = length(users)

    # Enqueue in batches to avoid overwhelming the queue
    users
    |> Enum.chunk_every(100)
    |> Enum.with_index()
    |> Enum.each(fn {batch, i} ->
      Enum.each(batch, fn user ->
        MyApp.Workers.CampaignEmailWorker.new(
          %{campaign_id: campaign_id, user_id: user.id},
          scheduled_at: DateTime.add(DateTime.utc_now(), i * 10)  # stagger
        )
        |> Oban.insert()
      end)
    end)

    campaign |> changeset(%{sent_count: total, status: :sent}) |> MyApp.Repo.update!()
  end

  defp get_audience(%{"plan" => plan}) do
    from(u in MyApp.Accounts.User,
      join: t in assoc(u, :tenant),
      where: t.plan == ^plan and u.email_preferences.marketing == true
    )
    |> MyApp.Repo.all()
  end

  defp get_audience(_), do: MyApp.Accounts.list_subscribed_users()
end
```

---

## Step 667: Email Digest

```elixir
defmodule MyApp.Workers.WeeklyDigestWorker do
  use Oban.Worker, queue: :email

  @impl Oban.Worker
  def perform(_job) do
    MyApp.Accounts.list_digest_subscribers()
    |> Enum.each(fn user ->
      digest = build_digest(user)
      
      if Enum.any?(digest.items) do
        MyApp.Emails.WeeklyDigest.new(user, digest)
        |> MyApp.Mailer.deliver()
      end
    end)
    :ok
  end

  defp build_digest(user) do
    since = DateTime.add(DateTime.utc_now(), -7 * 86400)

    %{
      new_products:   MyApp.Catalog.new_products_since(since, limit: 5),
      price_drops:    MyApp.Catalog.price_drops_since(since, user_id: user.id),
      recommendations: MyApp.Recommendations.for_user(user.id, limit: 3),
      items:          nil  # populated after
    }
    |> then(fn d ->
      items = d.new_products ++ d.price_drops ++ d.recommendations
      %{d | items: items}
    end)
  end
end

# Schedule weekly digest - every Monday 8am UTC
# Oban config:
# crontab: [{"0 8 * * 1", MyApp.Workers.WeeklyDigestWorker}]
```

---

## Step 668: Transactional Email Testing

```elixir
# config/test.exs
config :my_app, MyApp.Mailer, adapter: Swoosh.Adapters.Test

defmodule MyApp.EmailTest do
  use ExUnit.Case, async: true
  use Swoosh.Email.Mailbox

  test "welcome email has correct content" do
    user = %{name: "Alice", email: "alice@test.com", id: 1}
    
    email = MyApp.Emails.Welcome.new(user)
    
    assert email.subject == "Welcome to MyApp, Alice!"
    assert email.to == [{"Alice", "alice@test.com"}]
    assert email.html_body =~ "Alice"
    assert email.html_body =~ "dashboard"
    refute email.html_body =~ "<script"
  end

  test "email is delivered" do
    user = %{name: "Bob", email: "bob@test.com", id: 2}
    
    {:ok, _} = MyApp.Emails.Welcome.new(user) |> MyApp.Mailer.deliver()
    
    assert_email_sent(fn email ->
      email.to == [{"Bob", "bob@test.com"}]
    end)
  end

  test "order confirmation includes order details" do
    order = %{
      id: 123,
      user: %{name: "Carol", email: "carol@test.com"},
      items: [%{product: %{name: "Book"}, quantity: 1, price: Decimal.new("29.99")}],
      total: Decimal.new("29.99")
    }
    
    email = MyApp.Emails.OrderConfirmation.new(order)
    
    assert email.subject =~ "123"
    assert email.html_body =~ "Book"
    assert email.html_body =~ "29.99"
  end
end
```

---

## Step 669: Email Bounce Handling

```elixir
defmodule MyApp.EmailBounces do
  # Handle SendGrid/SES bounce webhooks

  def handle_bounce(email, bounce_type) do
    user = MyApp.Accounts.get_by_email(email)
    
    case bounce_type do
      :hard ->
        # Permanent failure - disable email delivery for this address
        mark_email_invalid(email)
        if user, do: notify_user_of_bounce(user)

      :soft ->
        # Temporary failure - track and potentially disable after N soft bounces
        increment_soft_bounce(email)

      :spam_complaint ->
        # User marked as spam - unsubscribe immediately
        if user, do: MyApp.EmailPreferences.unsubscribe_all(user.id)
    end
  end

  defp mark_email_invalid(email) do
    MyApp.Repo.update_all(
      from(u in MyApp.Accounts.User, where: u.email == ^email),
      set: [email_valid: false]
    )

    # Also track in a separate bounce table
    %EmailBounce{}
    |> EmailBounce.changeset(%{email: email, bounce_type: :hard, occurred_at: DateTime.utc_now()})
    |> MyApp.Repo.insert(on_conflict: :replace_all, conflict_target: :email)
  end

  defp increment_soft_bounce(email) do
    {_, [result]} = MyApp.Repo.update_all(
      from(b in EmailBounce, where: b.email == ^email),
      [inc: [soft_count: 1]],
      returning: true
    )
    
    if result && result.soft_count >= 5 do
      mark_email_invalid(email)
    end
  end

  # SendGrid webhook controller
  defmodule MyAppWeb.SendGridWebhookController do
    use MyAppWeb, :controller

    def events(conn, params) when is_list(params) do
      Enum.each(params, fn event ->
        case event["event"] do
          "bounce"          -> MyApp.EmailBounces.handle_bounce(event["email"], :hard)
          "deferred"        -> MyApp.EmailBounces.handle_bounce(event["email"], :soft)
          "spamreport"      -> MyApp.EmailBounces.handle_bounce(event["email"], :spam_complaint)
          "unsubscribe"     -> handle_unsubscribe(event["email"])
          _ -> :ok
        end
      end)
      send_resp(conn, 200, "ok")
    end

    defp handle_unsubscribe(email) do
      case MyApp.Accounts.get_by_email(email) do
        nil  -> :ok
        user -> MyApp.EmailPreferences.unsubscribe_all(user.id)
      end
    end
  end
end
```

---

## Step 670: Email Delivery Monitoring

```elixir
defmodule MyApp.EmailMonitor do
  use GenServer

  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)

  def record_delivery(email_type, status, duration_ms) do
    :telemetry.execute(
      [:my_app, :email, :delivery],
      %{duration: duration_ms},
      %{type: email_type, status: status}
    )
  end

  def init(_) do
    :telemetry.attach_many(
      "email-monitor",
      [
        [:my_app, :email, :delivery]
      ],
      &handle_event/4,
      nil
    )
    {:ok, %{counts: %{}, failures: []}}
  end

  def handle_event([:my_app, :email, :delivery], measurements, %{status: :error} = meta, _) do
    Logger.warning("Email delivery failed: #{meta.type}")
    
    # Alert if failure rate > threshold
    :ets.update_counter(:email_stats, {meta.type, :failures}, {2, 1}, {{meta.type, :failures}, 0})
  end

  def handle_event([:my_app, :email, :delivery], measurements, meta, _) do
    :ets.update_counter(:email_stats, {meta.type, :success}, {2, 1}, {{meta.type, :success}, 0})
  end

  def delivery_stats do
    :ets.tab2list(:email_stats)
    |> Enum.group_by(fn {{type, _}, _} -> type end)
    |> Enum.map(fn {type, entries} ->
      successes = Enum.find_value(entries, 0, fn {{_, :success}, count} -> count; _ -> false end)
      failures  = Enum.find_value(entries, 0, fn {{_, :failures}, count} -> count; _ -> false end)
      total = successes + failures
      
      %{
        type:         type,
        total:        total,
        success_rate: if(total > 0, do: successes / total * 100, else: 0)
      }
    end)
  end
end
```

---

## สรุป Part 61

✅ **Step 661** - Swoosh basics  
✅ **Step 662** - HEEX email templates  
✅ **Step 663** - Email queue with Oban  
✅ **Step 664** - Email tracking (opens/clicks)  
✅ **Step 665** - Preferences & unsubscribe  
✅ **Step 666** - Bulk campaigns  
✅ **Step 667** - Weekly digest  
✅ **Step 668** - Email testing  
✅ **Step 669** - Bounce handling  
✅ **Step 670** - Delivery monitoring  

➡️ [Part 62: File Processing & Storage](./part-62-file-processing.md)
