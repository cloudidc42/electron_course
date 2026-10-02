# Part 66: Rate Limiting & Throttling (Steps 721-740)

## Step 721: Token Bucket Algorithm

```elixir
defmodule MyApp.RateLimit.TokenBucket do
  use GenServer

  # Token bucket: refill at constant rate, burst allowed up to capacity

  def start_link(opts) do
    name     = Keyword.fetch!(opts, :name)
    capacity = Keyword.get(opts, :capacity, 100)
    refill_rate = Keyword.get(opts, :refill_rate, 10)  # tokens per second

    GenServer.start_link(__MODULE__,
      %{capacity: capacity, refill_rate: refill_rate, buckets: %{}},
      name: name
    )
  end

  def allow?(server, key, cost \\ 1) do
    GenServer.call(server, {:allow?, key, cost})
  end

  def init(state) do
    schedule_refill()
    {:ok, state}
  end

  def handle_call({:allow?, key, cost}, _from, state) do
    bucket = Map.get(state.buckets, key, state.capacity)

    if bucket >= cost do
      new_bucket = bucket - cost
      new_buckets = Map.put(state.buckets, key, new_bucket)
      {:reply, {:ok, new_bucket}, %{state | buckets: new_buckets}}
    else
      {:reply, {:error, :rate_limited}, state}
    end
  end

  def handle_info(:refill, state) do
    new_buckets = Map.new(state.buckets, fn {key, tokens} ->
      {key, min(tokens + state.refill_rate, state.capacity)}
    end)

    # Clean up full buckets to save memory
    cleaned = Enum.reject(new_buckets, fn {_, t} -> t == state.capacity end)
              |> Map.new()

    schedule_refill()
    {:noreply, %{state | buckets: cleaned}}
  end

  defp schedule_refill, do: Process.send_after(self(), :refill, 1000)
end
```

---

## Step 722: Sliding Window Algorithm

```elixir
defmodule MyApp.RateLimit.SlidingWindow do
  # Sliding window using Redis sorted sets
  # More accurate than fixed window

  def allow?(key, limit, window_seconds) do
    now     = System.os_time(:millisecond)
    min_val = now - window_seconds * 1000
    pipe_key = "rate:#{key}"

    {:ok, results} = Redix.pipeline(:redix, [
      # Remove old entries
      ["ZREMRANGEBYSCORE", pipe_key, "-inf", min_val],
      # Count current entries
      ["ZCARD", pipe_key],
      # Add current request
      ["ZADD", pipe_key, now, "#{now}:#{:rand.uniform(1_000_000)}"],
      # Set expiry
      ["EXPIRE", pipe_key, window_seconds + 1]
    ])

    count = Enum.at(results, 1)

    if count < limit do
      {:ok, %{remaining: limit - count - 1, reset_at: now + window_seconds * 1000}}
    else
      {:error, %{
        retry_after: window_seconds,
        limit: limit,
        reset_at: min_val + window_seconds * 1000
      }}
    end
  end
end
```

---

## Step 723: Fixed Window Algorithm

```elixir
defmodule MyApp.RateLimit.FixedWindow do
  def allow?(key, limit, window_seconds) do
    window_key = div(System.os_time(:second), window_seconds)
    redis_key  = "rate:#{key}:#{window_key}"

    {:ok, count} = Redix.command(:redix, ["INCR", redis_key])
    
    if count == 1 do
      Redix.command(:redix, ["EXPIRE", redis_key, window_seconds])
    end

    if count <= limit do
      {:ok, %{count: count, remaining: limit - count}}
    else
      {:error, %{retry_after: window_seconds - rem(System.os_time(:second), window_seconds)}}
    end
  end
end
```

---

## Step 724: Plug-Based Rate Limiter

```elixir
defmodule MyApp.Plugs.RateLimit do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, opts) do
    limit          = Keyword.get(opts, :limit, 100)
    window_seconds = Keyword.get(opts, :window, 60)
    key_fn         = Keyword.get(opts, :key, &default_key/1)
    
    key = key_fn.(conn)

    case MyApp.RateLimit.SlidingWindow.allow?(key, limit, window_seconds) do
      {:ok, %{remaining: remaining, reset_at: reset_at}} ->
        conn
        |> put_resp_header("x-ratelimit-limit",     Integer.to_string(limit))
        |> put_resp_header("x-ratelimit-remaining",  Integer.to_string(remaining))
        |> put_resp_header("x-ratelimit-reset",      Integer.to_string(reset_at))

      {:error, %{retry_after: retry}} ->
        conn
        |> put_resp_header("retry-after",           Integer.to_string(retry))
        |> put_resp_header("x-ratelimit-limit",     Integer.to_string(limit))
        |> put_resp_header("x-ratelimit-remaining", "0")
        |> send_resp(429, Jason.encode!(%{error: "Too many requests", retry_after: retry}))
        |> halt()
    end
  end

  defp default_key(conn) do
    case conn.assigns[:current_user] do
      nil  -> conn.remote_ip |> :inet.ntoa() |> to_string()
      user -> "user:#{user.id}"
    end
  end
end

# Usage in router
pipeline :api do
  plug :accepts, ["json"]
  plug MyApp.Plugs.RateLimit, limit: 100, window: 60
end

# Stricter for auth endpoints
pipeline :auth do
  plug MyApp.Plugs.RateLimit,
    limit:  5,
    window: 300,
    key:    fn conn -> "auth:#{Map.get(conn.params, "email", conn.remote_ip)}" end
end
```

---

## Step 725: Adaptive Rate Limiting

```elixir
defmodule MyApp.RateLimit.Adaptive do
  # Adjust limits based on server load

  def allow?(user_id, base_limit) do
    load_factor = current_load_factor()
    adjusted_limit = round(base_limit * load_factor)

    MyApp.RateLimit.SlidingWindow.allow?("user:#{user_id}", adjusted_limit, 60)
  end

  defp current_load_factor do
    scheduler_util = :scheduler.utilization(1)
    avg_util       = scheduler_util |> Enum.map(& &1.total) |> average()

    # High load = lower limits
    cond do
      avg_util > 0.9  -> 0.3   # 70% reduction
      avg_util > 0.7  -> 0.6   # 40% reduction
      avg_util > 0.5  -> 0.8   # 20% reduction
      true            -> 1.0   # normal
    end
  end

  defp average([]),  do: 0.0
  defp average(list), do: Enum.sum(list) / length(list)
end
```

---

## Step 726: Per-Endpoint Rate Limits

```elixir
defmodule MyAppWeb.Router do
  pipeline :api, do: plug :accepts, ["json"]

  # Define rate limit rules per endpoint
  @rate_limits %{
    login:         {5,   300},   # 5 per 5 minutes
    register:      {3,   3600},  # 3 per hour
    send_email:    {10,  3600},  # 10 per hour
    ai_request:    {20,  60},    # 20 per minute
    api_general:   {100, 60}     # 100 per minute
  }

  scope "/api/v1" do
    pipe_through :api

    post "/users/login" do
      plug MyApp.Plugs.RateLimit,
        limit:  5,
        window: 300,
        key:    &("login:" <> (Map.get(&1.params, "email", "") |> String.downcase()))
    end

    post "/ai/chat" do
      plug MyApp.Plugs.RateLimit, limit: 20, window: 60
      plug MyApp.Plugs.RateLimit,
        limit:  500,
        window: 86400,
        key:    fn conn -> "ai_daily:#{conn.assigns[:current_user].id}" end
    end
  end
end
```

---

## Step 727: Circuit Breaker

```elixir
defmodule MyApp.CircuitBreaker do
  use GenServer

  @failure_threshold 5
  @reset_timeout     30_000  # 30 seconds
  @half_open_max     3

  defstruct state: :closed,
            failures: 0,
            success_in_half_open: 0,
            opened_at: nil

  def start_link(opts) do
    name = Keyword.fetch!(opts, :name)
    GenServer.start_link(__MODULE__, %__MODULE__{}, name: name)
  end

  def call(server, fun) do
    case GenServer.call(server, :request) do
      :open ->
        {:error, :circuit_open}
      :allow ->
        result = try_execute(fun)
        GenServer.cast(server, {:result, result})
        result
    end
  end

  def handle_call(:request, _from, %{state: :open} = state) do
    if should_try_reset?(state) do
      {:reply, :allow, %{state | state: :half_open}}
    else
      {:reply, :open, state}
    end
  end

  def handle_call(:request, _from, state), do: {:reply, :allow, state}

  def handle_cast({:result, {:ok, _}}, %{state: :half_open} = state) do
    new_successes = state.success_in_half_open + 1
    if new_successes >= @half_open_max do
      {:noreply, %__MODULE__{state: :closed}}
    else
      {:noreply, %{state | success_in_half_open: new_successes}}
    end
  end

  def handle_cast({:result, {:ok, _}}, state) do
    {:noreply, %{state | failures: 0}}
  end

  def handle_cast({:result, {:error, _}}, state) do
    new_failures = state.failures + 1
    if new_failures >= @failure_threshold do
      Logger.warning("Circuit breaker #{inspect(self())} opened!")
      {:noreply, %__MODULE__{state: :open, opened_at: System.monotonic_time(:millisecond)}}
    else
      {:noreply, %{state | failures: new_failures}}
    end
  end

  defp should_try_reset?(%{opened_at: opened_at}) do
    elapsed = System.monotonic_time(:millisecond) - opened_at
    elapsed >= @reset_timeout
  end

  defp try_execute(fun) do
    try do
      result = fun.()
      if match?({:ok, _}, result), do: result, else: {:error, :unexpected}
    rescue
      e -> {:error, e}
    catch
      :exit, reason -> {:error, {:exit, reason}}
    end
  end
end
```

---

## Step 728: Bulkhead Pattern

```elixir
defmodule MyApp.Bulkhead do
  # Limit concurrent calls to isolate failures
  use GenServer

  def start_link(opts) do
    name = Keyword.fetch!(opts, :name)
    max  = Keyword.get(opts, :max_concurrent, 10)
    GenServer.start_link(__MODULE__, %{max: max, current: 0, queue: :queue.new()}, name: name)
  end

  def execute(server, fun, timeout \\ 5000) do
    case GenServer.call(server, :acquire, timeout) do
      :ok ->
        try do
          fun.()
        after
          GenServer.cast(server, :release)
        end
      :full ->
        {:error, :bulkhead_full}
    end
  end

  def handle_call(:acquire, from, %{current: c, max: max} = state) when c < max do
    {:reply, :ok, %{state | current: c + 1}}
  end

  def handle_call(:acquire, from, %{current: c, max: max} = state) when c >= max do
    # Queue the request
    new_queue = :queue.in(from, state.queue)
    {:noreply, %{state | queue: new_queue}}
  end

  def handle_cast(:release, state) do
    case :queue.out(state.queue) do
      {{:value, from}, new_queue} ->
        GenServer.reply(from, :ok)
        {:noreply, %{state | queue: new_queue}}
      {:empty, _} ->
        {:noreply, %{state | current: state.current - 1}}
    end
  end
end
```

---

## Step 729: Distributed Rate Limiting

```elixir
defmodule MyApp.RateLimit.Distributed do
  # Use Redis for distributed rate limiting across nodes

  def allow?(key, limit, window \\ 60) do
    lua_script = """
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    local count = redis.call('GET', key)
    if count == false then count = 0 else count = tonumber(count) end

    if count >= limit then
      return {0, count, redis.call('TTL', key)}
    end

    local new_count = redis.call('INCR', key)
    if new_count == 1 then
      redis.call('EXPIRE', key, window)
    end

    return {1, new_count - 1, window}
    """

    {:ok, [allowed, remaining, reset]} = Redix.command(:redix, [
      "EVAL", lua_script, "1", "rate:#{key}",
      Integer.to_string(limit),
      Integer.to_string(window),
      Integer.to_string(System.os_time(:second))
    ])

    if allowed == 1 do
      {:ok, %{remaining: remaining, reset_after: reset}}
    else
      {:error, %{retry_after: reset, limit: limit}}
    end
  end

  # Rate limit with multiple tiers (per-second AND per-minute)
  def allow_tiered?(key, limits) do
    results = Enum.map(limits, fn {window, limit} ->
      allow?("#{key}:#{window}", limit, window)
    end)

    Enum.find(results, {:ok, %{}}, &match?({:error, _}, &1))
  end
end
```

---

## Step 730: Rate Limit Testing

```elixir
defmodule MyApp.RateLimitTest do
  use MyAppWeb.ConnCase, async: false

  describe "API rate limiting" do
    test "allows requests under limit" do
      user = insert(:user)
      headers = api_auth_headers(user)

      for _ <- 1..10 do
        conn = get(build_conn(), "/api/v1/products", [], headers)
        assert conn.status == 200
        assert get_resp_header(conn, "x-ratelimit-remaining") |> List.first() != "0"
      end
    end

    test "blocks requests over limit" do
      user = insert(:user)
      headers = api_auth_headers(user)

      # Exhaust the limit
      for _ <- 1..100 do
        get(build_conn(), "/api/v1/products", [], headers)
      end

      conn = get(build_conn(), "/api/v1/products", [], headers)
      assert conn.status == 429
      assert get_resp_header(conn, "retry-after") |> List.first() != nil
    end

    test "different users have independent limits" do
      user1 = insert(:user)
      user2 = insert(:user)

      # Exhaust user1's limit
      for _ <- 1..100 do
        get(build_conn(), "/api/v1/products", [], api_auth_headers(user1))
      end

      # user2 should still work
      conn = get(build_conn(), "/api/v1/products", [], api_auth_headers(user2))
      assert conn.status == 200
    end
  end
end
```

---

## สรุป Part 66

✅ **Step 721** - Token bucket algorithm  
✅ **Step 722** - Sliding window  
✅ **Step 723** - Fixed window  
✅ **Step 724** - Plug-based rate limiter  
✅ **Step 725** - Adaptive rate limiting  
✅ **Step 726** - Per-endpoint limits  
✅ **Step 727** - Circuit breaker  
✅ **Step 728** - Bulkhead pattern  
✅ **Step 729** - Distributed rate limiting  
✅ **Step 730** - Testing rate limits  

➡️ [Part 67: IoT & Hardware Integration](./part-67-iot.md)
