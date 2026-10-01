# Part 19: Error Handling (Steps 201-220)

## Step 201: Error Philosophy ใน Elixir

```
Elixir มี 2 strategies:
1. "Let it Crash" - ให้ Supervisor handle
2. Defensive - {:ok, value} | {:error, reason}

เมื่อไหร่ใช้อะไร?
- Let it Crash: programming errors, unexpected states
- Defensive: expected failures (validation, network, user input)
```

---

## Step 202: {:ok, value} | {:error, reason} Pattern

```elixir
defmodule UserService do
  def get_user(id) when is_integer(id) and id > 0 do
    case Database.find_user(id) do
      nil  -> {:error, :not_found}
      user -> {:ok, user}
    end
  end
  
  def get_user(_), do: {:error, :invalid_id}
  
  def create_user(attrs) do
    with {:ok, validated} <- validate_attrs(attrs),
         {:ok, user}      <- Database.insert_user(validated) do
      {:ok, user}
    else
      {:error, :validation, errors} -> {:error, {:validation, errors}}
      {:error, :duplicate_email}    -> {:error, :email_taken}
      {:error, reason}              -> {:error, reason}
    end
  end
  
  defp validate_attrs(attrs) do
    errors = []
    errors = if blank?(attrs[:name]),  do: [{:name, "required"} | errors],  else: errors
    errors = if blank?(attrs[:email]), do: [{:email, "required"} | errors], else: errors
    errors = if !valid_email?(attrs[:email]), do: [{:email, "invalid"} | errors], else: errors
    
    if errors == [], do: {:ok, attrs}, else: {:error, :validation, errors}
  end
  
  defp blank?(nil), do: true
  defp blank?(""),  do: true
  defp blank?(_),   do: false
  
  defp valid_email?(email) when is_binary(email) do
    email =~ ~r/^[^\s]+@[^\s]+\.[^\s]+$/
  end
  defp valid_email?(_), do: false
end
```

---

## Step 203: with Expression สำหรับ Happy Path

```elixir
defmodule OrderService do
  def place_order(user_id, items, payment_info) do
    with {:ok, user}    <- get_user(user_id),
         {:ok, items}   <- validate_items(items),
         {:ok, total}   <- calculate_total(items),
         :ok            <- check_inventory(items),
         {:ok, payment} <- process_payment(payment_info, total),
         {:ok, order}   <- create_order(user, items, payment) do
      notify_user(user, order)
      {:ok, order}
    else
      {:error, :user_not_found}  -> {:error, "User not found"}
      {:error, :out_of_stock}    -> {:error, "Some items are out of stock"}
      {:error, :payment_failed}  -> {:error, "Payment processing failed"}
      {:error, reason}           -> {:error, "Order failed: #{inspect(reason)}"}
    end
  end
  
  defp get_user(id) do
    # Simulated
    if id > 0, do: {:ok, %{id: id, email: "user@example.com"}}, else: {:error, :user_not_found}
  end
  
  defp validate_items([]), do: {:error, :empty_order}
  defp validate_items(items) do
    valid = Enum.all?(items, fn item -> item[:quantity] > 0 && item[:product_id] end)
    if valid, do: {:ok, items}, else: {:error, :invalid_items}
  end
  
  defp calculate_total(items) do
    total = Enum.reduce(items, 0, fn item, acc ->
      acc + item[:price] * item[:quantity]
    end)
    {:ok, total}
  end
  
  defp check_inventory(items) do
    # Check stock
    :ok
  end
  
  defp process_payment(_info, _total), do: {:ok, %{transaction_id: "txn_123"}}
  
  defp create_order(user, items, payment) do
    order = %{
      id: "ord_#{:rand.uniform(9999)}",
      user_id: user.id,
      items: items,
      payment: payment,
      created_at: DateTime.utc_now()
    }
    {:ok, order}
  end
  
  defp notify_user(user, order) do
    IO.puts("Email sent to #{user.email}: Order #{order.id} confirmed!")
  end
end
```

---

## Step 204: try/rescue/catch/after

```elixir
defmodule SafeOperations do
  def safe_parse_int(str) do
    try do
      {:ok, String.to_integer(str)}
    rescue
      ArgumentError -> {:error, :not_a_number}
    end
  end
  
  def safe_file_read(path) do
    try do
      content = File.read!(path)  # raises on error
      {:ok, content}
    rescue
      File.Error -> {:error, :file_not_found}
      e          -> {:error, Exception.message(e)}
    after
      IO.puts("File operation completed")
    end
  end
  
  def with_db_transaction(func) do
    try do
      DB.begin_transaction()
      result = func.()
      DB.commit()
      {:ok, result}
    rescue
      e ->
        DB.rollback()
        {:error, Exception.message(e)}
    catch
      :exit, reason ->
        DB.rollback()
        {:error, {:exit, reason}}
      :throw, value ->
        DB.rollback()
        {:error, {:throw, value}}
    after
      DB.release_connection()
    end
  end
end

# catch vs rescue:
# rescue - catches exceptions (raised with raise/1)
# catch  - catches exits (:exit) and throws (:throw)

defmodule ThrowCatch do
  def find_first(list, pred) do
    try do
      Enum.each(list, fn item ->
        if pred.(item), do: throw({:found, item})
      end)
      nil
    catch
      {:found, item} -> item
    end
  end
end

ThrowCatch.find_first([1, 2, 3, 4, 5], &(&1 > 3))  # 4
```

---

## Step 205: Custom Exceptions

```elixir
defmodule AppError do
  defexception [:message, :code, :details]
  
  def exception(opts) do
    code    = Keyword.get(opts, :code, :internal_error)
    message = Keyword.get(opts, :message, "An error occurred")
    details = Keyword.get(opts, :details, %{})
    
    %__MODULE__{
      message: message,
      code: code,
      details: details
    }
  end
end

defmodule ValidationError do
  defexception [:message, :errors]
  
  def exception(errors) when is_list(errors) do
    message = "Validation failed: #{Enum.map_join(errors, ", ", fn {field, msg} -> "#{field} #{msg}" end)}"
    %__MODULE__{message: message, errors: errors}
  end
end

defmodule AuthError do
  defexception message: "Authentication failed"
end

# Usage
raise AppError, code: :not_found, message: "User not found", details: %{id: 123}
raise ValidationError, [name: "is required", email: "is invalid"]

# rescue specific exceptions
try do
  raise ValidationError, [email: "is invalid"]
rescue
  ValidationError = e ->
    IO.puts("Validation errors: #{inspect(e.errors)}")
  AppError = e ->
    IO.puts("App error #{e.code}: #{e.message}")
  e ->
    IO.puts("Unknown error: #{inspect(e)}")
end
```

---

## Step 206: Error Handling ใน Pipelines

```elixir
defmodule Pipeline do
  # Railway-oriented programming
  
  def ok(value), do: {:ok, value}
  def error(reason), do: {:error, reason}
  
  # >>= (bind) operator simulation
  def then_ok({:ok, value}, func), do: func.(value)
  def then_ok({:error, _} = err, _func), do: err
  
  # Map over success
  def map_ok({:ok, value}, func), do: {:ok, func.(value)}
  def map_ok({:error, _} = err, _func), do: err
  
  # Tap for side effects
  def tap_ok({:ok, value} = result, func) do
    func.(value)
    result
  end
  def tap_ok({:error, _} = err, _func), do: err
end

defmodule UserPipeline do
  import Pipeline
  
  def process(raw_input) do
    ok(raw_input)
    |> then_ok(&parse_input/1)
    |> then_ok(&validate/1)
    |> then_ok(&normalize/1)
    |> tap_ok(&log_success/1)
    |> then_ok(&save/1)
  end
  
  defp parse_input(input) when is_map(input), do: {:ok, input}
  defp parse_input(_), do: {:error, :invalid_input}
  
  defp validate(%{email: email} = input) do
    if email =~ ~r/@/ do
      {:ok, input}
    else
      {:error, :invalid_email}
    end
  end
  defp validate(_), do: {:error, :missing_email}
  
  defp normalize(input) do
    {:ok, Map.update(input, :email, "", &String.downcase/1)}
  end
  
  defp log_success(data) do
    IO.puts("Processing: #{data.email}")
  end
  
  defp save(data) do
    {:ok, Map.put(data, :id, "usr_123")}
  end
end

UserPipeline.process(%{email: "Alice@EXAMPLE.COM"})
# {:ok, %{email: "alice@example.com", id: "usr_123"}}

UserPipeline.process(%{email: "invalid"})
# {:error, :invalid_email}
```

---

## Step 207: Logger กับ Error Reporting

```elixir
defmodule ErrorReporter do
  require Logger
  
  def report(error, context \\ %{}) do
    severity = determine_severity(error)
    
    metadata = [
      error: inspect(error),
      context: inspect(context),
      timestamp: DateTime.utc_now() |> DateTime.to_iso8601()
    ]
    
    case severity do
      :critical ->
        Logger.critical("CRITICAL ERROR: #{format_error(error)}", metadata)
        notify_on_call(error, context)
      
      :error ->
        Logger.error("ERROR: #{format_error(error)}", metadata)
      
      :warning ->
        Logger.warning("WARNING: #{format_error(error)}", metadata)
    end
  end
  
  defp determine_severity(%{code: :database_error}), do: :critical
  defp determine_severity(%{code: :payment_failed}),  do: :critical
  defp determine_severity(%{code: :not_found}),        do: :warning
  defp determine_severity(_),                          do: :error
  
  defp format_error(%{message: msg}), do: msg
  defp format_error(error), do: inspect(error)
  
  defp notify_on_call(error, context) do
    # PagerDuty, Slack alert, etc.
    IO.puts("🚨 ON-CALL ALERT: #{inspect(error)}")
  end
end

# Logger configuration
# config/config.exs
config :logger,
  level: :debug,
  backends: [:console],
  utc_log: true

config :logger, :console,
  format: "[$level] $time $metadata[$message]\n",
  metadata: [:request_id, :user_id],
  level: :info
```

---

## Step 208: Supervisor กับ Error Recovery

```elixir
defmodule ResilienceDemo do
  use Supervisor
  
  def start_link(_) do
    Supervisor.start_link(__MODULE__, [], name: __MODULE__)
  end
  
  @impl true
  def init([]) do
    children = [
      # ถ้า crash มากกว่า 3 ครั้งใน 10 วินาที → supervisor crash
      {WorkerA, []},
      {WorkerB, [retry_on: [:network_error, :timeout]]}
    ]
    
    Supervisor.init(children, strategy: :one_for_one, max_restarts: 3, max_seconds: 10)
  end
end

defmodule WorkerB do
  use GenServer
  
  @max_retries 3
  @retry_delay 1000
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @impl true
  def init(opts) do
    {:ok, %{opts: opts, retries: 0}}
  end
  
  @impl true
  def handle_call({:do_work, data}, from, state) do
    case try_work(data, state) do
      {:ok, result} ->
        {:reply, {:ok, result}, %{state | retries: 0}}
      
      {:error, reason} when reason in state.opts[:retry_on] ->
        if state.retries < @max_retries do
          Process.send_after(self(), {:retry, data, from}, @retry_delay)
          {:noreply, %{state | retries: state.retries + 1}}
        else
          {:reply, {:error, :max_retries_exceeded}, %{state | retries: 0}}
        end
      
      {:error, reason} ->
        {:reply, {:error, reason}, state}
    end
  end
  
  @impl true
  def handle_info({:retry, data, from}, state) do
    case try_work(data, state) do
      {:ok, result} ->
        GenServer.reply(from, {:ok, result})
        {:noreply, %{state | retries: 0}}
      {:error, _reason} ->
        GenServer.reply(from, {:error, :failed_after_retries})
        {:noreply, %{state | retries: 0}}
    end
  end
  
  defp try_work(_data, _state) do
    # Actual work
    {:ok, :result}
  end
end
```

---

## Step 209: Telemetry สำหรับ Error Monitoring

```elixir
defmodule MyApp.Telemetry do
  def setup do
    events = [
      [:my_app, :request, :start],
      [:my_app, :request, :stop],
      [:my_app, :request, :exception],
      [:my_app, :job, :success],
      [:my_app, :job, :failure]
    ]
    
    :telemetry.attach_many(
      "my-app-telemetry",
      events,
      &handle_event/4,
      nil
    )
  end
  
  def handle_event([:my_app, :request, :exception], measurements, metadata, _config) do
    %{kind: kind, reason: reason, stacktrace: stacktrace} = metadata
    
    ErrorReporter.report(%{
      type: :request_exception,
      kind: kind,
      reason: reason,
      path: metadata[:path],
      duration_ms: measurements[:duration] / 1_000_000
    })
  end
  
  def handle_event([:my_app, :job, :failure], measurements, metadata, _config) do
    require Logger
    Logger.error("Job failed",
      job_id: metadata.job_id,
      attempt: metadata.attempt,
      error: inspect(metadata.error)
    )
  end
  
  def handle_event(_event, _measurements, _metadata, _config), do: :ok
end

# Emit telemetry events
defmodule MyService do
  def process_request(conn) do
    start_time = System.monotonic_time()
    
    :telemetry.span([:my_app, :request], %{path: conn.request_path}, fn ->
      result = do_process(conn)
      {result, %{}}
    end)
  end
  
  defp do_process(_conn), do: :ok
end
```

---

## Step 210: Complete Error Handling System

```elixir
defmodule ErrorHandling do
  @moduledoc """
  Complete error handling pattern สำหรับ production app
  """
  
  # Error types
  defmodule Errors do
    defexception [:type, :message, :code, :details, :stacktrace_info]
    
    # Predefined error types
    def not_found(resource, id) do
      %__MODULE__{
        type: :not_found,
        code: 404,
        message: "#{resource} with id #{id} not found",
        details: %{resource: resource, id: id}
      }
    end
    
    def unauthorized(reason \\ "Not authorized") do
      %__MODULE__{type: :unauthorized, code: 401, message: reason, details: %{}}
    end
    
    def validation(errors) do
      %__MODULE__{
        type: :validation,
        code: 422,
        message: "Validation failed",
        details: %{errors: errors}
      }
    end
    
    def internal(message, details \\ %{}) do
      %__MODULE__{type: :internal, code: 500, message: message, details: details}
    end
  end
  
  # Error response formatter
  defmodule Formatter do
    def to_api_response(%Errors{type: type, code: code, message: msg, details: details}) do
      %{
        error: %{
          type: type,
          code: code,
          message: msg,
          details: details
        }
      }
    end
    
    def to_api_response({:error, :not_found}), do: to_api_response(Errors.not_found("Resource", "unknown"))
    def to_api_response({:error, :unauthorized}), do: to_api_response(Errors.unauthorized())
    def to_api_response({:error, reason}), do: to_api_response(Errors.internal(inspect(reason)))
  end
  
  # Result monad helpers
  defmodule Result do
    def ok(value), do: {:ok, value}
    def error(reason), do: {:error, reason}
    
    def map({:ok, value}, func) do
      try do
        {:ok, func.(value)}
      rescue
        e -> {:error, Errors.internal(Exception.message(e))}
      end
    end
    def map({:error, _} = err, _func), do: err
    
    def flat_map({:ok, value}, func) do
      try do
        func.(value)
      rescue
        e -> {:error, Errors.internal(Exception.message(e))}
      end
    end
    def flat_map({:error, _} = err, _func), do: err
    
    def unwrap!({:ok, value}), do: value
    def unwrap!({:error, reason}), do: raise(Errors, reason)
    
    def unwrap_or({:ok, value}, _default), do: value
    def unwrap_or({:error, _}, default), do: default
  end
end

# ใช้งาน
alias ErrorHandling.{Errors, Formatter, Result}

# Chain operations
result = Result.ok(%{user_id: 1, amount: 100})
  |> Result.flat_map(fn %{user_id: id} ->
       case Database.find_user(id) do
         nil  -> Result.error(Errors.not_found("User", id))
         user -> Result.ok(user)
       end
     end)
  |> Result.map(&authorize_payment/1)
  |> Result.flat_map(&process_payment/1)

case result do
  {:ok, payment}  -> IO.puts("Payment processed: #{payment.id}")
  {:error, error} ->
    response = Formatter.to_api_response(error)
    IO.inspect(response)
end
```

---

## สรุป Part 19

✅ **Step 201** - Error philosophy  
✅ **Step 202** - {:ok}/{:error} pattern  
✅ **Step 203** - with expression  
✅ **Step 204** - try/rescue/catch/after  
✅ **Step 205** - Custom exceptions  
✅ **Step 206** - Railway-oriented programming  
✅ **Step 207** - Logger + error reporting  
✅ **Step 208** - Supervisor + retry  
✅ **Step 209** - Telemetry  
✅ **Step 210** - Complete error handling system  

➡️ [Part 20: Documentation และ Typespecs](./part-20-docs-typespecs.md)
