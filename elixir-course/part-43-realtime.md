# Part 43: Real-Time Systems (Steps 481-500)

## Step 481: Real-Time Architecture

```
Real-Time in Elixir:
- Phoenix Channels: WebSocket multiplexed channels
- Phoenix LiveView: server-rendered reactive UI
- Phoenix PubSub: distributed pub/sub
- BEAM: actor model = natural fit for real-time

Latency tiers:
- <10ms:  in-memory (ETS, process state)
- <50ms:  local Redis
- <100ms: database queries
- <500ms: external service
```

---

## Step 482: Real-Time Dashboard

```elixir
defmodule MyAppWeb.RealtimeDashboardLive do
  use MyAppWeb, :live_view
  
  alias Phoenix.PubSub
  
  def mount(_params, _session, socket) do
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "dashboard:metrics")
      PubSub.subscribe(MyApp.PubSub, "dashboard:alerts")
    end
    
    {:ok, assign(socket,
      metrics:     get_initial_metrics(),
      alerts:      [],
      connections: get_connection_count()
    )}
  end
  
  def handle_info({:metrics_updated, metrics}, socket) do
    {:noreply, assign(socket, metrics: metrics)}
  end
  
  def handle_info({:alert, alert}, socket) do
    alerts = [alert | Enum.take(socket.assigns.alerts, 9)]
    {:noreply, assign(socket, alerts: alerts)}
  end
  
  def render(assigns) do
    ~H"""
    <div id="dashboard" class="realtime-dashboard">
      <h1>Live Dashboard</h1>
      
      <div class="metrics-grid">
        <.metric_card title="RPS" value={@metrics.requests_per_second} />
        <.metric_card title="Latency P99" value={"#{@metrics.p99}ms"} />
        <.metric_card title="Active Users" value={@metrics.active_users} />
        <.metric_card title="Error Rate" value={"#{@metrics.error_rate}%"} />
      </div>
      
      <div class="alerts-panel">
        <h2>Recent Alerts</h2>
        <%= for alert <- @alerts do %>
          <div class={"alert alert-#{alert.severity}"}>
            <%= alert.message %>
          </div>
        <% end %>
      </div>
    </div>
    """
  end
  
  defp metric_card(assigns) do
    ~H"""
    <div class="metric-card">
      <span class="title"><%= @title %></span>
      <span class="value"><%= @value %></span>
    </div>
    """
  end
end
```

---

## Step 483: Collaborative Editing

```elixir
defmodule MyAppWeb.CollaborativeEditorLive do
  use MyAppWeb, :live_view
  
  def mount(%{"doc_id" => doc_id}, session, socket) do
    user = get_user(session)
    
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "doc:#{doc_id}")
      # Track presence
      MyAppWeb.Presence.track(self(), "doc:#{doc_id}", user.id, %{
        name:  user.name,
        color: generate_color(user.id)
      })
    end
    
    doc     = MyApp.Documents.get!(doc_id)
    users   = MyAppWeb.Presence.list("doc:#{doc_id}")
    
    {:ok, assign(socket,
      doc:     doc,
      content: doc.content,
      users:   users,
      cursor:  nil
    )}
  end
  
  def handle_event("content_change", %{"content" => content, "cursor" => cursor}, socket) do
    doc_id = socket.assigns.doc.id
    
    # Broadcast to other users
    Phoenix.PubSub.broadcast(MyApp.PubSub, "doc:#{doc_id}", {
      :content_updated,
      %{content: content, from: socket.assigns.current_user.id}
    })
    
    # Debounced save
    Process.send_after(self(), {:save, content}, 1_000)
    
    {:noreply, assign(socket, content: content)}
  end
  
  def handle_info({:content_updated, %{content: content, from: from_id}}, socket) do
    if from_id != socket.assigns.current_user.id do
      {:noreply, push_event(socket, "remote_update", %{content: content})}
    else
      {:noreply, socket}
    end
  end
  
  def handle_info({:save, content}, socket) do
    MyApp.Documents.update(socket.assigns.doc, %{content: content})
    {:noreply, socket}
  end
  
  def handle_info(%Phoenix.Socket.Broadcast{event: "presence_diff"}, socket) do
    users = MyAppWeb.Presence.list("doc:#{socket.assigns.doc.id}")
    {:noreply, assign(socket, users: users)}
  end
end
```

---

## Step 484: Real-Time Chat

```elixir
defmodule MyAppWeb.ChatChannel do
  use MyAppWeb, :channel
  
  alias MyAppWeb.Presence
  
  def join("room:" <> room_id, _params, socket) do
    room = MyApp.Chat.get_room!(room_id)
    
    send(self(), {:after_join, room_id})
    
    {:ok, assign(socket, room_id: room_id, room: room)}
  end
  
  def handle_info({:after_join, room_id}, socket) do
    # Track presence
    {:ok, _} = Presence.track(socket, socket.assigns.current_user.id, %{
      name:    socket.assigns.current_user.name,
      status:  :online,
      joined_at: DateTime.utc_now() |> DateTime.to_iso8601()
    })
    
    # Send presence state to joining user
    push(socket, "presence_state", Presence.list(socket))
    
    # Send message history
    messages = MyApp.Chat.get_messages(room_id, limit: 50)
    push(socket, "history", %{messages: messages})
    
    {:noreply, socket}
  end
  
  def handle_in("new_message", %{"body" => body}, socket) do
    user = socket.assigns.current_user
    
    case MyApp.Chat.create_message(%{
      room_id: socket.assigns.room_id,
      user_id: user.id,
      body:    body
    }) do
      {:ok, message} ->
        broadcast!(socket, "new_message", %{
          id:         message.id,
          body:       message.body,
          user:       %{id: user.id, name: user.name},
          inserted_at: message.inserted_at |> DateTime.to_iso8601()
        })
        {:reply, :ok, socket}
      
      {:error, cs} ->
        {:reply, {:error, %{errors: format_errors(cs)}}, socket}
    end
  end
  
  def handle_in("typing", _params, socket) do
    broadcast_from!(socket, "user_typing", %{
      user_id: socket.assigns.current_user.id,
      name:    socket.assigns.current_user.name
    })
    {:noreply, socket}
  end
end
```

---

## Step 485: Presence Tracking

```elixir
defmodule MyAppWeb.Presence do
  use Phoenix.Presence,
    otp_app: :my_app,
    pubsub_server: MyApp.PubSub
end

# Show online users in LiveView
defmodule MyAppWeb.OnlineUsersLive do
  use MyAppWeb, :live_view
  
  alias MyAppWeb.Presence
  
  def mount(_params, session, socket) do
    user = get_user(session)
    topic = "online_users"
    
    if connected?(socket) do
      {:ok, _} = Presence.track(self(), topic, user.id, %{
        name:     user.name,
        status:   :online,
        online_at: System.system_time(:second)
      })
      
      Phoenix.PubSub.subscribe(MyApp.PubSub, topic)
    end
    
    presences = Presence.list(topic)
    
    {:ok, assign(socket,
      online_users: presences_to_list(presences),
      current_user: user
    )}
  end
  
  def handle_info(%Phoenix.Socket.Broadcast{event: "presence_diff", payload: diff}, socket) do
    presences = socket.assigns.online_users
      |> apply_joins(diff.joins)
      |> apply_leaves(diff.leaves)
    
    {:noreply, assign(socket, online_users: presences)}
  end
  
  defp presences_to_list(presences) do
    Enum.map(presences, fn {user_id, %{metas: [meta | _]}} ->
      Map.put(meta, :id, user_id)
    end)
  end
  
  defp apply_joins(users, joins) do
    new_users = joins
      |> Enum.map(fn {user_id, %{metas: [meta | _]}} -> Map.put(meta, :id, user_id) end)
    
    users ++ new_users
  end
  
  defp apply_leaves(users, leaves) do
    leave_ids = Map.keys(leaves)
    Enum.reject(users, fn u -> u.id in leave_ids end)
  end
end
```

---

## Step 486: Live Notifications

```elixir
defmodule MyApp.Notifications do
  alias Phoenix.PubSub
  
  @topic_prefix "notifications:"
  
  def send_to_user(user_id, notification) do
    PubSub.broadcast(MyApp.PubSub, topic(user_id), {:notification, notification})
  end
  
  def broadcast_global(notification) do
    PubSub.broadcast(MyApp.PubSub, "notifications:global", {:notification, notification})
  end
  
  defp topic(user_id), do: "#{@topic_prefix}#{user_id}"
end

defmodule MyAppWeb.NotificationComponent do
  use MyAppWeb, :live_component
  
  def mount(socket) do
    if connected?(socket) do
      user_id = socket.assigns.current_user.id
      Phoenix.PubSub.subscribe(MyApp.PubSub, "notifications:#{user_id}")
      Phoenix.PubSub.subscribe(MyApp.PubSub, "notifications:global")
    end
    
    {:ok, assign(socket, notifications: [], unread: 0)}
  end
  
  def handle_info({:notification, notif}, socket) do
    notifications = [notif | Enum.take(socket.assigns.notifications, 19)]
    unread = socket.assigns.unread + 1
    
    {:noreply, assign(socket, notifications: notifications, unread: unread)}
  end
  
  def handle_event("mark_read", _params, socket) do
    {:noreply, assign(socket, unread: 0)}
  end
  
  def render(assigns) do
    ~H"""
    <div class="notifications">
      <button phx-click="mark_read" phx-target={@myself}>
        Notifications <span class="badge"><%= @unread %></span>
      </button>
      <%= for notif <- @notifications do %>
        <div class="notification"><%= notif.message %></div>
      <% end %>
    </div>
    """
  end
end
```

---

## Step 487: Streaming Data

```elixir
defmodule MyAppWeb.DataStreamLive do
  use MyAppWeb, :live_view
  
  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Subscribe to data stream
      Phoenix.PubSub.subscribe(MyApp.PubSub, "sensor_data")
    end
    
    # Use streams for efficient DOM updates
    {:ok, stream(socket, :readings, [])}
  end
  
  def handle_info({:sensor_reading, reading}, socket) do
    {:noreply, stream_insert(socket, :readings, reading, at: 0, limit: 100)}
  end
  
  def render(assigns) do
    ~H"""
    <div id="readings">
      <table>
        <tbody id="readings-body" phx-update="stream">
          <tr :for={{dom_id, reading} <- @streams.readings} id={dom_id}>
            <td><%= reading.sensor_id %></td>
            <td><%= reading.value %></td>
            <td><%= reading.timestamp %></td>
          </tr>
        </tbody>
      </table>
    </div>
    """
  end
end

# Stream publisher
defmodule MyApp.SensorPublisher do
  use GenServer
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def init(_) do
    :timer.send_interval(100, :publish)
    {:ok, %{}}
  end
  
  def handle_info(:publish, state) do
    reading = %{
      id:         System.unique_integer([:positive]),
      sensor_id:  "sensor-#{:rand.uniform(5)}",
      value:      :rand.uniform() * 100 |> Float.round(2),
      timestamp:  DateTime.utc_now() |> DateTime.to_iso8601()
    }
    
    Phoenix.PubSub.broadcast(MyApp.PubSub, "sensor_data", {:sensor_reading, reading})
    {:noreply, state}
  end
end
```

---

## Step 488: Rate-Limited Real-Time Updates

```elixir
defmodule MyAppWeb.RateLimitedLive do
  use MyAppWeb, :live_view
  
  # Throttle UI updates to max 10/second
  @max_updates_per_second 10
  @min_interval_ms div(1000, @max_updates_per_second)
  
  def mount(_params, _session, socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "high_frequency_events")
    
    {:ok, assign(socket,
      data:         [],
      pending:      [],
      last_update:  0
    )}
  end
  
  def handle_info({:event, data}, socket) do
    now = System.monotonic_time(:millisecond)
    elapsed = now - socket.assigns.last_update
    
    socket = %{socket | assigns: %{socket.assigns | pending: [data | socket.assigns.pending]}}
    
    if elapsed >= @min_interval_ms do
      flush_updates(socket, now)
    else
      schedule_flush(elapsed)
      {:noreply, socket}
    end
  end
  
  def handle_info(:flush, socket) do
    now = System.monotonic_time(:millisecond)
    flush_updates(socket, now)
  end
  
  defp flush_updates(socket, now) do
    new_data = Enum.reverse(socket.assigns.pending) ++ Enum.take(socket.assigns.data, 90)
    
    {:noreply, assign(socket,
      data:        new_data,
      pending:     [],
      last_update: now
    )}
  end
  
  defp schedule_flush(elapsed) do
    delay = max(@min_interval_ms - elapsed, 0)
    Process.send_after(self(), :flush, delay)
  end
end
```

---

## Step 489: WebRTC Signaling

```elixir
defmodule MyAppWeb.VideoCallChannel do
  use MyAppWeb, :channel
  
  def join("video:room:" <> room_id, _params, socket) do
    room = MyApp.VideoRooms.get_or_create(room_id)
    {:ok, assign(socket, room_id: room_id)}
  end
  
  # WebRTC offer from caller
  def handle_in("offer", %{"sdp" => sdp, "to" => target_user_id}, socket) do
    broadcast_from!(socket, "offer", %{
      sdp:    sdp,
      from:   socket.assigns.current_user.id
    })
    {:noreply, socket}
  end
  
  # WebRTC answer from callee
  def handle_in("answer", %{"sdp" => sdp, "to" => target_user_id}, socket) do
    broadcast_from!(socket, "answer", %{
      sdp:  sdp,
      from: socket.assigns.current_user.id
    })
    {:noreply, socket}
  end
  
  # ICE candidate exchange
  def handle_in("ice_candidate", %{"candidate" => candidate}, socket) do
    broadcast_from!(socket, "ice_candidate", %{
      candidate: candidate,
      from:      socket.assigns.current_user.id
    })
    {:noreply, socket}
  end
end
```

---

## Step 490: Load Testing Real-Time

```elixir
# Load test WebSockets with k6
# k6/ws_test.js:
# import ws from 'k6/ws';
# export function setup() {
#   const token = getToken();
#   return { token };
# }
# export default function(data) {
#   ws.connect(`wss://app.example.com/socket/websocket?token=${data.token}`, {}, function(socket) {
#     socket.on('open', () => {
#       socket.send(JSON.stringify({topic: "room:1", event: "phx_join", payload: {}, ref: "1"}));
#     });
#     socket.on('message', (data) => {
#       const msg = JSON.parse(data);
#       if (msg.event === 'phx_reply') {
#         socket.send(JSON.stringify({
#           topic: "room:1",
#           event: "new_message",
#           payload: {body: "Hello!"},
#           ref: "2"
#         }));
#       }
#     });
#     socket.setTimeout(() => socket.close(), 10000);
#   });
# }

# Elixir load test via processes
defmodule MyApp.LoadTest do
  def run_ws_test(n_clients) do
    results = Task.async_stream(
      1..n_clients,
      fn i ->
        simulate_client("user_#{i}")
      end,
      max_concurrency: n_clients,
      timeout: 30_000
    )
    |> Enum.to_list()
    
    success = Enum.count(results, &match?({:ok, :ok}, &1))
    IO.puts("#{success}/#{n_clients} clients succeeded")
  end
  
  defp simulate_client(user) do
    {:ok, socket} = connect(user)
    join_channel(socket, "room:1")
    send_messages(socket, 10)
    disconnect(socket)
    :ok
  end
end
```

---

## สรุป Part 43

✅ **Step 481** - Real-time architecture  
✅ **Step 482** - Live dashboard  
✅ **Step 483** - Collaborative editing  
✅ **Step 484** - Real-time chat  
✅ **Step 485** - Presence tracking  
✅ **Step 486** - Live notifications  
✅ **Step 487** - Streaming data  
✅ **Step 488** - Rate-limited updates  
✅ **Step 489** - WebRTC signaling  
✅ **Step 490** - Load testing  

➡️ [Part 44: Database Advanced](./part-44-database-advanced.md)
