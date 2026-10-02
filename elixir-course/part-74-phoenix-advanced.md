# Part 74: Advanced Phoenix Patterns (Steps 801-820)

## Step 801: Phoenix Contexts Deep Dive

```elixir
# Phoenix contexts are the boundary layer between web and business logic
# They group related functionality and expose a clean public API

defmodule MyApp.Accounts do
  # Public API
  def register_user(attrs), do: UserRegistration.call(attrs)
  def authenticate(email, password), do: Authentication.call(email, password)
  def get_user(id), do: Repo.get(User, id)
  def update_user(user, attrs), do: User.update_changeset(user, attrs) |> Repo.update()

  # Internal modules (not called directly by web layer)
  defmodule UserRegistration do
    def call(attrs) do
      Ecto.Multi.new()
      |> Ecto.Multi.insert(:user, User.registration_changeset(%User{}, attrs))
      |> Ecto.Multi.run(:send_welcome, fn _, %{user: user} ->
        MyApp.Mailer.send_welcome(user)
        {:ok, :sent}
      end)
      |> MyApp.Repo.transaction()
      |> case do
        {:ok, %{user: user}} -> {:ok, user}
        {:error, :user, changeset, _} -> {:error, changeset}
        {:error, _, reason, _} -> {:error, reason}
      end
    end
  end
end
```

---

## Step 802: Plug Pipeline Composition

```elixir
defmodule MyAppWeb.Plugs.LoadResource do
  import Plug.Conn
  import Phoenix.Controller

  def init(opts), do: opts

  def call(conn, schema: schema) do
    id = conn.params["id"]

    case MyApp.Repo.get(schema, id) do
      nil ->
        conn
        |> put_status(404)
        |> put_view(MyAppWeb.ErrorHTML)
        |> render(:"404")
        |> halt()

      resource ->
        assign(conn, :resource, resource)
    end
  end
end

defmodule MyAppWeb.Plugs.Authorize do
  import Plug.Conn
  import Phoenix.Controller

  def init(opts), do: opts

  def call(conn, action: action) do
    user     = conn.assigns.current_user
    resource = conn.assigns.resource

    if MyApp.Policy.can?(user, action, resource) do
      conn
    else
      conn
      |> put_status(403)
      |> json(%{error: "Forbidden"})
      |> halt()
    end
  end
end

# Controller usage
defmodule MyAppWeb.PostController do
  use MyAppWeb, :controller

  plug MyAppWeb.Plugs.LoadResource, schema: MyApp.Post
  plug MyAppWeb.Plugs.Authorize, action: :edit when action in [:edit, :update]

  def edit(conn, _params) do
    post = conn.assigns.resource
    changeset = MyApp.Posts.change_post(post)
    render(conn, :edit, post: post, changeset: changeset)
  end
end
```

---

## Step 803: Custom Error Handling

```elixir
defmodule MyAppWeb.FallbackController do
  use Phoenix.Controller

  def call(conn, {:error, :not_found}) do
    conn
    |> put_status(:not_found)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"404")
  end

  def call(conn, {:error, :unauthorized}) do
    conn
    |> put_status(:forbidden)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"403")
  end

  def call(conn, {:error, %Ecto.Changeset{} = changeset}) do
    conn
    |> put_status(:unprocessable_entity)
    |> put_view(json: MyAppWeb.ChangesetJSON)
    |> render(:error, changeset: changeset)
  end

  def call(conn, {:error, reason}) do
    conn
    |> put_status(:internal_server_error)
    |> json(%{error: inspect(reason)})
  end
end

defmodule MyAppWeb.ErrorJSON do
  def render("404.json", _assigns),  do: %{error: "Not found"}
  def render("403.json", _assigns),  do: %{error: "Forbidden"}
  def render("500.json", _assigns),  do: %{error: "Internal server error"}
end

defmodule MyAppWeb.ChangesetJSON do
  def render("error.json", %{changeset: changeset}) do
    %{errors: translate_errors(changeset)}
  end

  defp translate_errors(changeset) do
    Ecto.Changeset.traverse_errors(changeset, fn {msg, opts} ->
      Regex.replace(~r"%{(\w+)}", msg, fn _, key ->
        opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
      end)
    end)
  end
end
```

---

## Step 804: Phoenix LiveView Hooks

```javascript
// assets/js/hooks/index.js
const Hooks = {}

// Auto-resize textarea
Hooks.AutoResize = {
  mounted() {
    this.el.addEventListener("input", () => {
      this.el.style.height = "auto"
      this.el.style.height = this.el.scrollHeight + "px"
    })
  }
}

// Infinite scroll
Hooks.InfiniteScroll = {
  mounted() {
    this.observer = new IntersectionObserver(entries => {
      const entry = entries[0]
      if (entry.isIntersecting) {
        this.pushEvent("load_more", {})
      }
    })
    this.observer.observe(this.el)
  },
  destroyed() {
    this.observer.disconnect()
  }
}

// Copy to clipboard
Hooks.CopyToClipboard = {
  mounted() {
    this.el.addEventListener("click", () => {
      const text = this.el.dataset.value
      navigator.clipboard.writeText(text).then(() => {
        this.pushEvent("copied", {value: text})
      })
    })
  }
}

// Chart.js integration
Hooks.Chart = {
  mounted() {
    const data = JSON.parse(this.el.dataset.chartData)
    this.chart = new Chart(this.el, {
      type:    data.type,
      data:    data.data,
      options: data.options
    })
  },
  updated() {
    const data = JSON.parse(this.el.dataset.chartData)
    this.chart.data = data.data
    this.chart.update()
  },
  destroyed() {
    this.chart.destroy()
  }
}

export default Hooks
```

---

## Step 805: LiveView Components

```elixir
defmodule MyAppWeb.Components.Modal do
  use Phoenix.Component

  attr :id,    :string,  required: true
  attr :show,  :boolean, default: false
  attr :on_cancel, :any, default: nil
  slot :inner_block, required: true
  slot :title

  def modal(assigns) do
    ~H"""
    <div
      id={@id}
      phx-mounted={@show && show_modal(@id)}
      phx-remove={hide_modal(@id)}
      class="relative z-50 hidden"
    >
      <div class="fixed inset-0 bg-zinc-400/25 backdrop-blur-sm" aria-hidden="true" />
      <div class="fixed inset-0 overflow-y-auto" role="dialog">
        <div class="flex min-h-full items-center justify-center p-4">
          <div class="max-w-2xl w-full bg-white rounded-2xl shadow-xl p-8">
            <div :if={@title != []} class="text-xl font-semibold mb-4">
              <%= render_slot(@title) %>
            </div>
            <button
              :if={@on_cancel}
              phx-click={@on_cancel}
              class="absolute top-4 right-4"
            >
              &times;
            </button>
            <%= render_slot(@inner_block) %>
          </div>
        </div>
      </div>
    </div>
    """
  end

  defp show_modal(id) do
    JS.show(to: "##{id}")
    |> JS.add_class("overflow-hidden", to: "body")
  end

  defp hide_modal(id) do
    JS.hide(to: "##{id}")
    |> JS.remove_class("overflow-hidden", to: "body")
  end
end
```

---

## Step 806: Optimistic UI Updates

```elixir
defmodule MyAppWeb.TodoLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok, assign(socket, :todos, MyApp.Todos.list())}
  end

  # Optimistic: update UI immediately, then sync
  def handle_event("toggle", %{"id" => id}, socket) do
    # Immediately update the local UI
    todos = Enum.map(socket.assigns.todos, fn todo ->
      if to_string(todo.id) == id do
        %{todo | completed: !todo.completed}
      else
        todo
      end
    end)

    socket = assign(socket, :todos, todos)

    # Async server update
    Task.start(fn ->
      case MyApp.Todos.toggle(id) do
        {:ok, _} -> :ok
        {:error, _} ->
          # Revert on failure
          send(self(), {:revert_todo, id})
      end
    end)

    {:noreply, socket}
  end

  def handle_info({:revert_todo, id}, socket) do
    todos = Enum.map(socket.assigns.todos, fn todo ->
      if to_string(todo.id) == id do
        %{todo | completed: !todo.completed}
      else
        todo
      end
    end)
    {:noreply, assign(socket, :todos, todos)}
  end
end
```

---

## Step 807: LiveView Streams

```elixir
defmodule MyAppWeb.FeedLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "feed")
    end

    # Stream: efficient rendering of large lists
    socket = stream(socket, :posts, MyApp.Posts.recent(limit: 20))

    {:ok, socket}
  end

  def handle_info({:new_post, post}, socket) do
    # Prepend new post to stream (no full list re-render)
    {:noreply, stream_insert(socket, :posts, post, at: 0)}
  end

  def handle_info({:delete_post, post}, socket) do
    {:noreply, stream_delete(socket, :posts, post)}
  end

  def handle_event("delete", %{"id" => id}, socket) do
    post = MyApp.Posts.get!(id)
    {:ok, _} = MyApp.Posts.delete(post)
    {:noreply, stream_delete(socket, :posts, post)}
  end

  def render(assigns) do
    ~H"""
    <div id="posts" phx-update="stream">
      <div :for={{dom_id, post} <- @streams.posts} id={dom_id} class="post">
        <h3><%= post.title %></h3>
        <p><%= post.body %></p>
        <button phx-click="delete" phx-value-id={post.id}>Delete</button>
      </div>
    </div>
    """
  end
end
```

---

## Step 808: Multi-Step Forms

```elixir
defmodule MyAppWeb.OnboardingLive do
  use MyAppWeb, :live_view

  @steps [:profile, :preferences, :confirmation]

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(:current_step, :profile)
     |> assign(:form_data, %{})
     |> assign(:changeset, profile_changeset(%{}))}
  end

  def handle_event("next", %{"user" => attrs}, socket) do
    current = socket.assigns.current_step

    case validate_step(current, attrs) do
      {:ok, data} ->
        form_data  = Map.merge(socket.assigns.form_data, data)
        next_step  = next_step(current)
        changeset  = changeset_for(next_step, form_data)

        {:noreply,
         socket
         |> assign(:current_step, next_step)
         |> assign(:form_data, form_data)
         |> assign(:changeset, changeset)}

      {:error, changeset} ->
        {:noreply, assign(socket, :changeset, changeset)}
    end
  end

  def handle_event("submit", _attrs, socket) do
    case MyApp.Accounts.create_user(socket.assigns.form_data) do
      {:ok, user} ->
        {:noreply,
         socket
         |> put_flash(:info, "Account created!")
         |> redirect(to: ~p"/dashboard")}
      {:error, changeset} ->
        {:noreply, assign(socket, :changeset, changeset)}
    end
  end

  defp next_step(:profile),      do: :preferences
  defp next_step(:preferences),  do: :confirmation
  defp next_step(:confirmation), do: :done
end
```

---

## Step 809: Phoenix API Versioning

```elixir
defmodule MyAppWeb.Router do
  use MyAppWeb, :router

  # Version via URL path
  scope "/api/v1", MyAppWeb.V1, as: :api_v1 do
    pipe_through [:api, :auth_required]
    resources "/users", UserController
  end

  scope "/api/v2", MyAppWeb.V2, as: :api_v2 do
    pipe_through [:api, :auth_required]
    resources "/users", UserController
  end
end

# Version via Accept header
defmodule MyAppWeb.Plugs.APIVersion do
  import Plug.Conn

  @default_version "v1"
  @supported_versions ~w(v1 v2)

  def init(opts), do: opts

  def call(conn, _opts) do
    version = get_version(conn)
    assign(conn, :api_version, version)
  end

  defp get_version(conn) do
    # Check Accept: application/vnd.myapp.v2+json
    case get_req_header(conn, "accept") do
      [accept | _] ->
        case Regex.run(~r/vnd\.myapp\.(\w+)\+json/, accept) do
          [_, version] when version in @supported_versions -> version
          _ -> @default_version
        end
      [] -> @default_version
    end
  end
end
```

---

## Step 810: Server-Sent Events (SSE)

```elixir
defmodule MyAppWeb.SSEController do
  use MyAppWeb, :controller

  def stream(conn, _params) do
    user = conn.assigns.current_user

    conn
    |> put_resp_content_type("text/event-stream")
    |> put_resp_header("cache-control", "no-cache")
    |> put_resp_header("connection", "keep-alive")
    |> send_chunked(200)
    |> stream_events(user)
  end

  defp stream_events(conn, user) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "user:#{user.id}")
    
    # Send heartbeat to keep connection alive
    :timer.send_interval(30_000, :heartbeat)

    receive_loop(conn)
  end

  defp receive_loop(conn) do
    receive do
      {:notification, data} ->
        event = format_event("notification", data)
        case chunk(conn, event) do
          {:ok, conn}    -> receive_loop(conn)
          {:error, _}    -> conn  # Client disconnected
        end

      :heartbeat ->
        case chunk(conn, ": heartbeat\n\n") do
          {:ok, conn}    -> receive_loop(conn)
          {:error, _}    -> conn
        end
    after
      60_000 -> conn  # Timeout after 1 minute of inactivity
    end
  end

  defp format_event(event, data) do
    "event: #{event}\ndata: #{Jason.encode!(data)}\n\n"
  end
end
```

---

## สรุป Part 74

✅ **Step 801** - Phoenix contexts deep dive  
✅ **Step 802** - Plug pipeline composition  
✅ **Step 803** - Custom error handling  
✅ **Step 804** - LiveView hooks  
✅ **Step 805** - LiveView components  
✅ **Step 806** - Optimistic UI updates  
✅ **Step 807** - LiveView streams  
✅ **Step 808** - Multi-step forms  
✅ **Step 809** - API versioning  
✅ **Step 810** - Server-Sent Events  

➡️ [Part 75: Distributed Systems Patterns](./part-75-distributed.md)
