# Part 51: LiveView Advanced (Steps 561-580)

## Step 561: LiveView Streams

```elixir
defmodule MyAppWeb.ProductLive.Index do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    products = Products.list()
    {:ok, stream(socket, :products, products)}
  end

  def handle_event("delete", %{"id" => id}, socket) do
    product = Products.get!(id)
    {:ok, _} = Products.delete(product)
    {:noreply, stream_delete(socket, :products, product)}
  end

  def handle_event("load_more", _, socket) do
    cursor  = socket.assigns.cursor
    {more, new_cursor} = Products.list_after(cursor, limit: 20)
    {:noreply, socket |> stream_insert(:products, more) |> assign(:cursor, new_cursor)}
  end

  # PubSub: add new product from another user
  def handle_info({:product_created, product}, socket) do
    {:noreply, stream_insert(socket, :products, product, at: 0)}
  end

  # Template
  # <div id="products" phx-update="stream">
  #   <div :for={{id, product} <- @streams.products} id={id}>
  #     <%= product.name %>
  #   </div>
  # </div>
end
```

---

## Step 562: LiveView Multi-Step Forms

```elixir
defmodule MyAppWeb.CheckoutLive do
  use MyAppWeb, :live_view

  @steps [:cart, :shipping, :payment, :confirm]

  def mount(_params, _session, socket) do
    {:ok, assign(socket,
      step:     :cart,
      form:     to_form(Checkout.changeset(%Checkout{}, %{})),
      order:    %{}
    )}
  end

  def handle_event("next", %{"checkout" => params}, socket) do
    changeset = Checkout.changeset(%Checkout{}, params)
    
    if changeset.valid? do
      next_step = next_step(socket.assigns.step)
      {:noreply, socket
        |> assign(:step, next_step)
        |> assign(:order, Map.merge(socket.assigns.order, params))}
    else
      {:noreply, assign(socket, :form, to_form(changeset))}
    end
  end

  def handle_event("back", _, socket) do
    prev = prev_step(socket.assigns.step)
    {:noreply, assign(socket, :step, prev)}
  end

  def handle_event("submit", _, socket) do
    case Orders.create(socket.assigns.order) do
      {:ok, order} ->
        {:noreply, push_navigate(socket, to: ~p"/orders/#{order.id}")}
      {:error, cs} ->
        {:noreply, assign(socket, :form, to_form(cs))}
    end
  end

  defp next_step(:cart),     do: :shipping
  defp next_step(:shipping), do: :payment
  defp next_step(:payment),  do: :confirm
  defp prev_step(:confirm),  do: :payment
  defp prev_step(:payment),  do: :shipping
  defp prev_step(:shipping), do: :cart
end
```

---

## Step 563: File Uploads

```elixir
defmodule MyAppWeb.UploadLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok, allow_upload(socket, :photos,
      accept:         ~w(.jpg .jpeg .png .webp),
      max_entries:    5,
      max_file_size:  10 * 1_024 * 1_024,  # 10MB
      chunk_size:     64 * 1_024
    )}
  end

  def handle_event("validate", _params, socket) do
    {:noreply, socket}
  end

  def handle_event("save", _params, socket) do
    urls =
      consume_uploaded_entries(socket, :photos, fn %{path: path}, entry ->
        dest = "uploads/#{entry.uuid}#{Path.extname(entry.client_name)}"
        
        # Resize image before saving
        Mogrify.open(path)
        |> Mogrify.resize_to_limit("1200x1200")
        |> Mogrify.save(path: dest)
        
        # Upload to S3
        {:ok, url} = S3.upload(dest, "my-bucket/photos/#{entry.uuid}")
        {:ok, url}
      end)
    
    {:noreply, assign(socket, :uploaded_urls, urls)}
  end

  # Template
  # <.live_file_input upload={@uploads.photos} />
  # <div :for={entry <- @uploads.photos.entries}>
  #   <.live_img_preview entry={entry} />
  #   <progress value={entry.progress} max="100" />
  #   <button phx-click="cancel-upload" phx-value-ref={entry.ref}>x</button>
  #   <p :for={err <- upload_errors(@uploads.photos, entry)}><%= error_to_string(err) %></p>
  # </div>
end
```

---

## Step 564: LiveView Hooks (JS Interop)

```javascript
// assets/js/hooks.js
const Hooks = {}

Hooks.Chart = {
  mounted() {
    const ctx = this.el.querySelector('canvas').getContext('2d')
    this.chart = new Chart(ctx, {
      type: 'line',
      data: JSON.parse(this.el.dataset.chartData),
      options: { responsive: true }
    })

    // Listen for LiveView push events
    this.handleEvent("update_chart", ({ data }) => {
      this.chart.data = data
      this.chart.update()
    })
  },
  
  updated() {
    const newData = JSON.parse(this.el.dataset.chartData)
    this.chart.data = newData
    this.chart.update()
  },
  
  destroyed() {
    this.chart.destroy()
  }
}

Hooks.InfiniteScroll = {
  mounted() {
    this.observer = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) {
        this.pushEvent("load_more", {})
      }
    }, { threshold: 0.5 })
    
    this.observer.observe(this.el)
  },
  
  destroyed() {
    this.observer.disconnect()
  }
}

Hooks.AutoSave = {
  mounted() {
    this.timeout = null
    this.el.addEventListener('input', () => {
      clearTimeout(this.timeout)
      this.timeout = setTimeout(() => {
        this.pushEvent("auto_save", { content: this.el.value })
      }, 500)
    })
  }
}

// app.js
import { LiveSocket } from "phoenix_liveview"
const liveSocket = new LiveSocket("/live", Socket, { hooks: Hooks, params: {_csrf_token: csrfToken} })
liveSocket.connect()
```

```elixir
defmodule MyAppWeb.ChartLive do
  use MyAppWeb, :live_view

  def mount(_, _, socket) do
    if connected?(socket) do
      :timer.send_interval(5_000, :update)
    end
    {:ok, assign(socket, :data, get_chart_data())}
  end

  def handle_info(:update, socket) do
    data = get_chart_data()
    {:noreply, push_event(socket, "update_chart", %{data: data})}
  end
end
```

---

## Step 565: LiveComponent

```elixir
defmodule MyAppWeb.CartComponent do
  use MyAppWeb, :live_component

  def update(%{user_id: user_id} = assigns, socket) do
    cart = Cart.get_for_user(user_id)
    {:ok, assign(socket, assigns) |> assign(:cart, cart)}
  end

  def handle_event("add_item", %{"product_id" => pid}, socket) do
    Cart.add_item(socket.assigns.user_id, pid, 1)
    cart = Cart.get_for_user(socket.assigns.user_id)
    {:noreply, assign(socket, :cart, cart)}
  end

  def handle_event("remove_item", %{"product_id" => pid}, socket) do
    Cart.remove_item(socket.assigns.user_id, pid)
    cart = Cart.get_for_user(socket.assigns.user_id)
    {:noreply, assign(socket, :cart, cart)}
  end

  def render(assigns) do
    ~H"""
    <div id={"cart-#{@id}"}>
      <h3>Cart (<%= @cart.item_count %> items)</h3>
      <div :for={item <- @cart.items}>
        <span><%= item.name %></span>
        <span><%= item.quantity %></span>
        <button phx-click="remove_item" phx-value-product_id={item.product_id} phx-target={@myself}>
          Remove
        </button>
      </div>
      <p>Total: <%= Money.to_string(@cart.total) %></p>
    </div>
    """
  end
end

# Use in parent LiveView:
# <.live_component module={MyAppWeb.CartComponent} id="main-cart" user_id={@current_user.id} />
```

---

## Step 566: LiveView Navigation

```elixir
defmodule MyAppWeb.ProductLive.Show do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok, socket}
  end

  # handle_params is called on mount AND on navigation
  def handle_params(%{"id" => id}, _url, socket) do
    product = Products.get!(id)
    {:noreply, assign(socket, :product, product)}
  end

  # patch: update URL without full remount
  def handle_event("tab", %{"tab" => tab}, socket) do
    {:noreply, push_patch(socket, to: ~p"/products/#{socket.assigns.product.id}?tab=#{tab}")}
  end

  # navigate: full LiveView transition (new mount)
  def handle_event("go_next", _, socket) do
    {:noreply, push_navigate(socket, to: ~p"/products/#{socket.assigns.product.id + 1}")}
  end

  # redirect: full page load
  def handle_event("go_home", _, socket) do
    {:noreply, redirect(socket, to: "/")}
  end
end

# Router:
# live "/products/:id", ProductLive.Show, :show
# live "/products/:id/edit", ProductLive.Show, :edit
# (action differentiates via socket.assigns.live_action)
```

---

## Step 567: Phoenix.Component (Function Components)

```elixir
defmodule MyAppWeb.CoreComponents do
  use Phoenix.Component
  import Phoenix.HTML

  # Basic function component
  attr :type,  :string,  default: "button"
  attr :class, :string,  default: ""
  attr :rest,  :global
  slot :inner_block, required: true

  def button(assigns) do
    ~H"""
    <button type={@type} class={["btn", @class]} {@rest}>
      <%= render_slot(@inner_block) %>
    </button>
    """
  end

  # Card component with slots
  attr :title, :string, required: true
  slot :header
  slot :inner_block, required: true
  slot :footer

  def card(assigns) do
    ~H"""
    <div class="card">
      <div :if={@header != []} class="card-header">
        <%= render_slot(@header) %>
      </div>
      <div class="card-body">
        <h3><%= @title %></h3>
        <%= render_slot(@inner_block) %>
      </div>
      <div :if={@footer != []} class="card-footer">
        <%= render_slot(@footer) %>
      </div>
    </div>
    """
  end

  # Usage:
  # <.card title="Hello">
  #   <:header>Custom Header</:header>
  #   Main content
  #   <:footer>Footer</:footer>
  # </.card>

  # Table component
  attr :rows, :list,   required: true
  attr :row_id, :any,  default: nil
  slot :col, required: true do
    attr :label, :string
  end

  def table(assigns) do
    ~H"""
    <table>
      <thead>
        <tr>
          <th :for={col <- @col}><%= col[:label] %></th>
        </tr>
      </thead>
      <tbody>
        <tr :for={row <- @rows} id={@row_id && @row_id.(row)}>
          <td :for={col <- @col}><%= render_slot(col, row) %></td>
        </tr>
      </tbody>
    </table>
    """
  end
end
```

---

## Step 568: LiveView Authentication

```elixir
defmodule MyAppWeb.UserAuth do
  import Plug.Conn
  import Phoenix.Controller
  use MyAppWeb, :verified_routes

  def log_in_user(conn, user, params \\ %{}) do
    token        = Accounts.generate_user_session_token(user)
    user_return_to = get_session(conn, :user_return_to)
    
    conn
    |> put_session(:user_token, token)
    |> put_session(:live_socket_id, "users_sessions:#{Base.url_encode64(token)}")
    |> maybe_write_remember_me_cookie(token, params)
    |> redirect(to: user_return_to || signed_in_path(conn))
  end

  def log_out_user(conn) do
    user_token = get_session(conn, :user_token)
    user_token && Accounts.delete_user_session_token(user_token)
    
    if live_socket_id = get_session(conn, :live_socket_id) do
      MyAppWeb.Endpoint.broadcast(live_socket_id, "disconnect", %{})
    end
    
    conn
    |> renew_session()
    |> redirect(to: ~p"/")
  end

  def on_mount(:require_authenticated_user, _params, session, socket) do
    socket = mount_current_user(socket, session)

    if socket.assigns.current_user do
      {:cont, socket}
    else
      {:halt, Phoenix.LiveView.redirect(socket, to: ~p"/users/log_in")}
    end
  end

  defp mount_current_user(socket, session) do
    Phoenix.Component.assign_new(socket, :current_user, fn ->
      if user_token = session["user_token"] do
        Accounts.get_user_by_session_token(user_token)
      end
    end)
  end

  defp signed_in_path(_conn), do: ~p"/"

  defp maybe_write_remember_me_cookie(conn, token, %{"remember_me" => "true"}) do
    put_resp_cookie(conn, "remember_me", token, sign: true, max_age: 60 * 60 * 24 * 60, same_site: "Lax")
  end
  defp maybe_write_remember_me_cookie(conn, _token, _params), do: conn
end
```

---

## Step 569: LiveView Testing

```elixir
defmodule MyAppWeb.ProductLiveTest do
  use MyAppWeb.ConnCase, async: true

  import Phoenix.LiveViewTest
  import MyApp.Factory

  setup :register_and_log_in_user

  test "lists products", %{conn: conn} do
    product = insert(:product, name: "Test Widget")
    
    {:ok, view, html} = live(conn, ~p"/products")
    
    assert html =~ "Test Widget"
    assert has_element?(view, "#product-#{product.id}")
  end

  test "filters products by category", %{conn: conn} do
    insert(:product, category: "books")
    insert(:product, category: "electronics")
    
    {:ok, view, _html} = live(conn, ~p"/products")
    
    html = view |> element("#filter-form") |> render_change(%{category: "books"})
    
    assert html =~ "books"
    refute html =~ "electronics"
  end

  test "creates product", %{conn: conn} do
    {:ok, view, _html} = live(conn, ~p"/products/new")
    
    html = view
      |> form("#product-form", product: %{name: "New Product", price: "29.99"})
      |> render_submit()
    
    assert_redirect(view, ~p"/products")
  end

  test "real-time update from PubSub", %{conn: conn} do
    {:ok, view, _html} = live(conn, ~p"/products")
    
    product = insert(:product, name: "Live Product")
    MyAppWeb.Endpoint.broadcast("products", "product_created", %{product: product})
    
    assert has_element?(view, "#product-#{product.id}", "Live Product")
  end
end
```

---

## Step 570: LiveView Performance

```elixir
defmodule MyAppWeb.OptimizedLive do
  use MyAppWeb, :live_view

  # 1. assign_async - non-blocking data loading
  def mount(_params, _session, socket) do
    {:ok, socket
      |> assign(:products_loading, true)
      |> start_async(:load_products, fn -> Products.list_featured() end)
    }
  end

  def handle_async(:load_products, {:ok, products}, socket) do
    {:noreply, assign(socket, products: products, products_loading: false)}
  end

  def handle_async(:load_products, {:exit, _reason}, socket) do
    {:noreply, assign(socket, products: [], products_loading: false)}
  end

  # 2. Use streams for large lists (DOM patching vs full re-render)
  # stream/3 instead of assign/3 for lists

  # 3. Debounce user input
  # <input phx-change="search" phx-debounce="300" />

  # 4. Throttle high-frequency updates
  def handle_info(:tick, socket) do
    now = System.monotonic_time(:millisecond)
    last = socket.assigns[:last_update] || 0
    
    if now - last >= 100 do  # max 10 updates/sec
      data = get_realtime_data()
      {:noreply, assign(socket, data: data, last_update: now)}
    else
      {:noreply, socket}
    end
  end

  # 5. Temporary assigns (free memory after render)
  def mount(_, _, socket) do
    {:ok, socket, temporary_assigns: [messages: []]}
  end
end
```

---

## สรุป Part 51

✅ **Step 561** - LiveView Streams  
✅ **Step 562** - Multi-step forms  
✅ **Step 563** - File uploads  
✅ **Step 564** - JS Hooks  
✅ **Step 565** - LiveComponent  
✅ **Step 566** - Navigation  
✅ **Step 567** - Function components  
✅ **Step 568** - Authentication  
✅ **Step 569** - Testing  
✅ **Step 570** - Performance  

➡️ [Part 52: GraphQL/Absinthe Production](./part-52-graphql.md)
