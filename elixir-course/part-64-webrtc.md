# Part 64: WebRTC & Real-Time Media (Steps 701-720)

## Step 701: WebRTC Fundamentals

```
WebRTC Architecture:

Browser A                    Signaling Server              Browser B
    |                         (Phoenix Channel)                |
    |------ SDP Offer ------->|                               |
    |                         |------- SDP Offer ------------>|
    |                         |<------ SDP Answer ------------|
    |<----- SDP Answer -------|                               |
    |                         |                               |
    |---- ICE Candidate ----->|                               |
    |                         |---- ICE Candidate ----------->|
    |                         |<--- ICE Candidate ------------|
    |<--- ICE Candidate ------|                               |
    |                                                         |
    |<============== Direct P2P Connection ==================>|
    |          (audio/video/data channel)                     |

Key components:
- SDP (Session Description Protocol): describes media capabilities
- ICE (Interactive Connectivity Establishment): NAT traversal
- TURN/STUN servers: relay traffic through firewalls
- RTCPeerConnection: browser API for WebRTC
```

---

## Step 702: Phoenix Signaling Server

```elixir
defmodule MyAppWeb.RoomChannel do
  use MyAppWeb, :channel

  def join("room:" <> room_id, _params, socket) do
    # Track participants
    {:ok, _} = Presence.track(socket, socket.assigns.user_id, %{
      joined_at: System.system_time(:second)
    })

    # Notify existing participants
    send(self(), :after_join)
    
    {:ok, assign(socket, :room_id, room_id)}
  end

  def handle_info(:after_join, socket) do
    participants = Presence.list(socket)
    push(socket, "presence_state", participants)
    {:noreply, socket}
  end

  # Relay WebRTC signaling messages
  def handle_in("sdp_offer", %{"target" => target, "sdp" => sdp}, socket) do
    MyAppWeb.Endpoint.broadcast!("user:#{target}", "sdp_offer", %{
      from: socket.assigns.user_id,
      sdp: sdp
    })
    {:noreply, socket}
  end

  def handle_in("sdp_answer", %{"target" => target, "sdp" => sdp}, socket) do
    MyAppWeb.Endpoint.broadcast!("user:#{target}", "sdp_answer", %{
      from: socket.assigns.user_id,
      sdp: sdp
    })
    {:noreply, socket}
  end

  def handle_in("ice_candidate", %{"target" => target, "candidate" => candidate}, socket) do
    MyAppWeb.Endpoint.broadcast!("user:#{target}", "ice_candidate", %{
      from: socket.assigns.user_id,
      candidate: candidate
    })
    {:noreply, socket}
  end

  def handle_in("leave", _params, socket) do
    Presence.untrack(socket, socket.assigns.user_id)
    {:noreply, socket}
  end
end
```

---

## Step 703: WebRTC JavaScript Client

```javascript
// assets/js/webrtc.js
import { Socket, Presence } from "phoenix"

const configuration = {
  iceServers: [
    { urls: "stun:stun.l.google.com:19302" },
    {
      urls:       "turn:your-turn-server.com:3478",
      username:   "username",
      credential: "password"
    }
  ]
}

class WebRTCClient {
  constructor(userId, channelTopic) {
    this.userId    = userId
    this.peers     = {}
    this.localStream = null
    
    const socket  = new Socket("/socket", { params: { token: userToken } })
    socket.connect()
    
    this.channel = socket.channel(channelTopic)
    this.setupChannelListeners()
    this.channel.join()
  }

  async start() {
    this.localStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: true
    })
    document.getElementById("local-video").srcObject = this.localStream
  }

  setupChannelListeners() {
    this.channel.on("sdp_offer", async ({ from, sdp }) => {
      const peer = this.createPeer(from)
      await peer.setRemoteDescription(new RTCSessionDescription(sdp))
      const answer = await peer.createAnswer()
      await peer.setLocalDescription(answer)
      this.channel.push("sdp_answer", { target: from, sdp: answer })
    })

    this.channel.on("sdp_answer", async ({ from, sdp }) => {
      await this.peers[from]?.setRemoteDescription(new RTCSessionDescription(sdp))
    })

    this.channel.on("ice_candidate", ({ from, candidate }) => {
      this.peers[from]?.addIceCandidate(new RTCIceCandidate(candidate))
    })

    this.channel.on("presence_state", (state) => {
      Presence.syncState(this.presence, state)
      Object.keys(state).forEach(userId => {
        if (userId !== this.userId) this.callUser(userId)
      })
    })
  }

  createPeer(userId) {
    const peer = new RTCPeerConnection(configuration)
    this.peers[userId] = peer

    this.localStream.getTracks().forEach(track => peer.addTrack(track, this.localStream))

    peer.onicecandidate = ({ candidate }) => {
      if (candidate) {
        this.channel.push("ice_candidate", { target: userId, candidate })
      }
    }

    peer.ontrack = ({ streams }) => {
      const remoteVideo       = document.createElement("video")
      remoteVideo.srcObject   = streams[0]
      remoteVideo.autoplay    = true
      remoteVideo.setAttribute("data-user-id", userId)
      document.getElementById("remote-videos").appendChild(remoteVideo)
    }

    peer.onconnectionstatechange = () => {
      if (peer.connectionState === "disconnected") {
        this.removePeer(userId)
      }
    }

    return peer
  }

  async callUser(userId) {
    const peer  = this.createPeer(userId)
    const offer = await peer.createOffer()
    await peer.setLocalDescription(offer)
    this.channel.push("sdp_offer", { target: userId, sdp: offer })
  }

  removePeer(userId) {
    this.peers[userId]?.close()
    delete this.peers[userId]
    document.querySelector(`[data-user-id="${userId}"]`)?.remove()
  }
}

export default WebRTCClient
```

---

## Step 704: Screen Sharing

```javascript
// Add screen share capability
class ScreenShare {
  constructor(webrtcClient) {
    this.client = webrtcClient
    this.screenStream = null
    this.originalVideoTrack = null
  }

  async start() {
    this.screenStream = await navigator.mediaDevices.getDisplayMedia({
      video: { cursor: "always" },
      audio: false
    })

    const screenTrack = this.screenStream.getVideoTracks()[0]
    
    // Replace video track in all peer connections
    Object.values(this.client.peers).forEach(peer => {
      const sender = peer.getSenders().find(s => s.track?.kind === "video")
      sender?.replaceTrack(screenTrack)
    })

    // Save original video track
    this.originalVideoTrack = this.client.localStream.getVideoTracks()[0]

    // Show screen in local preview
    document.getElementById("local-video").srcObject = this.screenStream

    // Handle user stopping screen share
    screenTrack.onended = () => this.stop()
  }

  async stop() {
    this.screenStream?.getTracks().forEach(t => t.stop())
    
    // Restore camera
    Object.values(this.client.peers).forEach(peer => {
      const sender = peer.getSenders().find(s => s.track?.kind === "video")
      sender?.replaceTrack(this.originalVideoTrack)
    })

    document.getElementById("local-video").srcObject = this.client.localStream
  }
}
```

---

## Step 705: Recording

```elixir
defmodule MyApp.Recording do
  # Server-side recording integration (e.g., with Janus or Mediasoup)
  
  def start_recording(room_id) do
    # Signal to media server to start recording
    :ok
  end
  
  def stop_recording(room_id) do
    # Get recording URL
    {:ok, "https://recordings.myapp.com/#{room_id}.webm"}
  end
end
```

```javascript
// Client-side recording with MediaRecorder API
class Recorder {
  constructor(stream) {
    this.stream   = stream
    this.recorder = null
    this.chunks   = []
  }

  start() {
    this.recorder = new MediaRecorder(this.stream, {
      mimeType: "video/webm;codecs=vp9,opus"
    })

    this.recorder.ondataavailable = ({ data }) => {
      if (data.size > 0) this.chunks.push(data)
    }

    this.recorder.start(1000)  // collect chunks every second
  }

  stop() {
    return new Promise(resolve => {
      this.recorder.onstop = () => {
        const blob = new Blob(this.chunks, { type: "video/webm" })
        resolve(blob)
      }
      this.recorder.stop()
    })
  }

  async saveToServer(blob) {
    const formData = new FormData()
    formData.append("recording", blob, "recording.webm")
    
    const response = await fetch("/api/recordings", {
      method: "POST",
      body: formData
    })
    return response.json()
  }
}
```

---

## Step 706: TURN Server Integration

```elixir
defmodule MyApp.TurnServer do
  # Generate temporary TURN credentials (for coturn)
  
  @secret System.get_env("TURN_SECRET")
  @ttl    3600  # 1 hour

  def credentials(user_id) do
    timestamp = System.os_time(:second) + @ttl
    username  = "#{timestamp}:#{user_id}"
    
    password = :crypto.mac(:hmac, :sha, @secret, username)
      |> Base.encode64()

    %{
      username:  username,
      password:  password,
      ttl:       @ttl,
      uris:      [
        "stun:turn.myapp.com:3478",
        "turn:turn.myapp.com:3478?transport=udp",
        "turn:turn.myapp.com:3478?transport=tcp"
      ]
    }
  end
end

defmodule MyAppWeb.TurnController do
  use MyAppWeb, :controller

  def credentials(conn, _params) do
    user = conn.assigns.current_user
    credentials = MyApp.TurnServer.credentials(user.id)
    json(conn, credentials)
  end
end
```

---

## Step 707: Real-Time Whiteboard

```elixir
defmodule MyAppWeb.WhiteboardChannel do
  use MyAppWeb, :channel

  def join("whiteboard:" <> board_id, _params, socket) do
    # Load existing drawing state from ETS/Redis
    state = get_board_state(board_id)
    {:ok, %{state: state}, assign(socket, :board_id, board_id)}
  end

  def handle_in("draw", %{"type" => type, "data" => data}, socket) do
    board_id = socket.assigns.board_id
    
    # Store the draw operation
    append_operation(board_id, %{
      type:      type,
      data:      data,
      user_id:   socket.assigns.user_id,
      timestamp: System.system_time(:millisecond)
    })

    # Broadcast to all other participants
    broadcast_from!(socket, "draw", %{
      type:    type,
      data:    data,
      user_id: socket.assigns.user_id
    })

    {:noreply, socket}
  end

  def handle_in("clear", _params, socket) do
    board_id = socket.assigns.board_id
    clear_board(board_id)
    broadcast!(socket, "clear", %{user_id: socket.assigns.user_id})
    {:noreply, socket}
  end

  defp get_board_state(board_id) do
    case Redix.command(:redix, ["GET", "whiteboard:#{board_id}"]) do
      {:ok, nil}   -> []
      {:ok, state} -> Jason.decode!(state)
    end
  end

  defp append_operation(board_id, op) do
    {:ok, state} = Redix.command(:redix, ["GET", "whiteboard:#{board_id}"])
    operations   = if state, do: Jason.decode!(state), else: []
    new_state    = operations ++ [op]
    Redix.command(:redix, ["SET", "whiteboard:#{board_id}", Jason.encode!(new_state)])
  end

  defp clear_board(board_id) do
    Redix.command(:redix, ["DEL", "whiteboard:#{board_id}"])
  end
end
```

---

## Step 708: Video Room LiveView

```elixir
defmodule MyAppWeb.VideoRoomLive do
  use MyAppWeb, :live_view
  alias MyAppWeb.Presence

  def mount(%{"room_id" => room_id}, session, socket) do
    user = get_user_from_session(session)

    if connected?(socket) do
      MyAppWeb.Endpoint.subscribe("room:#{room_id}")
      Presence.track(self(), "room:#{room_id}", user.id, %{
        name:      user.name,
        joined_at: System.system_time(:second)
      })
    end

    participants = Presence.list("room:#{room_id}")

    {:ok,
     socket
     |> assign(:room_id, room_id)
     |> assign(:user, user)
     |> assign(:participants, participants)
     |> assign(:turn_credentials, MyApp.TurnServer.credentials(user.id))}
  end

  def handle_info(%{event: "presence_diff", payload: diff}, socket) do
    participants = Presence.list("room:#{socket.assigns.room_id}")
    {:noreply, assign(socket, :participants, participants)}
  end

  def render(assigns) do
    ~H"""
    <div id="video-room"
         phx-hook="VideoRoom"
         data-room-id={@room_id}
         data-user-id={@user.id}
         data-turn-credentials={Jason.encode!(@turn_credentials)}>
      
      <div class="video-grid">
        <video id="local-video" autoplay muted class="local-video"></video>
        <div id="remote-videos"></div>
      </div>
      
      <div class="participants">
        <h3>Participants (<%= map_size(@participants) %>)</h3>
        <ul>
          <li :for={{user_id, meta} <- @participants}>
            <%= meta.metas |> List.first() |> Map.get(:name, user_id) %>
          </li>
        </ul>
      </div>
      
      <div class="controls">
        <button phx-click="toggle-audio">Toggle Audio</button>
        <button phx-click="toggle-video">Toggle Video</button>
        <button phx-click="share-screen">Share Screen</button>
        <button phx-click="leave">Leave</button>
      </div>
    </div>
    """
  end
end
```

---

## Step 709: Bandwidth Adaptation

```javascript
// Monitor WebRTC stats and adapt quality
class QualityAdapter {
  constructor(peerConnections) {
    this.peers   = peerConnections
    this.history = []
    this.intervalId = null
  }

  start() {
    this.intervalId = setInterval(() => this.measure(), 5000)
  }

  stop() {
    clearInterval(this.intervalId)
  }

  async measure() {
    for (const [peerId, pc] of Object.entries(this.peers)) {
      const stats = await pc.getStats()
      
      stats.forEach(report => {
        if (report.type === "inbound-rtp" && report.kind === "video") {
          this.history.push({
            peerId,
            timestamp:    report.timestamp,
            packetsLost:  report.packetsLost,
            jitter:       report.jitter,
            bytesReceived: report.bytesReceived
          })
        }
      })
    }

    this.adapt()
  }

  adapt() {
    const recent = this.history.slice(-10)
    if (recent.length < 2) return

    const lossRate = recent.reduce((sum, s) => sum + s.packetsLost, 0) / recent.length
    const avgJitter = recent.reduce((sum, s) => sum + s.jitter, 0) / recent.length

    if (lossRate > 0.05 || avgJitter > 30) {
      this.reduceQuality()
    } else if (lossRate < 0.01 && avgJitter < 10) {
      this.increaseQuality()
    }
  }

  reduceQuality() {
    // Reduce video resolution
    const stream = document.getElementById("local-video").srcObject
    const track  = stream?.getVideoTracks()[0]
    if (track) {
      track.applyConstraints({ width: 320, height: 240, frameRate: 15 })
    }
    console.log("Reduced video quality due to poor network")
  }

  increaseQuality() {
    const stream = document.getElementById("local-video").srcObject
    const track  = stream?.getVideoTracks()[0]
    if (track) {
      track.applyConstraints({ width: 1280, height: 720, frameRate: 30 })
    }
  }
}
```

---

## Step 710: Room Management

```elixir
defmodule MyApp.Rooms do
  use Ecto.Schema

  schema "rooms" do
    field :name,          :string
    field :room_code,     :string
    field :max_users,     :integer, default: 10
    field :active,        :boolean, default: true
    field :recording,     :boolean, default: false
    field :scheduled_at,  :utc_datetime
    belongs_to :host,     MyApp.Accounts.User
    timestamps()
  end

  def create(host, attrs) do
    %__MODULE__{}
    |> changeset(Map.put(attrs, :host_id, host.id))
    |> Ecto.Changeset.put_change(:room_code, generate_code())
    |> MyApp.Repo.insert()
  end

  def get_by_code(code) do
    MyApp.Repo.get_by(__MODULE__, room_code: code, active: true)
  end

  def current_participants(room_id) do
    "room:#{room_id}"
    |> MyAppWeb.Presence.list()
    |> map_size()
  end

  def can_join?(room, user) do
    current = current_participants(room.id)
    current < room.max_users
  end

  defp generate_code do
    :crypto.strong_rand_bytes(4)
    |> Base.encode16()
    |> String.upcase()
  end
end
```

---

## สรุป Part 64

✅ **Step 701** - WebRTC fundamentals  
✅ **Step 702** - Phoenix signaling server  
✅ **Step 703** - JavaScript WebRTC client  
✅ **Step 704** - Screen sharing  
✅ **Step 705** - Recording  
✅ **Step 706** - TURN server  
✅ **Step 707** - Real-time whiteboard  
✅ **Step 708** - Video room LiveView  
✅ **Step 709** - Bandwidth adaptation  
✅ **Step 710** - Room management  

➡️ [Part 65: gRPC & Protocol Buffers](./part-65-grpc.md)
