# Part 23: Phoenix Channels (Steps 241-260)

## Step 241: Phoenix Channels คืออะไร?

```
Phoenix Channels = Real-time bidirectional communication
- WebSocket based
- Topic/Event model
- Client can join/leave channels
- Broadcast to multiple clients
- Presence tracking
- ใช้สำหรับ: Chat, Notifications, Game, Live Updates
```

### 241.1 Channel Architecture

```
Client Browser
    │
    │ WebSocket
    │
Socket (Endpoint)
    │
    ├─ topic: "room:1" → RoomChannel
    ├─ topic: "user:123" → UserChannel
    └─ topic: "game:*" → GameChannel
```

---

## Step 242: Socket และ Channel Setup

```elixir
# lib/my_app_web/channels/user_socket.ex
defmodule MyAppWeb.UserSocket do
  use Phoenix.Socket
  
  # Route topic patterns to channels
  channel "room:*",     MyAppWeb.RoomChannel
  channel "user:*",     MyAppWeb.UserChannel
  channel "game:lobby", MyAppWeb.LobbyChannel
  
  @impl true
  def connect(%{"token" => token}, socket, _connect_info) do
    case Phoenix.Token.verify(MyAppWeb.Endpoint, "user socket", token, max_age: 86400) do
      {:ok, user_id} ->
        {:ok, assign(socket, :user_id, user_id)}
      
      {:error, _reason} ->
        :error
    end
  end
  
  def connect(_params, _socket, _connect_info) do
    :error  # reject unauthenticated connections
  end
  
  @impl true
  def id(socket), do: "users_socket:#{socket.assigns.user_id}"
end

# endpoint.ex - เพิ่ม socket
socket "/socket", MyAppWeb.UserSocket,
  websocket: true,
  longpoll: false
```

---

## Step 243: Room Channel

```elixir
defmodule MyAppWeb.RoomChannel do
  use Phoenix.Channel
  
  alias MyApp.Chat
  alias MyAppWeb.Presence
  
  @impl true
  def join("room:" <> room_id, _params, socket) do
    # Check authorization
    if Chat.room_exists?(room_id) do
      socket = assign(socket, :room_id, room_id)
      
      # Load recent messages
      messages = Chat.get_recent_messages(room_id, limit: 50)
      
      # Track presence after join
      send(self(), :after_join)
      
      {:ok, %{messages: messages}, socket}
    else
      {:error, %{reason: "room not found"}}
    end
  end
  
  @impl true
  def handle_info(:after_join, socket) do
    {:ok, _} = Presence.track(socket, socket.assigns.user_id, %{
      online_at: DateTime.utc_now(),
      user_id: socket.assigns.user_id
    })
    
    # Push presence state to new user
    push(socket, "presence_state", Presence.list(socket))
    
    {:noreply, socket}
  end
  
  # Handle incoming messages
  @impl true
  def handle_in("new_msg", %{"body" => body}, socket) do
    user_id = socket.assigns.user_id
    room_id = socket.assigns.room_id
    
    case Chat.create_message(%{
      room_id: room_id,
      user_id: user_id,
      body: body
    }) do
      {:ok, message} ->
        # Broadcast to everyone in the room
        broadcast!(socket, "new_msg", %{
          id:      message.id,
          body:    message.body,
          user_id: message.user_id,
          at:      message.inserted_at
        })
        {:noreply, socket}
      
      {:error, _changeset} ->
        {:reply, {:error, %{reason: "invalid message"}}, socket}
    end
  end
  
  @impl true
  def handle_in("typing", _params, socket) do
    broadcast_from!(socket, "typing", %{user_id: socket.assigns.user_id})
    {:noreply, socket}
  end
  
  @impl true
  def handle_in("mark_read", %{"message_id" => message_id}, socket) do
    Chat.mark_message_read(socket.assigns.user_id, message_id)
    {:reply, :ok, socket}
  end
  
  @impl true
  def terminate(reason, socket) do
    IO.puts("User #{socket.assigns.user_id} left room #{socket.assigns.room_id}")
    :ok
  end
end
```

---

## Step 244: Presence

```elixir
# lib/my_app_web/channels/presence.ex
defmodule MyAppWeb.Presence do
  use Phoenix.Presence,
    otp_app: :my_app,
    pubsub_server: MyApp.PubSub
end

# Supervisor setup (application.ex)
children = [
  # ...
  MyAppWeb.Presence
]

# ใช้ Presence ใน Channel
defmodule MyAppWeb.RoomChannel do
  use Phoenix.Channel
  alias MyAppWeb.Presence
  
  def join("room:" <> room_id, _params, socket) do
    send(self(), :after_join)
    {:ok, socket}
  end
  
  def handle_info(:after_join, socket) do
    {:ok, _} = Presence.track(socket, socket.assigns.user_id, %{
      username: socket.assigns.username,
      online_at: System.system_time(:second)
    })
    
    push(socket, "presence_state", Presence.list(socket))
    {:noreply, socket}
  end
end
```

---

## Step 245: JavaScript Client

```javascript
// assets/js/socket.js
import { Socket, Presence } from "phoenix"

const token = document.querySelector("meta[name='user-token']")?.content

const socket = new Socket("/socket", {
  params: { token: token }
})

socket.connect()

// Join a channel
const channel = socket.channel("room:lobby", {})

channel.join()
  .receive("ok", (resp) => {
    console.log("Joined room:", resp.messages)
  })
  .receive("error", (resp) => {
    console.log("Failed to join:", resp.reason)
  })

// Send message
function sendMessage(text) {
  channel.push("new_msg", { body: text })
    .receive("ok", () => console.log("sent"))
    .receive("error", (err) => console.log("error:", err))
    .receive("timeout", () => console.log("timeout"))
}

// Receive messages
channel.on("new_msg", (msg) => {
  console.log(`${msg.user_id}: ${msg.body}`)
  appendMessage(msg)
})

// Typing indicators
let typingTimer
input.addEventListener("keypress", () => {
  clearTimeout(typingTimer)
  channel.push("typing", {})
  typingTimer = setTimeout(() => {}, 1000)
})

channel.on("typing", ({ user_id }) => {
  showTypingIndicator(user_id)
})

// Presence
let presence = new Presence(channel)

presence.onSync(() => {
  const users = presence.list((id, { metas: [first, ...rest] }) => {
    return { id, username: first.username }
  })
  renderUserList(users)
})

presence.onJoin((id, current, newPresences) => {
  if (!current) {
    console.log(`${id} joined`)
  }
})

presence.onLeave((id, current, leftPresences) => {
  if (current.metas.length === 0) {
    console.log(`${id} left`)
  }
})
```

---

## Step 246: Channel กับ PubSub

```elixir
defmodule MyAppWeb.NotificationChannel do
  use Phoenix.Channel
  
  def join("notifications:" <> user_id, _params, socket) do
    # Verify this user can access this channel
    if socket.assigns.user_id == String.to_integer(user_id) do
      {:ok, socket}
    else
      {:error, %{reason: "unauthorized"}}
    end
  end
  
  # Server สามารถ broadcast ไปที่ channel ด้วย PubSub
  # เรียกจาก anywhere ใน application
  def notify_user(user_id, event, payload) do
    MyAppWeb.Endpoint.broadcast(
      "notifications:#{user_id}",
      event,
      payload
    )
  end
end

# เรียกจาก business logic
defmodule MyApp.Orders do
  def create_order(user_id, attrs) do
    case insert_order(attrs) do
      {:ok, order} ->
        # แจ้ง user ผ่าน channel
        MyAppWeb.NotificationChannel.notify_user(user_id, "order_created", %{
          order_id: order.id,
          total: order.total
        })
        {:ok, order}
      
      error -> error
    end
  end
end
```

---

## Step 247: Channel Authentication

```elixir
defmodule MyAppWeb.UserSocket do
  use Phoenix.Socket
  
  channel "private:*", MyAppWeb.PrivateChannel
  
  @impl true
  def connect(%{"token" => token}, socket, _connect_info) do
    case verify_token(token) do
      {:ok, user_id} ->
        user = MyApp.Accounts.get_user!(user_id)
        socket = socket
          |> assign(:user_id, user_id)
          |> assign(:user, user)
        {:ok, socket}
      
      {:error, _} ->
        :error
    end
  end
  
  defp verify_token(token) do
    Phoenix.Token.verify(
      MyAppWeb.Endpoint,
      "user auth",
      token,
      max_age: 2 * 24 * 60 * 60  # 2 days
    )
  end
  
  @impl true
  def id(socket), do: "user_socket:#{socket.assigns.user_id}"
end

# สร้าง token ใน controller/LiveView
defmodule MyAppWeb.PageController do
  use MyAppWeb, :controller
  
  def index(conn, _params) do
    user_token = if user = conn.assigns[:current_user] do
      Phoenix.Token.sign(conn, "user auth", user.id)
    end
    
    render(conn, :index, user_token: user_token)
  end
end
```

---

## Step 248: Rate Limiting ใน Channels

```elixir
defmodule MyAppWeb.ChatChannel do
  use Phoenix.Channel
  
  @max_messages_per_minute 30
  
  @impl true
  def join("chat:" <> room_id, _params, socket) do
    socket = assign(socket, :message_count, 0)
    socket = assign(socket, :window_start, System.monotonic_time(:second))
    {:ok, socket}
  end
  
  @impl true
  def handle_in("message", params, socket) do
    case check_rate_limit(socket) do
      {:ok, socket} ->
        do_handle_message(params, socket)
      {:error, :rate_limited} ->
        {:reply, {:error, %{reason: "Rate limit exceeded"}}, socket}
    end
  end
  
  defp check_rate_limit(socket) do
    now = System.monotonic_time(:second)
    window_start = socket.assigns.window_start
    count = socket.assigns.message_count
    
    if now - window_start >= 60 do
      # Reset window
      {:ok, socket |> assign(:window_start, now) |> assign(:message_count, 1)}
    else
      if count >= @max_messages_per_minute do
        {:error, :rate_limited}
      else
        {:ok, update(socket, :message_count, &(&1 + 1))}
      end
    end
  end
  
  defp do_handle_message(%{"body" => body}, socket) do
    broadcast!(socket, "new_message", %{
      body: body,
      user_id: socket.assigns.user_id,
      timestamp: DateTime.utc_now()
    })
    {:noreply, socket}
  end
end
```

---

## Step 249: Channel Testing

```elixir
defmodule MyAppWeb.RoomChannelTest do
  use MyAppWeb.ChannelCase
  
  alias MyAppWeb.UserSocket
  
  setup do
    user = user_fixture()
    token = Phoenix.Token.sign(MyAppWeb.Endpoint, "user auth", user.id)
    
    {:ok, socket} = connect(UserSocket, %{"token" => token})
    
    {:ok, _reply, socket} = subscribe_and_join(socket, "room:1", %{})
    
    {:ok, socket: socket, user: user}
  end
  
  test "sends messages", %{socket: socket} do
    push(socket, "new_msg", %{"body" => "Hello!"})
    
    assert_broadcast "new_msg", %{body: "Hello!"}
  end
  
  test "broadcasts to all users" do
    {:ok, _, socket1} = subscribe_and_join(socket(), "room:1", %{})
    {:ok, _, socket2} = subscribe_and_join(socket(), "room:1", %{})
    
    push(socket1, "new_msg", %{"body" => "Hi everyone!"})
    
    assert_broadcast "new_msg", %{body: "Hi everyone!"}
  end
  
  test "returns error for invalid message", %{socket: socket} do
    ref = push(socket, "new_msg", %{})  # no body
    
    assert_reply ref, :error, %{reason: _}
  end
  
  test "typing indicator is broadcast", %{socket: socket, user: user} do
    push(socket, "typing", %{})
    
    assert_broadcast "typing", %{user_id: ^(user.id)}
  end
end
```

---

## Step 250: Complete Chat Application

```elixir
# Full chat app with rooms, presence, typing indicators

defmodule MyApp.Chat do
  import Ecto.Query
  alias MyApp.{Repo, Chat.{Room, Message}}
  
  def list_rooms do
    Room |> order_by(:name) |> Repo.all()
  end
  
  def create_room(attrs) do
    %Room{} |> Room.changeset(attrs) |> Repo.insert()
  end
  
  def get_room!(id), do: Repo.get!(Room, id)
  def room_exists?(id), do: Repo.exists?(from r in Room, where: r.id == ^id)
  
  def get_recent_messages(room_id, opts \\ []) do
    limit = Keyword.get(opts, :limit, 50)
    
    from(m in Message,
      where: m.room_id == ^room_id,
      order_by: [desc: m.inserted_at],
      limit: ^limit,
      preload: :user
    )
    |> Repo.all()
    |> Enum.reverse()
  end
  
  def create_message(attrs) do
    %Message{}
    |> Message.changeset(attrs)
    |> Repo.insert()
  end
end

defmodule MyAppWeb.ChatRoomChannel do
  use Phoenix.Channel
  alias MyApp.Chat
  alias MyAppWeb.Presence
  
  def join("room:" <> room_id, _params, socket) do
    case Chat.get_room(room_id) do
      {:ok, room} ->
        socket = assign(socket, room: room)
        send(self(), :after_join)
        
        messages = Chat.get_recent_messages(room_id)
        {:ok, %{room: room, messages: messages}, socket}
      
      {:error, :not_found} ->
        {:error, %{reason: "Room not found"}}
    end
  end
  
  def handle_info(:after_join, socket) do
    user = socket.assigns.user
    
    {:ok, _} = Presence.track(socket, user.id, %{
      user_id: user.id,
      username: user.name,
      avatar: user.avatar_url,
      typing: false
    })
    
    push(socket, "presence_state", Presence.list(socket))
    {:noreply, socket}
  end
  
  def handle_in("message:new", %{"body" => body}, socket) do
    user = socket.assigns.user
    room = socket.assigns.room
    
    with {:ok, message} <- Chat.create_message(%{
           room_id: room.id,
           user_id: user.id,
           body: body
         }) do
      broadcast!(socket, "message:new", %{
        id:       message.id,
        body:     message.body,
        user:     %{id: user.id, name: user.name},
        inserted_at: message.inserted_at
      })
      {:noreply, socket}
    end
  end
  
  def handle_in("typing:start", _params, socket) do
    Presence.update(socket, socket.assigns.user.id, fn meta ->
      %{meta | typing: true}
    end)
    {:noreply, socket}
  end
  
  def handle_in("typing:stop", _params, socket) do
    Presence.update(socket, socket.assigns.user.id, fn meta ->
      %{meta | typing: false}
    end)
    {:noreply, socket}
  end
end
```

---

## สรุป Part 23

✅ **Step 241** - Channels overview  
✅ **Step 242** - Socket + Channel setup  
✅ **Step 243** - Room Channel implementation  
✅ **Step 244** - Presence tracking  
✅ **Step 245** - JavaScript client  
✅ **Step 246** - PubSub integration  
✅ **Step 247** - Authentication  
✅ **Step 248** - Rate limiting  
✅ **Step 249** - Channel testing  
✅ **Step 250** - Complete chat app  

➡️ [Part 24: Ecto Advanced](./part-24-ecto-advanced.md)
