# Part 88: Email & Notifications (Steps 941-960)

## Step 941: Swoosh Email

```elixir
# mix.exs: {:swoosh, "~> 1.17"}, {:finch, "~> 0.18"}

defmodule MyApp.Mailer do
  use Swoosh.Mailer, otp_app: :my_app
end

defmodule MyApp.Emails do
  import Swoosh.Email

  def welcome(user) do
    new()
    |> to({user.name, user.email})
    |> from({"MyApp", "no-reply@myapp.com"})
    |> reply_to("support@myapp.com")
    |> subject("Welcome to MyApp, #{user.name}!")
    |> html_body(welcome_html(user))
    |> text_body(welcome_text(user))
    |> put_provider_option(:track_opens, true)
    |> put_provider_option(:tags, ["welcome", "transactional"])
  end

  def order_confirmation(order) do
    new()
    |> to(order.customer.email)
    |> from({"MyApp Orders", "orders@myapp.com"})
    |> subject("Order Confirmation ##{order.id}")
    |> html_body(order_html(order))
    |> text_body(order_text(order))
  end

  def password_reset(user, token) do
    reset_url = MyAppWeb.Endpoint.url() <> "/reset-password/#{token}"
    
    new()
    |> to(user.email)
    |> from({"MyApp Security", "security@myapp.com"})
    |> subject("Reset Your Password")
    |> html_body("""
      <p>Click <a href="#{reset_url}">here</a> to reset your password.</p>
      <p>This link expires in 1 hour.</p>
    """)
    |> text_body("Reset: #{reset_url} (expires in 1 hour)")
  end
end

# config/prod.exs
config :my_app, MyApp.Mailer,
  adapter: Swoosh.Adapters.Sendgrid,
  api_key: System.get_env("SENDGRID_API_KEY")
```

---

## Step 942: Email Templates

```elixir
defmodule MyApp.Emails.Templates do
  # Use HEEx templates for emails

  def welcome_html(user) do
    assigns = %{user: user, year: Date.utc_today().year}
    Phoenix.Template.render_to_string(MyApp.Emails.HTML, "welcome", "html", assigns)
  end
end

# lib/my_app/emails/html/welcome.html.heex
defmodule MyApp.Emails.HTML do
  use Phoenix.Component
end
# welcome.html.heex:
# <!DOCTYPE html>
# <html>
# <head>
#   <meta charset="utf-8">
#   <style>
#     body { font-family: sans-serif; max-width: 600px; margin: 0 auto; }
#     .button { background: #4CAF50; color: white; padding: 12px 24px; text-decoration: none; border-radius: 4px; }
#   </style>
# </head>
# <body>
#   <h1>Welcome, <%= @user.name %>!</h1>
#   <p>Thanks for joining MyApp.</p>
#   <a class="button" href="<%= MyAppWeb.Endpoint.url() %>">Get Started</a>
#   <footer><p>&copy; <%= @year %> MyApp</p></footer>
# </body>
# </html>

defmodule MyApp.Mailer.HTML do
  use Phoenix.Component

  def layout(assigns) do
    ~H"""
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8">
      <meta name="viewport" content="width=device-width, initial-scale=1">
    </head>
    <body style="font-family: Arial, sans-serif; background: #f5f5f5; padding: 20px;">
      <div style="max-width: 600px; margin: 0 auto; background: white; padding: 40px; border-radius: 8px;">
        <%= render_slot(@inner_block) %>
      </div>
    </body>
    </html>
    """
  end
end
```

---

## Step 943: Email Queue

```elixir
defmodule MyApp.Workers.EmailWorker do
  use Oban.Worker, queue: :emails, max_attempts: 3

  def perform(%Oban.Job{args: %{"type" => type, "user_id" => user_id} = args}) do
    user = MyApp.Accounts.get_user!(user_id)
    
    email = build_email(type, user, args)
    
    case MyApp.Mailer.deliver(email) do
      {:ok, _}        -> :ok
      {:error, reason} ->
        Logger.error("Email delivery failed: #{inspect(reason)}")
        {:error, reason}
    end
  end

  defp build_email("welcome", user, _args) do
    MyApp.Emails.welcome(user)
  end

  defp build_email("order_confirmation", user, %{"order_id" => order_id}) do
    order = MyApp.Orders.get!(order_id)
    MyApp.Emails.order_confirmation(%{order | customer: user})
  end

  defp build_email("password_reset", user, %{"token" => token}) do
    MyApp.Emails.password_reset(user, token)
  end

  # Convenience function
  def enqueue(type, user_id, extra \\ %{}) do
    args = Map.merge(%{"type" => type, "user_id" => user_id}, extra)
    __MODULE__.new(args) |> Oban.insert!()
  end
end
```

---

## Step 944: Email Deliverability

```elixir
defmodule MyApp.Email.Deliverability do
  # Manage bounces, unsubscribes, and spam complaints

  def handle_bounce(email_address, bounce_type) do
    case bounce_type do
      :hard ->
        # Permanent failure: mark as invalid
        MyApp.Accounts.mark_email_invalid(email_address)
      :soft ->
        # Temporary failure: track and possibly suppress
        MyApp.EmailBounces.record(email_address, :soft)
        
        count = MyApp.EmailBounces.count(email_address, :soft, days: 7)
        if count >= 3 do
          MyApp.Accounts.temporarily_suppress_email(email_address, days: 30)
        end
    end
  end

  def handle_spam_complaint(email_address) do
    MyApp.Accounts.unsubscribe(email_address, :all)
    Logger.warning("Spam complaint from: #{email_address}")
  end

  def can_send?(email_address) do
    user = MyApp.Accounts.get_by_email(email_address)
    
    cond do
      is_nil(user)             -> false
      user.email_invalid       -> false
      user.unsubscribed        -> false
      user.email_suppressed_until != nil and
        DateTime.compare(user.email_suppressed_until, DateTime.utc_now()) == :gt -> false
      true -> true
    end
  end
end
```

---

## Step 945: SMS Notifications

```elixir
defmodule MyApp.SMS do
  # Twilio SMS integration

  @base_url "https://api.twilio.com/2010-04-01"

  def send(to, message) do
    account_sid = System.get_env("TWILIO_ACCOUNT_SID")
    auth_token  = System.get_env("TWILIO_AUTH_TOKEN")
    from        = System.get_env("TWILIO_PHONE_NUMBER")

    case Req.post(
      "#{@base_url}/Accounts/#{account_sid}/Messages.json",
      auth:  {account_sid, auth_token},
      form: [To: to, From: from, Body: message]
    ) do
      {:ok, %{status: 201, body: body}} ->
        {:ok, %{sid: body["sid"], status: body["status"]}}
      {:ok, %{status: _, body: body}} ->
        {:error, body["message"]}
      {:error, reason} ->
        {:error, reason}
    end
  end

  def send_verification(phone_number) do
    code = :rand.uniform(899_999) + 100_000
    send(phone_number, "Your MyApp verification code is: #{code}")
    
    # Store code with TTL
    Redix.command(:redix, ["SETEX", "verify:#{phone_number}", 300, code])
    :ok
  end

  def verify(phone_number, code) do
    case Redix.command(:redix, ["GET", "verify:#{phone_number}"]) do
      {:ok, stored_code} when stored_code == to_string(code) ->
        Redix.command(:redix, ["DEL", "verify:#{phone_number}"])
        :ok
      _ ->
        {:error, :invalid_code}
    end
  end
end
```

---

## Step 946: Push Notifications

```elixir
defmodule MyApp.PushNotifications do
  # Firebase Cloud Messaging

  @fcm_url "https://fcm.googleapis.com/v1/projects"

  def send(device_token, title, body, data \\ %{}) do
    project_id   = System.get_env("FIREBASE_PROJECT_ID")
    access_token = get_access_token()
    
    message = %{
      message: %{
        token:        device_token,
        notification: %{title: title, body: body},
        data:         Map.new(data, fn {k, v} -> {to_string(k), to_string(v)} end),
        android: %{
          notification: %{channel_id: "default", priority: "high"}
        },
        apns: %{
          payload: %{aps: %{sound: "default", badge: 1}}
        }
      }
    }

    Req.post("#{@fcm_url}/#{project_id}/messages:send",
      json:    message,
      headers: [{"authorization", "Bearer #{access_token}"}]
    )
    |> case do
      {:ok, %{status: 200, body: body}} -> {:ok, body["name"]}
      {:ok, %{body: body}}              -> {:error, body}
      {:error, reason}                  -> {:error, reason}
    end
  end

  def send_to_user(user_id, notification) do
    tokens = MyApp.DeviceTokens.list(user_id)
    
    Enum.each(tokens, fn token ->
      case send(token.token, notification.title, notification.body) do
        {:ok, _}    -> :ok
        {:error, %{"error" => %{"code" => 404}}} ->
          # Token expired, remove it
          MyApp.DeviceTokens.delete(token)
        {:error, reason} ->
          Logger.warning("Push notification failed: #{inspect(reason)}")
      end
    end)
  end

  defp get_access_token do
    # Use Goth for Google OAuth2 access token
    {:ok, token} = Goth.Token.for_scope("https://www.googleapis.com/auth/firebase.messaging")
    token.token
  end
end
```

---

## Step 947: In-App Notifications System

```elixir
defmodule MyApp.InAppNotifications do
  use Ecto.Schema
  import Ecto.Query

  schema "notifications" do
    field :user_id,      :integer
    field :type,         :string
    field :title,        :string
    field :body,         :string
    field :data,         :map, default: %{}
    field :read,         :boolean, default: false
    field :read_at,      :utc_datetime
    field :action_url,   :string
    field :icon,         :string

    timestamps()
  end

  def create(user_id, attrs) do
    %__MODULE__{user_id: user_id}
    |> cast(attrs, [:type, :title, :body, :data, :action_url, :icon])
    |> validate_required([:type, :title])
    |> MyApp.Repo.insert()
    |> tap(fn
      {:ok, notif} ->
        # Real-time delivery
        Phoenix.PubSub.broadcast(MyApp.PubSub, "user:#{user_id}:notifs",
          {:new_notification, format(notif)})
      _ -> :ok
    end)
  end

  def list_unread(user_id, limit \\ 20) do
    from n in __MODULE__,
      where: n.user_id == ^user_id and n.read == false,
      order_by: [desc: n.inserted_at],
      limit: ^limit
    |> MyApp.Repo.all()
  end

  def mark_read(user_id, notification_id) do
    from(n in __MODULE__, where: n.id == ^notification_id and n.user_id == ^user_id)
    |> MyApp.Repo.update_all(set: [read: true, read_at: DateTime.utc_now()])
  end

  def mark_all_read(user_id) do
    from(n in __MODULE__, where: n.user_id == ^user_id and n.read == false)
    |> MyApp.Repo.update_all(set: [read: true, read_at: DateTime.utc_now()])
  end

  def unread_count(user_id) do
    from(n in __MODULE__, where: n.user_id == ^user_id and n.read == false, select: count())
    |> MyApp.Repo.one()
  end

  defp format(notif) do
    %{
      id:         notif.id,
      type:       notif.type,
      title:      notif.title,
      body:       notif.body,
      action_url: notif.action_url,
      icon:       notif.icon,
      created_at: notif.inserted_at
    }
  end
end
```

---

## Step 948: Notification Preferences

```elixir
defmodule MyApp.NotificationPreferences do
  use Ecto.Schema

  schema "notification_preferences" do
    field :user_id,         :integer
    field :email_enabled,   :boolean, default: true
    field :sms_enabled,     :boolean, default: false
    field :push_enabled,    :boolean, default: true
    field :in_app_enabled,  :boolean, default: true
    field :quiet_hours_start, :time
    field :quiet_hours_end,   :time
    field :categories,      :map, default: %{}  # per-category overrides

    timestamps()
  end

  def can_notify?(user_id, channel, category \\ nil) do
    prefs = MyApp.Repo.get_by(__MODULE__, user_id: user_id)
    || %__MODULE__{}

    # Check channel
    channel_enabled = case channel do
      :email  -> prefs.email_enabled
      :sms    -> prefs.sms_enabled
      :push   -> prefs.push_enabled
      :in_app -> prefs.in_app_enabled
    end

    # Check category override
    category_enabled = if category do
      get_in(prefs.categories, [to_string(category), to_string(channel)])
      |> Kernel.||(channel_enabled)
    else
      channel_enabled
    end

    # Check quiet hours
    quiet = in_quiet_hours?(prefs)

    category_enabled and (channel != :push or not quiet)
  end

  defp in_quiet_hours?(%{quiet_hours_start: nil}), do: false
  defp in_quiet_hours?(%{quiet_hours_start: start, quiet_hours_end: stop}) do
    now = Time.utc_now()
    if Time.compare(start, stop) == :lt do
      Time.compare(now, start) != :lt and Time.compare(now, stop) == :lt
    else
      Time.compare(now, start) != :lt or Time.compare(now, stop) == :lt
    end
  end
end
```

---

## Step 949: Digest Emails

```elixir
defmodule MyApp.Workers.DigestEmailWorker do
  use Oban.Worker, queue: :emails

  def perform(%Oban.Job{args: %{"type" => "weekly_digest"}}) do
    users = MyApp.Accounts.users_with_weekly_digest()

    Enum.each(users, fn user ->
      digest_data = gather_digest_data(user)
      
      if has_content?(digest_data) do
        email = MyApp.Emails.weekly_digest(user, digest_data)
        MyApp.Mailer.deliver(email)
      end
    end)

    :ok
  end

  defp gather_digest_data(user) do
    %{
      new_posts:     MyApp.Posts.new_since(user.last_digest_at, 10),
      new_comments:  MyApp.Comments.on_user_posts(user.id, since: user.last_digest_at),
      activity:      MyApp.Activity.summary(user.id, since: user.last_digest_at),
      stats:         MyApp.Stats.user_weekly(user.id)
    }
  end

  defp has_content?(%{new_posts: [], new_comments: [], activity: []}), do: false
  defp has_content?(_), do: true
end

# Schedule weekly digest via Oban Cron
# config/config.exs
config :my_app, Oban,
  cron: [
    {"0 9 * * 1", MyApp.Workers.DigestEmailWorker,
     args: %{type: "weekly_digest"}, timezone: "UTC"}
  ]
```

---

## Step 950: Email Analytics

```elixir
defmodule MyApp.EmailAnalytics do
  # Track email opens, clicks via SendGrid webhooks

  def handle_webhook(events) when is_list(events) do
    Enum.each(events, &process_event/1)
  end

  defp process_event(%{"event" => "open", "email" => email, "timestamp" => ts, "sg_message_id" => msg_id}) do
    MyApp.Repo.insert!(%EmailEvent{
      event_type: :open,
      email:      email,
      message_id: msg_id,
      occurred_at: DateTime.from_unix!(ts)
    })
    
    :telemetry.execute([:email, :opened], %{count: 1}, %{email: email})
  end

  defp process_event(%{"event" => "click", "email" => email, "url" => url, "timestamp" => ts, "sg_message_id" => msg_id}) do
    MyApp.Repo.insert!(%EmailEvent{
      event_type: :click,
      email:      email,
      message_id: msg_id,
      url:        url,
      occurred_at: DateTime.from_unix!(ts)
    })
  end

  defp process_event(%{"event" => "bounce", "email" => email, "type" => type}) do
    bounce_type = if type == "bounce", do: :hard, else: :soft
    MyApp.Email.Deliverability.handle_bounce(email, bounce_type)
  end

  defp process_event(%{"event" => "spamreport", "email" => email}) do
    MyApp.Email.Deliverability.handle_spam_complaint(email)
  end

  defp process_event(_event), do: :ok

  def campaign_stats(campaign_id) do
    import Ecto.Query
    
    from(e in EmailEvent,
      where: e.campaign_id == ^campaign_id,
      group_by: e.event_type,
      select: {e.event_type, count(e.id)}
    )
    |> MyApp.Repo.all()
    |> Map.new()
  end
end
```

---

## สรุป Part 88

✅ **Step 941** - Swoosh email  
✅ **Step 942** - Email templates  
✅ **Step 943** - Email queue  
✅ **Step 944** - Email deliverability  
✅ **Step 945** - SMS notifications  
✅ **Step 946** - Push notifications  
✅ **Step 947** - In-app notifications  
✅ **Step 948** - Notification preferences  
✅ **Step 949** - Digest emails  
✅ **Step 950** - Email analytics  

➡️ [Part 89: Authentication & Authorization](./part-89-auth-advanced.md)
