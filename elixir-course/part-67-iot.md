# Part 67: IoT & Hardware Integration (Steps 731-750)

## Step 731: Nerves Framework

```
Nerves = Framework for building embedded Linux systems with Elixir

Architecture:
┌─────────────────────────────────────────────────────┐
│                  Nerves System                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Buildroot │  │  Erlang  │  │  BEAM            │  │
│  │  (Linux)   │  │  OTP     │  │  Application     │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                      │
│  Target hardware:                                    │
│  - Raspberry Pi (2/3/4/Zero)                        │
│  - BeagleBone Black                                  │
│  - GRiSP (bare-metal BEAM)                          │
│  - x86_64 VMs                                        │
└─────────────────────────────────────────────────────┘

mix nerves.new my_firmware
cd my_firmware
export MIX_TARGET=rpi4
mix deps.get
mix firmware
mix burn   # burn to SD card
mix upload # OTA update
```

---

## Step 732: GPIO Control

```elixir
# mix.exs: {:circuits_gpio, "~> 1.1"}

defmodule MyFirmware.LED do
  alias Circuits.GPIO

  def start_link(pin) do
    GenServer.start_link(__MODULE__, pin, name: __MODULE__)
  end

  def on,    do: GenServer.call(__MODULE__, :on)
  def off,   do: GenServer.call(__MODULE__, :off)
  def blink(ms \\ 500), do: GenServer.cast(__MODULE__, {:blink, ms})

  def init(pin) do
    {:ok, gpio} = GPIO.open(pin, :output)
    {:ok, %{gpio: gpio, pin: pin, state: :off}}
  end

  def handle_call(:on, _from, state) do
    GPIO.write(state.gpio, 1)
    {:reply, :ok, %{state | state: :on}}
  end

  def handle_call(:off, _from, state) do
    GPIO.write(state.gpio, 0)
    {:reply, :ok, %{state | state: :off}}
  end

  def handle_cast({:blink, interval}, state) do
    Process.send_after(self(), {:blink_toggle, interval}, interval)
    {:noreply, state}
  end

  def handle_info({:blink_toggle, interval}, state) do
    new_val = if state.state == :on, do: 0, else: 1
    GPIO.write(state.gpio, new_val)
    Process.send_after(self(), {:blink_toggle, interval}, interval)
    {:noreply, %{state | state: if(new_val == 1, do: :on, else: :off)}}
  end

  def terminate(_reason, state) do
    GPIO.close(state.gpio)
  end
end

# Button input with interrupt
defmodule MyFirmware.Button do
  alias Circuits.GPIO

  def start_link(pin) do
    GenServer.start_link(__MODULE__, pin)
  end

  def init(pin) do
    {:ok, gpio} = GPIO.open(pin, :input, pull_mode: :pullup)
    GPIO.set_interrupts(gpio, :both)
    {:ok, %{gpio: gpio, pin: pin}}
  end

  def handle_info({:circuits_gpio, pin, timestamp, value}, state) do
    case value do
      0 -> IO.puts("Button #{pin} pressed at #{timestamp}")
      1 -> IO.puts("Button #{pin} released")
    end
    {:noreply, state}
  end
end
```

---

## Step 733: I2C Sensors

```elixir
# mix.exs: {:circuits_i2c, "~> 1.0"}

defmodule MyFirmware.BME280 do
  # BME280 temperature/humidity/pressure sensor
  alias Circuits.I2C

  @addr 0x76  # I2C address

  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)

  def read, do: GenServer.call(__MODULE__, :read)

  def init(_) do
    {:ok, bus} = I2C.open("i2c-1")
    init_sensor(bus)
    {:ok, %{bus: bus}}
  end

  def handle_call(:read, _from, %{bus: bus} = state) do
    data = read_sensor(bus)
    {:reply, data, state}
  end

  defp init_sensor(bus) do
    # Set to normal mode, 16x oversampling
    I2C.write(bus, @addr, <<0xF3, 0x27>>)  # ctrl_meas register
    Process.sleep(100)
  end

  defp read_sensor(bus) do
    # Read raw data registers
    {:ok, data} = I2C.write_read(bus, @addr, <<0xF7>>, 8)
    
    <<p_msb, p_lsb, p_xlsb, t_msb, t_lsb, t_xlsb, h_msb, h_lsb>> = data
    
    raw_temp     = (t_msb <<< 12) ||| (t_lsb <<< 4) ||| (t_xlsb >>> 4)
    raw_pressure = (p_msb <<< 12) ||| (p_lsb <<< 4) ||| (p_xlsb >>> 4)
    raw_humidity = (h_msb <<< 8)  ||| h_lsb

    %{
      temperature: compensate_temperature(raw_temp),
      pressure:    compensate_pressure(raw_pressure),
      humidity:    compensate_humidity(raw_humidity)
    }
  end

  # Compensation formulas from BME280 datasheet
  defp compensate_temperature(raw) do
    # Simplified version
    raw / 5120.0 - 50.0
  end

  defp compensate_pressure(raw) do
    raw / 256.0
  end

  defp compensate_humidity(raw) do
    raw / 1024.0
  end
end
```

---

## Step 734: SPI Communication

```elixir
# mix.exs: {:circuits_spi, "~> 1.3"}

defmodule MyFirmware.MCP3008 do
  # MCP3008 - 8-channel 10-bit ADC (analog-to-digital converter)
  alias Circuits.SPI

  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)

  def read_channel(ch) when ch in 0..7 do
    GenServer.call(__MODULE__, {:read, ch})
  end

  def init(_) do
    {:ok, spi} = SPI.open("spidev0.0",
      mode:           0,
      bits_per_word:  8,
      speed_hz:       1_000_000
    )
    {:ok, %{spi: spi}}
  end

  def handle_call({:read, channel}, _from, %{spi: spi} = state) do
    # MCP3008 SPI protocol
    cmd    = <<0x01, (0x80 ||| (channel <<< 4)), 0x00>>
    {:ok, <<_, hi, lo>>} = SPI.transfer(spi, cmd)
    
    value = ((hi &&& 0x03) <<< 8) ||| lo  # 10-bit value (0-1023)
    voltage = value / 1023.0 * 3.3         # Convert to voltage

    {:reply, %{raw: value, voltage: voltage}, state}
  end
end
```

---

## Step 735: MQTT Client

```elixir
# mix.exs: {:tortoise311, "~> 0.11"}

defmodule MyApp.MQTT.Client do
  use Tortoise311.Handler

  def start_link(opts) do
    broker  = Keyword.get(opts, :broker, "mqtt://broker.hivemq.com")
    client_id = "elixir-#{System.unique_integer([:positive])}"

    Tortoise311.Supervisor.start_child(
      client_id: client_id,
      handler:   {__MODULE__, []},
      server:    {Tortoise311.Transport.Tcp,
                  host: URI.parse(broker).host,
                  port: URI.parse(broker).port || 1883},
      subscriptions: [
        {"sensors/#", 0},        # QoS 0
        {"commands/+/set", 1}    # QoS 1
      ]
    )
  end

  def publish(topic, payload, opts \\ []) do
    qos = Keyword.get(opts, :qos, 0)
    retain = Keyword.get(opts, :retain, false)
    
    Tortoise311.publish("my_client", topic, Jason.encode!(payload),
      qos: qos,
      retain: retain
    )
  end

  # Handler callbacks
  @impl true
  def handle_message(["sensors", device_id, "temperature"], payload, state) do
    data = Jason.decode!(payload)
    
    MyApp.IoT.record_telemetry(device_id, :temperature, data["value"])
    
    {:ok, state}
  end

  @impl true
  def handle_message(["commands", device_id, "set"], payload, state) do
    command = Jason.decode!(payload)
    MyApp.IoT.dispatch_command(device_id, command)
    {:ok, state}
  end

  @impl true
  def handle_message(topic, _payload, state) do
    Logger.debug("Unhandled MQTT message: #{inspect(topic)}")
    {:ok, state}
  end
end
```

---

## Step 736: Device Registry

```elixir
defmodule MyApp.IoT.DeviceRegistry do
  use GenServer

  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)

  def register(device_id, metadata \\ %{}) do
    GenServer.call(__MODULE__, {:register, device_id, metadata})
  end

  def update_status(device_id, status) do
    GenServer.cast(__MODULE__, {:update_status, device_id, status})
  end

  def list_online do
    GenServer.call(__MODULE__, :list_online)
  end

  def get(device_id) do
    GenServer.call(__MODULE__, {:get, device_id})
  end

  def init(_) do
    :ets.new(:devices, [:set, :public, :named_table, read_concurrency: true])
    {:ok, %{}}
  end

  def handle_call({:register, device_id, metadata}, _from, state) do
    device = %{
      id:          device_id,
      metadata:    metadata,
      status:      :online,
      last_seen:   DateTime.utc_now(),
      registered:  DateTime.utc_now()
    }
    :ets.insert(:devices, {device_id, device})
    
    Phoenix.PubSub.broadcast(MyApp.PubSub, "devices", {:device_online, device})
    {:reply, :ok, state}
  end

  def handle_call(:list_online, _from, state) do
    devices = :ets.select(:devices,
      [{{:"$1", %{status: :online}}, [], [:"$_"]}]
    )
    {:reply, devices, state}
  end

  def handle_cast({:update_status, device_id, status}, state) do
    case :ets.lookup(:devices, device_id) do
      [{_, device}] ->
        updated = %{device | status: status, last_seen: DateTime.utc_now()}
        :ets.insert(:devices, {device_id, updated})
        Phoenix.PubSub.broadcast(MyApp.PubSub, "devices", {:device_status, device_id, status})
      [] -> :ok
    end
    {:noreply, state}
  end
end
```

---

## Step 737: Telemetry Ingestion Pipeline

```elixir
defmodule MyApp.IoT.TelemetryPipeline do
  use Broadway

  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name:     __MODULE__,
      producer: [
        module: {BroadwayRabbitMQ.Producer,
          queue: "iot.telemetry",
          connection: [host: "localhost"],
          qos: [prefetch_count: 100]}
      ],
      processors: [
        default: [concurrency: 5]
      ],
      batchers: [
        timescaledb: [concurrency: 2, batch_size: 500, batch_timeout: 1000]
      ]
    )
  end

  @impl true
  def handle_message(:default, message, _context) do
    telemetry = Jason.decode!(message.data)
    
    if valid_telemetry?(telemetry) do
      message
      |> Broadway.Message.update_data(fn _ -> normalize(telemetry) end)
      |> Broadway.Message.put_batcher(:timescaledb)
    else
      Broadway.Message.failed(message, "invalid telemetry")
    end
  end

  @impl true
  def handle_batch(:timescaledb, messages, _batch_info, _context) do
    rows = Enum.map(messages, & &1.data)
    
    MyApp.Repo.insert_all("device_telemetry", rows,
      on_conflict: :nothing
    )
    
    messages
  end

  defp valid_telemetry?(%{"device_id" => _, "metric" => _, "value" => _, "timestamp" => _}), do: true
  defp valid_telemetry?(_), do: false

  defp normalize(t) do
    %{
      device_id:  t["device_id"],
      metric:     t["metric"],
      value:      t["value"],
      unit:       t["unit"],
      timestamp:  DateTime.from_unix!(t["timestamp"])
    }
  end
end
```

---

## Step 738: OTA (Over-the-Air) Updates

```elixir
defmodule MyFirmware.OTA do
  # Nerves OTA updates via NervesHub or custom server
  # mix.exs: {:nerves_hub_link, "~> 2.2"}

  def check_for_updates do
    current = Nerves.Runtime.KV.get("nerves_fw_uuid")
    
    case get_latest_firmware() do
      {:ok, %{uuid: uuid, url: url}} when uuid != current ->
        Logger.info("New firmware available: #{uuid}")
        apply_update(url)
      {:ok, _} ->
        Logger.info("Already on latest firmware")
        :up_to_date
      {:error, reason} ->
        {:error, reason}
    end
  end

  defp get_latest_firmware do
    Req.get("https://firmware.myapp.com/latest",
      headers: [{"x-device-id", Nerves.Runtime.serial_number()}]
    )
    |> case do
      {:ok, %{status: 200, body: body}} -> {:ok, body}
      _ -> {:error, :fetch_failed}
    end
  end

  defp apply_update(firmware_url) do
    Logger.info("Downloading firmware from #{firmware_url}")
    
    case Nerves.Runtime.Heart.connect_to_upgrade(firmware_url) do
      :ok ->
        Logger.info("Firmware applied, rebooting...")
        Nerves.Runtime.reboot()
      {:error, reason} ->
        Logger.error("Firmware update failed: #{inspect(reason)}")
        {:error, reason}
    end
  end
end
```

---

## Step 739: Device Dashboard LiveView

```elixir
defmodule MyAppWeb.DeviceDashboardLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "devices")
      Phoenix.PubSub.subscribe(MyApp.PubSub, "telemetry")
      :timer.send_interval(5_000, :refresh_stats)
    end

    devices = MyApp.IoT.DeviceRegistry.list_online()
    stats   = MyApp.IoT.stats()

    {:ok, assign(socket, devices: devices, stats: stats)}
  end

  def handle_info({:device_online, device}, socket) do
    {:noreply, update(socket, :devices, &[device | &1])}
  end

  def handle_info({:device_status, device_id, :offline}, socket) do
    devices = Enum.reject(socket.assigns.devices, & &1.id == device_id)
    {:noreply, assign(socket, :devices, devices)}
  end

  def handle_info({:telemetry, data}, socket) do
    # Update device stats
    {:noreply, push_event(socket, "telemetry", data)}
  end

  def handle_info(:refresh_stats, socket) do
    {:noreply, assign(socket, :stats, MyApp.IoT.stats())}
  end

  def render(assigns) do
    ~H"""
    <div class="iot-dashboard">
      <div class="stats-bar">
        <span>Online: <%= @stats.online_count %></span>
        <span>Total: <%= @stats.total_count %></span>
        <span>Alerts: <%= @stats.alert_count %></span>
      </div>

      <div class="device-grid">
        <div :for={device <- @devices} class="device-card">
          <h3><%= device.id %></h3>
          <p>Status: <span class={status_class(device.status)}><%= device.status %></span></p>
          <p>Last seen: <%= format_time(device.last_seen) %></p>
          <canvas
            id={"chart-#{device.id}"}
            phx-hook="DeviceChart"
            data-device-id={device.id}>
          </canvas>
        </div>
      </div>
    </div>
    """
  end

  defp status_class(:online),  do: "text-green-500"
  defp status_class(:offline), do: "text-red-500"
  defp status_class(_),        do: "text-yellow-500"
end
```

---

## Step 740: Alert & Threshold System

```elixir
defmodule MyApp.IoT.Alerts do
  use GenServer

  @check_interval 60_000  # check every minute

  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)

  def init(_) do
    schedule_check()
    {:ok, %{}}
  end

  def handle_info(:check_thresholds, state) do
    check_all_devices()
    schedule_check()
    {:noreply, state}
  end

  defp check_all_devices do
    thresholds = MyApp.IoT.Threshold.list_active()
    
    Enum.each(thresholds, fn threshold ->
      case evaluate_threshold(threshold) do
        {:breached, value} ->
          unless already_alerted?(threshold) do
            create_alert(threshold, value)
            notify(threshold, value)
          end
        :ok -> clear_alert(threshold)
      end
    end)
  end

  defp evaluate_threshold(%{device_id: did, metric: metric, operator: op, value: limit}) do
    recent = MyApp.IoT.latest_value(did, metric)
    
    breached = case op do
      ">"  -> recent > limit
      ">=" -> recent >= limit
      "<"  -> recent < limit
      "<=" -> recent <= limit
      "==" -> recent == limit
    end

    if breached, do: {:breached, recent}, else: :ok
  end

  defp notify(threshold, value) do
    MyApp.Workers.AlertWorker.new(%{
      device_id: threshold.device_id,
      metric:    threshold.metric,
      value:     value,
      limit:     threshold.value,
      operator:  threshold.operator
    })
    |> Oban.insert()
  end

  defp schedule_check, do: Process.send_after(self(), :check_thresholds, @check_interval)
  defp already_alerted?(_threshold), do: false  # Check in DB
  defp create_alert(_threshold, _value), do: :ok
  defp clear_alert(_threshold), do: :ok
end
```

---

## สรุป Part 67

✅ **Step 731** - Nerves framework  
✅ **Step 732** - GPIO control  
✅ **Step 733** - I2C sensors  
✅ **Step 734** - SPI communication  
✅ **Step 735** - MQTT client  
✅ **Step 736** - Device registry  
✅ **Step 737** - Telemetry pipeline  
✅ **Step 738** - OTA updates  
✅ **Step 739** - Device dashboard LiveView  
✅ **Step 740** - Alert & threshold system  

➡️ [Part 68: Advanced Deployment Strategies](./part-68-deployment.md)
