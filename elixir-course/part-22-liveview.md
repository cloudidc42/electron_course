# Part 22: Phoenix LiveView (Steps 231-250)

## Step 231: LiveView คืออะไร?

```
Phoenix LiveView = Real-time UI โดยไม่ต้องเขียน JavaScript
- Server-side rendering + WebSocket
- DOM diff update (เหมือน React แต่ทำ server-side)
- Real-time ด้วย WebSocket connection
- ไม่ต้องเขียน API endpoints สำหรับ interactions
- Form validation real-time
```

### 231.1 LiveView Lifecycle

```
Browser ─────── WebSocket ─────── LiveView Process
   │                                     │
   │ GET /page ──────────────────────→  mount/3
   │ ← HTML (full page) ─────────────── render/1
   │                                     │
   │ connect (WebSocket) ─────────────→ mount/3 (connected)
   │                                     │
   │ user click ─────────────────────→ handle_event/3
   │ ← diff only ─────────────────────── render/1 (diff)
```

---

## Step 232: Basic LiveView

```elixir
# lib/my_app_web/live/counter_live.ex
defmodule MyAppWeb.CounterLive do
  use MyAppWeb, :live_view
  
  @impl true
  def mount(_params, _session, socket) do
    {:ok, assign(socket, count: 0)}
  end
  
  @impl true
  def render(assigns) do
    ~H"""
    <div class="counter">
      <h1>Counter: <%= @count %></h1>
      
      <button phx-click="increment">+1</button>
      <button phx-click="decrement">-1</button>
      <button phx-click="reset">Reset</button>
    </div>
    """
  end
  
  @impl true
  def handle_event("increment", _params, socket) do
    {:noreply, update(socket, :count, &(&1 + 1))}
  end
  
  def handle_event("decrement", _params, socket) do
    {:noreply, update(socket, :count, &(&1 - 1))}
  end
  
  def handle_event("reset", _params, socket) do
    {:noreply, assign(socket, count: 0)}
  end
end

# router.ex
live "/counter", CounterLive
```

---

## Step 233: LiveView Forms

```elixir
defmodule MyAppWeb.UserFormLive do
  use MyAppWeb, :live_view
  
  alias MyApp.Accounts
  alias MyApp.Accounts.User
  
  @impl true
  def mount(_params, _session, socket) do
    changeset = Accounts.change_user(%User{})
    {:ok, assign(socket, form: to_form(changeset))}
  end
  
  @impl true
  def render(assigns) do
    ~H"""
    <div class="form-container">
      <h1>Create User</h1>
      
      <.form for={@form} phx-change="validate" phx-submit="save">
        <.input field={@form[:name]}  type="text"  label="Name" />
        <.input field={@form[:email]} type="email" label="Email" />
        <.input field={@form[:password]} type="password" label="Password" />
        
        <div>
          <.button type="submit">Create</.button>
        </div>
      </.form>
      
      <%= if @form.source.valid? do %>
        <p class="text-green">Looks good!</p>
      <% end %>
    </div>
    """
  end
  
  @impl true
  def handle_event("validate", %{"user" => params}, socket) do
    changeset = Accounts.change_user(%User{}, params)
    {:noreply, assign(socket, form: to_form(changeset, action: :validate))}
  end
  
  @impl true
  def handle_event("save", %{"user" => params}, socket) do
    case Accounts.create_user(params) do
      {:ok, user} ->
        {:noreply,
          socket
          |> put_flash(:info, "User created!")
          |> push_navigate(to: ~p"/users/#{user}")}
      
      {:error, changeset} ->
        {:noreply, assign(socket, form: to_form(changeset))}
    end
  end
end
```

---

## Step 234: LiveView กับ Real-time Updates

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  
  @impl true
  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Subscribe to updates เมื่อ WebSocket connected
      Phoenix.PubSub.subscribe(MyApp.PubSub, "metrics")
      :timer.send_interval(1000, self(), :tick)
    end
    
    {:ok, assign(socket,
      metrics: get_metrics(),
      uptime: 0,
      users_online: 0
    )}
  end
  
  @impl true
  def render(assigns) do
    ~H"""
    <div class="dashboard">
      <h1>System Dashboard</h1>
      
      <div class="metrics-grid">
        <div class="metric-card">
          <h3>CPU Usage</h3>
          <div class="metric-value"><%= @metrics.cpu %>%</div>
          <div class="progress-bar" style={"width: #{@metrics.cpu}%"}></div>
        </div>
        
        <div class="metric-card">
          <h3>Memory</h3>
          <div class="metric-value"><%= @metrics.memory_mb %> MB</div>
        </div>
        
        <div class="metric-card">
          <h3>Users Online</h3>
          <div class="metric-value"><%= @users_online %></div>
        </div>
        
        <div class="metric-card">
          <h3>Uptime</h3>
          <div class="metric-value"><%= format_uptime(@uptime) %></div>
        </div>
      </div>
    </div>
    """
  end
  
  @impl true
  def handle_info(:tick, socket) do
    {:noreply, update(socket, :uptime, &(&1 + 1))}
  end
  
  @impl true
  def handle_info({:metrics_update, metrics}, socket) do
    {:noreply, assign(socket, metrics: metrics)}
  end
  
  @impl true
  def handle_info({:users_count, count}, socket) do
    {:noreply, assign(socket, users_online: count)}
  end
  
  defp get_metrics do
    %{
      cpu: :rand.uniform(100),
      memory_mb: :rand.uniform(1024),
      requests_per_sec: :rand.uniform(1000)
    }
  end
  
  defp format_uptime(seconds) do
    hours   = div(seconds, 3600)
    minutes = div(rem(seconds, 3600), 60)
    secs    = rem(seconds, 60)
    "#{pad(hours)}:#{pad(minutes)}:#{pad(secs)}"
  end
  
  defp pad(n), do: String.pad_leading("#{n}", 2, "0")
end
```

---

## Step 235: LiveComponents

```elixir
# Stateful component ที่ reusable
defmodule MyAppWeb.SearchComponent do
  use MyAppWeb, :live_component
  
  @impl true
  def render(assigns) do
    ~H"""
    <div id={@id} class="search-component">
      <input
        type="text"
        value={@query}
        placeholder="Search..."
        phx-change="search"
        phx-debounce="300"
        phx-target={@myself}
      />
      
      <%= if @loading do %>
        <div class="spinner">Loading...</div>
      <% end %>
      
      <ul class="results">
        <%= for result <- @results do %>
          <li phx-click="select" phx-value-id={result.id} phx-target={@myself}>
            <%= result.name %>
          </li>
        <% end %>
      </ul>
    </div>
    """
  end
  
  @impl true
  def mount(socket) do
    {:ok, assign(socket, query: "", results: [], loading: false)}
  end
  
  @impl true
  def handle_event("search", %{"value" => query}, socket) do
    if String.length(query) >= 2 do
      send(self(), {:search, query, socket.assigns.id})
      {:noreply, assign(socket, query: query, loading: true)}
    else
      {:noreply, assign(socket, query: query, results: [], loading: false)}
    end
  end
  
  @impl true
  def handle_event("select", %{"id" => id}, socket) do
    send(self(), {:selected, id})
    {:noreply, assign(socket, results: [], query: "")}
  end
  
  @impl true
  def update(%{results: results}, socket) do
    {:ok, assign(socket, results: results, loading: false)}
  end
  
  @impl true
  def update(assigns, socket) do
    {:ok, assign(socket, assigns)}
  end
end

# Parent LiveView ใช้ component
defmodule MyAppWeb.ProductSearchLive do
  use MyAppWeb, :live_view
  
  @impl true
  def mount(_params, _session, socket) do
    {:ok, assign(socket, selected_product: nil)}
  end
  
  @impl true
  def render(assigns) do
    ~H"""
    <div>
      <.live_component
        module={MyAppWeb.SearchComponent}
        id="product-search"
      />
      
      <%= if @selected_product do %>
        <div class="selected">
          Selected: <%= @selected_product.name %>
        </div>
      <% end %>
    </div>
    """
  end
  
  @impl true
  def handle_info({:search, query, component_id}, socket) do
    results = MyApp.Products.search(query)
    send_update(MyAppWeb.SearchComponent, id: component_id, results: results)
    {:noreply, socket}
  end
  
  @impl true
  def handle_info({:selected, product_id}, socket) do
    product = MyApp.Products.get_product!(product_id)
    {:noreply, assign(socket, selected_product: product)}
  end
end
```

---

## Step 236: Streams (Efficient Large Lists)

```elixir
defmodule MyAppWeb.MessagesLive do
  use MyAppWeb, :live_view
  
  @impl true
  def mount(_params, _session, socket) do
    messages = load_messages()
    
    socket = socket
      |> stream(:messages, messages)
      |> assign(form: to_form(%{"text" => ""}))
    
    {:ok, socket}
  end
  
  @impl true
  def render(assigns) do
    ~H"""
    <div class="chat-container">
      <ul id="messages" phx-update="stream">
        <li :for={{dom_id, message} <- @streams.messages} id={dom_id}>
          <strong><%= message.user %>:</strong> <%= message.text %>
          <button phx-click="delete" phx-value-id={message.id}>×</button>
        </li>
      </ul>
      
      <.form for={@form} phx-submit="send">
        <input type="text" name="text" placeholder="Type message..." />
        <button type="submit">Send</button>
      </.form>
    </div>
    """
  end
  
  @impl true
  def handle_event("send", %{"text" => text}, socket) do
    message = %{id: System.unique_integer(), user: "Me", text: text}
    
    {:noreply,
      socket
      |> stream_insert(:messages, message)
      |> assign(form: to_form(%{"text" => ""}))}
  end
  
  @impl true
  def handle_event("delete", %{"id" => id}, socket) do
    message = %{id: String.to_integer(id)}
    {:noreply, stream_delete(socket, :messages, message)}
  end
  
  defp load_messages do
    Enum.map(1..20, fn i ->
      %{id: i, user: "User#{i}", text: "Message #{i}"}
    end)
  end
end
```

---

## Step 237: JavaScript Hooks

```elixir
# หน้า LiveView มี JS Hooks สำหรับ client-side behavior

# assets/js/app.js
let Hooks = {}

Hooks.Chart = {
  mounted() {
    this.chart = new Chart(this.el, {
      type: 'line',
      data: JSON.parse(this.el.dataset.chartData)
    })
    
    // Listen for updates from server
    this.handleEvent("update-chart", ({data}) => {
      this.chart.data.datasets[0].data = data
      this.chart.update()
    })
  },
  
  destroyed() {
    this.chart.destroy()
  }
}

Hooks.InfiniteScroll = {
  mounted() {
    this.observer = new IntersectionObserver((entries) => {
      const entry = entries[0]
      if (entry.isIntersecting) {
        this.pushEvent("load-more", {})
      }
    })
    
    this.observer.observe(this.el)
  }
}

# LiveView side
defmodule MyAppWeb.ChartLive do
  use MyAppWeb, :live_view
  
  @impl true
  def render(assigns) do
    ~H"""
    <div>
      <canvas id="my-chart"
        phx-hook="Chart"
        data-chart-data={Jason.encode!(@chart_data)}>
      </canvas>
    </div>
    """
  end
  
  # Push JS event to client
  def update_chart(socket, new_data) do
    push_event(socket, "update-chart", %{data: new_data})
  end
end
```

---

## Step 238: PubSub กับ LiveView

```elixir
defmodule MyAppWeb.ChatLive do
  use MyAppWeb, :live_view
  
  alias MyApp.Chat
  alias Phoenix.PubSub
  
  @impl true
  def mount(%{"room_id" => room_id}, session, socket) do
    user_id = session["user_id"]
    
    if connected?(socket) do
      PubSub.subscribe(MyApp.PubSub, "room:#{room_id}")
      PubSub.broadcast(MyApp.PubSub, "room:#{room_id}", {:user_joined, user_id})
    end
    
    {:ok, assign(socket,
      room_id: room_id,
      messages: Chat.get_messages(room_id),
      users: Chat.get_users(room_id),
      current_user: user_id
    )}
  end
  
  @impl true
  def render(assigns) do
    ~H"""
    <div class="chat-room">
      <aside class="users">
        <h3>Online Users (<%= length(@users) %>)</h3>
        <%= for user <- @users do %>
          <div class="user"><%= user %></div>
        <% end %>
      </aside>
      
      <main class="messages">
        <div id="messages" phx-update="stream">
          <div :for={{id, msg} <- @streams.messages} id={id} class="message">
            <strong><%= msg.user %>:</strong> <%= msg.text %>
            <small><%= msg.timestamp %></small>
          </div>
        </div>
        
        <form phx-submit="send_message">
          <input type="text" name="text" placeholder="Message..." autocomplete="off"/>
          <button type="submit">Send</button>
        </form>
      </main>
    </div>
    """
  end
  
  @impl true
  def handle_event("send_message", %{"text" => text}, socket) do
    msg = %{
      id: System.unique_integer([:positive]),
      user: socket.assigns.current_user,
      text: text,
      timestamp: DateTime.utc_now()
    }
    
    PubSub.broadcast(MyApp.PubSub, "room:#{socket.assigns.room_id}", {:new_message, msg})
    {:noreply, socket}
  end
  
  @impl true
  def handle_info({:new_message, message}, socket) do
    {:noreply, stream_insert(socket, :messages, message)}
  end
  
  @impl true
  def handle_info({:user_joined, user_id}, socket) do
    {:noreply, update(socket, :users, fn users -> Enum.uniq([user_id | users]) end)}
  end
  
  @impl true
  def handle_info({:user_left, user_id}, socket) do
    {:noreply, update(socket, :users, fn users -> List.delete(users, user_id) end)}
  end
end
```

---

## Step 239: LiveView Testing

```elixir
defmodule MyAppWeb.CounterLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest
  
  test "renders counter" do
    {:ok, view, html} = live(build_conn(), "/counter")
    
    assert html =~ "Counter: 0"
  end
  
  test "increments counter" do
    {:ok, view, _html} = live(build_conn(), "/counter")
    
    view |> element("button", "+1") |> render_click()
    
    assert render(view) =~ "Counter: 1"
  end
  
  test "form validation" do
    {:ok, view, _html} = live(build_conn(), "/users/new")
    
    view
    |> form("#user-form", user: %{email: "invalid"})
    |> render_change()
    
    assert render(view) =~ "must have the @ sign"
  end
  
  test "submits form successfully" do
    {:ok, view, _html} = live(build_conn(), "/users/new")
    
    {:ok, conn} = view
      |> form("#user-form", user: %{
           name: "Alice",
           email: "alice@example.com",
           password: "secret123"
         })
      |> render_submit()
      |> follow_redirect(build_conn())
    
    assert conn.status == 200
    assert get_flash(conn, :info) == "User created!"
  end
end
```

---

## Step 240: Complete LiveView App - Todo List

```elixir
defmodule MyAppWeb.TodoLive do
  use MyAppWeb, :live_view
  
  @impl true
  def mount(_params, _session, socket) do
    {:ok,
      socket
      |> stream(:todos, [])
      |> assign(
           filter: :all,
           new_todo: "",
           editing_id: nil
         )}
  end
  
  @impl true
  def render(assigns) do
    ~H"""
    <div class="todo-app">
      <h1>Todo List</h1>
      
      <form phx-submit="add">
        <input
          type="text"
          name="title"
          value={@new_todo}
          placeholder="What needs to be done?"
          phx-change="update_new"
        />
        <button type="submit">Add</button>
      </form>
      
      <div class="filters">
        <button phx-click="filter" phx-value-type="all"
                class={if @filter == :all, do: "active"}>All</button>
        <button phx-click="filter" phx-value-type="active"
                class={if @filter == :active, do: "active"}>Active</button>
        <button phx-click="filter" phx-value-type="completed"
                class={if @filter == :completed, do: "active"}>Completed</button>
      </div>
      
      <ul id="todos" phx-update="stream">
        <li :for={{dom_id, todo} <- @streams.todos}
            id={dom_id}
            class={if todo.done, do: "done"}>
          <input
            type="checkbox"
            checked={todo.done}
            phx-click="toggle"
            phx-value-id={todo.id}
          />
          
          <%= if @editing_id == todo.id do %>
            <form phx-submit="save_edit">
              <input type="text" name="title" value={todo.title}
                     phx-blur="cancel_edit" autofocus />
              <input type="hidden" name="id" value={todo.id} />
            </form>
          <% else %>
            <span phx-dblclick="start_edit" phx-value-id={todo.id}>
              <%= todo.title %>
            </span>
          <% end %>
          
          <button phx-click="delete" phx-value-id={todo.id}>×</button>
        </li>
      </ul>
      
      <div class="todo-count">
        <%= active_count(@streams.todos) %> items left
      </div>
      
      <button phx-click="clear_completed">Clear Completed</button>
    </div>
    """
  end
  
  @impl true
  def handle_event("add", %{"title" => title}, socket) do
    if String.trim(title) != "" do
      todo = %{id: System.unique_integer([:positive]), title: title, done: false}
      {:noreply, socket |> stream_insert(:todos, todo) |> assign(new_todo: "")}
    else
      {:noreply, socket}
    end
  end
  
  def handle_event("update_new", %{"value" => value}, socket) do
    {:noreply, assign(socket, new_todo: value)}
  end
  
  def handle_event("toggle", %{"id" => id}, socket) do
    id = String.to_integer(id)
    # In real app, update database
    {:noreply, socket}  # simplified
  end
  
  def handle_event("delete", %{"id" => id}, socket) do
    id = String.to_integer(id)
    todo = %{id: id}  # only need id for delete
    {:noreply, stream_delete(socket, :todos, todo)}
  end
  
  def handle_event("start_edit", %{"id" => id}, socket) do
    {:noreply, assign(socket, editing_id: String.to_integer(id))}
  end
  
  def handle_event("save_edit", %{"id" => id, "title" => title}, socket) do
    # Update todo title
    {:noreply, assign(socket, editing_id: nil)}
  end
  
  def handle_event("cancel_edit", _params, socket) do
    {:noreply, assign(socket, editing_id: nil)}
  end
  
  def handle_event("filter", %{"type" => type}, socket) do
    {:noreply, assign(socket, filter: String.to_atom(type))}
  end
  
  def handle_event("clear_completed", _params, socket) do
    # Remove completed todos
    {:noreply, socket}
  end
  
  defp active_count(todos) do
    todos
    |> Enum.count(fn {_, todo} -> !todo.done end)
  end
end
```

---

## สรุป Part 22

✅ **Step 231** - LiveView concept  
✅ **Step 232** - Basic LiveView + events  
✅ **Step 233** - Forms + validation  
✅ **Step 234** - Real-time updates  
✅ **Step 235** - LiveComponents  
✅ **Step 236** - Streams  
✅ **Step 237** - JavaScript Hooks  
✅ **Step 238** - PubSub integration  
✅ **Step 239** - Testing  
✅ **Step 240** - Complete Todo app  

➡️ [Part 23: Phoenix Channels](./part-23-channels.md)
