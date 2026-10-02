# Part 84: Real-Time Features (Steps 901-920)

## Step 901: Phoenix Channels Advanced

```elixir
defmodule MyAppWeb.GameChannel do
  use MyAppWeb, :channel

  @max_players 8

  def join("game:" <> game_id, %{"token" => token}, socket) do
    with {:ok, user} <- MyApp.Auth.verify_token(token),
         {:ok, game}  <- MyApp.Games.get(game_id),
         :ok          <- check_capacity(game_id) do
      
      send(self(), :after_join)
      
      {:ok,
       %{player_id: user.id, game_state: game},
       assign(socket, :user, user) |> assign(:game_id, game_id)}
    else
      {:error, :token_expired}  -> {:error, %{reason: "token_expired"}}
      {:error, :not_found}      -> {:error, %{reason: "game_not_found"}}
      {:error, :room_full}      -> {:error, %{reason: "room_full"}}
    end
  end

  def handle_info(:after_join, socket) do
    game_id = socket.assigns.game_id
    user    = socket.assigns.user
    
    MyApp.Presence.track(socket, user.id, %{
      online_at: DateTime.utc_now(),
      username:  user.name
    })
    
    push(socket, "presence_state", MyApp.Presence.list(socket))
    broadcast_from!(socket, "player_joined", %{user_id: user.id, name: user.name})
    
    {:noreply, socket}
  end

  def handle_in("move", %{"x" => x, "y" => y}, socket) do
    user_id = socket.assigns.user.id
    game_id = socket.assigns.game_id
    
    case MyApp.Games.make_move(game_id, user_id, %{x: x, y: y}) do
      {:ok, new_state} ->
        broadcast!(socket, "game_updated", %{state: new_state})
        {:reply, {:ok, %{state: new_state}}, socket}
      {:error, :invalid_move} ->
        {:reply, {:error, %{reason: "invalid_move"}}, socket}
    end
  end

  defp check_capacity(game_id) do
    count = MyApp.Presence.list("game:#{game_id}") |> map_size()
    if count < @max_players, do: :ok, else: {:error, :room_full}
  end
end
```

---

## Step 902: Presence Tracking

```elixir
defmodule MyApp.Presence do
  use Phoenix.Presence,
    otp_app: :my_app,
    pubsub_server: MyApp.PubSub

  def track_user(socket, user) do
    track(socket, user.id, %{
      username:    user.name,
      avatar_url:  user.avatar_url,
      status:      :online,
      joined_at:   DateTime.utc_now(),
      metadata:    %{}
    })
  end

  def online_users(topic) do
    topic
    |> list()
    |> Enum.map(fn {user_id, %{metas: [meta | _]}} ->
      %{user_id: String.to_integer(user_id), meta: meta}
    end)
  end

  def user_count(topic) do
    list(topic) |> map_size()
  end
end

# LiveView with presence
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view

  def mount(%{"room_id" => room_id}, %{"user_id" => user_id}, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "chat:#{room_id}")
      {:ok, _} = MyApp.Presence.track(self(), "chat:#{room_id}", user_id, %{
        online_at: DateTime.utc_now()
      })
    end

    {:ok, assign(socket,
      room_id:  room_id,
      messages: MyApp.Chat.recent_messages(room_id),
      users:    MyApp.Presence.online_users("chat:#{room_id}")
    )}
  end

  def handle_info(%Phoenix.Socket.Broadcast{event: "presence_diff", payload: diff}, socket) do
    users = MyApp.Presence.online_users("chat:#{socket.assigns.room_id}")
    {:noreply, assign(socket, :users, users)}
  end
end
```

---

## Step 903: PubSub Patterns

```elixir
defmodule MyApp.PubSubPatterns do
  # Topic namespacing
  def topic(:user, user_id),       do: "user:#{user_id}"
  def topic(:order, order_id),     do: "order:#{order_id}"
  def topic(:room, room_id),       do: "room:#{room_id}"
  def topic(:global, event_type),  do: "global:#{event_type}"

  # Broadcast helpers
  def broadcast_to_user(user_id, event, payload) do
    Phoenix.PubSub.broadcast(MyApp.PubSub, topic(:user, user_id), {event, payload})
  end

  def broadcast_global(event_type, payload) do
    Phoenix.PubSub.broadcast(MyApp.PubSub, topic(:global, event_type), {event_type, payload})
  end

  # Fan-out to multiple topics
  def broadcast_to_users(user_ids, event, payload) do
    Enum.each(user_ids, fn id ->
      broadcast_to_user(id, event, payload)
    end)
  end

  # Distributed PubSub
  def broadcast_cluster(topic, message) do
    Phoenix.PubSub.broadcast(MyApp.PubSub, topic, message)
    # Phoenix.PubSub handles cluster distribution automatically
    # via Phoenix.PubSub.PG2 or Phoenix.PubSub.Redis adapter
  end
end

# Subscribe in GenServer
defmodule MyApp.OrderTracker do
  use GenServer

  def init(order_id) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "order:#{order_id}")
    {:ok, %{order_id: order_id}}
  end

  def handle_info({:order_updated, order}, state) do
    Logger.info("Order #{state.order_id} updated: #{order.status}")
    {:noreply, state}
  end
end
```

---

## Step 904: LiveView Real-Time Updates

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "dashboard:updates")
      :timer.send_interval(5_000, :refresh_stats)
    end

    {:ok, load_initial(socket)}
  end

  def handle_info(:refresh_stats, socket) do
    {:noreply, assign(socket, :stats, MyApp.Stats.current())}
  end

  def handle_info({:new_order, order}, socket) do
    {:noreply,
     socket
     |> update(:recent_orders, fn orders -> [order | Enum.take(orders, 9)] end)
     |> update(:stats, &Map.update!(&1, :order_count, fn c -> c + 1 end))}
  end

  def handle_info({:payment_received, payment}, socket) do
    {:noreply,
     socket
     |> update(:stats, fn stats ->
       %{stats | revenue: Decimal.add(stats.revenue, payment.amount)}
     end)
     |> push_event("confetti", %{})}
  end

  defp load_initial(socket) do
    assign(socket,
      stats:         MyApp.Stats.current(),
      recent_orders: MyApp.Orders.recent(10)
    )
  end
end
```

---

## Step 905: WebSocket Binary Messages

```elixir
defmodule MyAppWeb.BinaryChannel do
  use MyAppWeb, :channel

  # Handle binary messages (e.g., audio/video streaming)
  
  def join("stream:" <> stream_id, _params, socket) do
    {:ok, assign(socket, :stream_id, stream_id)}
  end

  def handle_in("audio_chunk", %{data: data}, socket) when is_binary(data) do
    # Process binary audio chunk
    case decode_audio(data) do
      {:ok, pcm_data} ->
        broadcast!(socket, "audio", %{data: Base.encode64(pcm_data)})
      {:error, _} ->
        :noreply
    end
    {:noreply, socket}
  end

  # Send binary data to client
  def handle_info({:video_frame, frame_data}, socket) do
    push(socket, "video_frame", %{data: Base.encode64(frame_data)})
    {:noreply, socket}
  end

  # Custom serializer for binary frames
  defmodule BinarySerializer do
    @behaviour Phoenix.Socket.Serializer

    def fastlane!(%{event: event, payload: %{data: binary}}) when is_binary(binary) do
      # Pack: 2 bytes event length + event + payload
      event_bytes = event |> :erlang.iolist_to_binary()
      event_len   = byte_size(event_bytes)
      {:socket_push, :binary, <<event_len::16, event_bytes::binary, binary::binary>>}
    end
  end
end
```

---

## Step 906: Background Job Broadcasting

```elixir
defmodule MyApp.Workers.ExportWorker do
  use Oban.Worker, queue: :exports

  def perform(%Oban.Job{args: %{"user_id" => user_id, "format" => format}}) do
    broadcast_progress(user_id, 0, "Starting export...")

    data = MyApp.Exports.gather_data(user_id, fn percent, msg ->
      broadcast_progress(user_id, percent, msg)
    end)

    broadcast_progress(user_id, 80, "Formatting data...")
    file_url = MyApp.Exports.format_and_upload(data, format)

    broadcast_progress(user_id, 100, "Export complete!")
    broadcast_complete(user_id, file_url)

    :ok
  end

  defp broadcast_progress(user_id, percent, message) do
    Phoenix.PubSub.broadcast(MyApp.PubSub, "exports:#{user_id}", {:progress, %{
      percent: percent,
      message: message,
      at:      DateTime.utc_now()
    }})
  end

  defp broadcast_complete(user_id, file_url) do
    Phoenix.PubSub.broadcast(MyApp.PubSub, "exports:#{user_id}", {:complete, %{
      file_url: file_url,
      at:       DateTime.utc_now()
    }})
  end
end

# LiveView tracking export progress
defmodule MyAppWeb.ExportLive do
  use MyAppWeb, :live_view

  def handle_event("start_export", %{"format" => format}, socket) do
    user_id = socket.assigns.current_user.id
    Phoenix.PubSub.subscribe(MyApp.PubSub, "exports:#{user_id}")
    Oban.insert!(MyApp.Workers.ExportWorker.new(%{user_id: user_id, format: format}))
    {:noreply, assign(socket, :exporting, true)}
  end

  def handle_info({:progress, %{percent: p, message: m}}, socket) do
    {:noreply, assign(socket, progress: p, status_message: m)}
  end

  def handle_info({:complete, %{file_url: url}}, socket) do
    {:noreply, assign(socket, exporting: false, download_url: url)}
  end
end
```

---

## Step 907: Collaborative Editing

```elixir
defmodule MyAppWeb.DocumentChannel do
  use MyAppWeb, :channel

  # Operational Transformation for collaborative editing

  def join("document:" <> doc_id, _params, socket) do
    doc = MyApp.Documents.get!(doc_id)
    {:ok, %{content: doc.content, version: doc.version}, assign(socket, doc_id: doc_id)}
  end

  def handle_in("operation", %{"op" => op, "version" => client_version}, socket) do
    doc_id = socket.assigns.doc_id
    
    case MyApp.Documents.apply_operation(doc_id, op, client_version) do
      {:ok, server_op, new_version} ->
        # Broadcast transformed operation to others
        broadcast_from!(socket, "operation", %{
          op:      server_op,
          version: new_version,
          user:    socket.assigns.user_id
        })
        {:reply, {:ok, %{op: server_op, version: new_version}}, socket}
        
      {:error, :version_conflict} ->
        # Send current document state for client to merge
        doc = MyApp.Documents.get!(doc_id)
        {:reply, {:error, %{
          reason:  "version_conflict",
          content: doc.content,
          version: doc.version
        }}, socket}
    end
  end

  def handle_in("cursor", %{"position" => pos}, socket) do
    broadcast_from!(socket, "cursor_moved", %{
      user_id:  socket.assigns.user_id,
      position: pos
    })
    {:noreply, socket}
  end
end
```

---

## Step 908: Live Notifications

```elixir
defmodule MyApp.Notifications do
  def send(user_id, notification) do
    # Save to DB
    {:ok, notif} = MyApp.Repo.insert(%Notification{
      user_id: user_id,
      type:    notification.type,
      title:   notification.title,
      body:    notification.body,
      data:    notification.data,
      read:    false
    })

    # Broadcast in-app
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "user:#{user_id}",
      {:notification, notif}
    )

    # Push notification (mobile)
    if notification.push do
      MyApp.PushService.send(user_id, notification)
    end

    {:ok, notif}
  end

  def mark_read(notification_id, user_id) do
    Notification
    |> where(id: ^notification_id, user_id: ^user_id)
    |> Repo.update_all(set: [read: true, read_at: DateTime.utc_now()])
  end

  def mark_all_read(user_id) do
    Notification
    |> where(user_id: ^user_id, read: false)
    |> Repo.update_all(set: [read: true, read_at: DateTime.utc_now()])
  end
end

# LiveView notification bell
defmodule MyAppWeb.NotificationBell do
  use MyAppWeb, :live_component

  def update(%{user_id: user_id}, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "user:#{user_id}")
    end

    {:ok, assign(socket,
      notifications: MyApp.Notifications.recent_unread(user_id),
      count:         MyApp.Notifications.unread_count(user_id)
    )}
  end

  def handle_info({:notification, notif}, socket) do
    {:noreply,
     socket
     |> update(:notifications, &[notif | &1])
     |> update(:count, &(&1 + 1))}
  end
end
```

---

## Step 909: Typing Indicators

```elixir
defmodule MyApp.TypingIndicator do
  @typing_timeout 3_000  # clear after 3 seconds of no activity

  def user_typing(channel_pid, user_id) do
    Phoenix.Channel.broadcast_from!(channel_pid, "typing", %{
      user_id: user_id,
      typing:  true
    })

    # Schedule clearing the indicator
    Process.send_after(self(), {:clear_typing, user_id}, @typing_timeout)
  end

  def user_stopped_typing(channel_pid, user_id) do
    Phoenix.Channel.broadcast_from!(channel_pid, "typing", %{
      user_id: user_id,
      typing:  false
    })
  end
end

# In channel
def handle_in("typing", %{"is_typing" => true}, socket) do
  user_id = socket.assigns.user.id
  broadcast_from!(socket, "typing", %{user_id: user_id, typing: true})
  Process.send_after(self(), {:stop_typing, user_id}, 3_000)
  {:noreply, socket}
end

def handle_info({:stop_typing, user_id}, socket) do
  broadcast!(socket, "typing", %{user_id: user_id, typing: false})
  {:noreply, socket}
end
```

---

## Step 910: Real-Time Analytics

```elixir
defmodule MyApp.RealtimeAnalytics do
  use GenServer

  @flush_interval 5_000

  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)

  def track(event_type, user_id, metadata \\ %{}) do
    GenServer.cast(__MODULE__, {:track, event_type, user_id, metadata})
  end

  def current_stats do
    GenServer.call(__MODULE__, :stats)
  end

  def init(_) do
    schedule_flush()
    {:ok, %{events: [], stats: initial_stats()}}
  end

  def handle_cast({:track, type, user_id, metadata}, state) do
    event = %{type: type, user_id: user_id, metadata: metadata, at: DateTime.utc_now()}
    
    new_stats = update_stats(state.stats, event)
    
    # Broadcast real-time update
    Phoenix.PubSub.broadcast(MyApp.PubSub, "analytics", {:stats_updated, new_stats})
    
    {:noreply, %{state | events: [event | state.events], stats: new_stats}}
  end

  def handle_info(:flush, state) do
    # Persist events to database asynchronously
    events = state.events
    Task.start(fn -> flush_to_db(events) end)
    schedule_flush()
    {:noreply, %{state | events: []}}
  end

  defp update_stats(stats, %{type: "page_view"}) do
    %{stats | page_views: stats.page_views + 1}
  end
  defp update_stats(stats, %{type: "purchase", metadata: %{amount: amount}}) do
    %{stats | revenue: Decimal.add(stats.stats[:revenue] || 0, amount)}
  end
  defp update_stats(stats, _), do: stats

  defp initial_stats, do: %{page_views: 0, active_users: 0, revenue: 0}
  defp schedule_flush, do: Process.send_after(self(), :flush, @flush_interval)
end
```

---

## สรุป Part 84

✅ **Step 901** - Channels advanced  
✅ **Step 902** - Presence tracking  
✅ **Step 903** - PubSub patterns  
✅ **Step 904** - LiveView real-time  
✅ **Step 905** - WebSocket binary  
✅ **Step 906** - Background job broadcasting  
✅ **Step 907** - Collaborative editing  
✅ **Step 908** - Live notifications  
✅ **Step 909** - Typing indicators  
✅ **Step 910** - Real-time analytics  

➡️ [Part 85: Production Readiness](./part-85-production.md)
