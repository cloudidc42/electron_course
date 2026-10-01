# Part 37: Nerves - Embedded Systems (Steps 401-420)

## Step 401: Nerves Overview

```
Nerves = Elixir framework for embedded systems
- Build firmware images for Raspberry Pi, BeagleBone, etc.
- Tiny Linux + OTP runtime (~25MB)
- Over-the-air updates
- Hardware access via Circuits libraries
- Port of all Elixir/Erlang goodies to embedded

Use Cases:
- IoT sensors
- Industrial automation
- Smart home devices
- Edge computing
- Robotics

Supported Hardware:
- Raspberry Pi Zero/3/4/5
- BeagleBone Black/Green
- x86_64
- GRiSP
```

---

## Step 402: Setup

```bash
# Install Nerves
mix archive.install hex nerves_bootstrap

# Create new Nerves project
mix nerves.new my_iot_device

# Project structure:
# my_iot_device/
#   config/
#     config.exs
#     target.exs
#   lib/
#     my_iot_device/         ← application code
#     my_iot_device_web/     ← optional web UI
#   rootfs_overlay/          ← files copied to firmware
#   mix.exs

# mix.exs targets
@targets [:rpi0, :rpi3, :rpi4, :bbb, :x86_64]

defp deps do
  [
    {:nerves, "~> 1.10", runtime: false},
    {:shoehorn, "~> 0.9"},
    {:ring_logger, "~> 0.11"},
    {:toolshed, "~> 0.4"},
    {:circuits_gpio, "~> 1.0"},
    {:circuits_i2c, "~> 2.0"},
    {:circuits_uart, "~> 1.5"}
  ] ++ target_deps(Mix.target())
end

defp target_deps(:host), do: []
defp target_deps(target) do
  [
    {:nerves_runtime, "~> 0.13"},
    {:nerves_pack, "~> 0.7"},
    {:"nerves_system_#{target}", "~> 1.26", runtime: false, targets: target}
  ]
end
```

---

## Step 403: GPIO

```elixir
# mix.exs: {:circuits_gpio, "~> 1.0"}

defmodule MyDevice.LED do
  use GenServer
  
  alias Circuits.GPIO
  
  @led_pin 17  # BCM pin 17 on Raspberry Pi
  
  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end
  
  def on,  do: GenServer.call(__MODULE__, :on)
  def off, do: GenServer.call(__MODULE__, :off)
  def toggle, do: GenServer.call(__MODULE__, :toggle)
  
  def init(_) do
    {:ok, gpio} = GPIO.open(@led_pin, :output)
    GPIO.write(gpio, 0)
    {:ok, %{gpio: gpio, state: :off}}
  end
  
  def handle_call(:on, _from, %{gpio: gpio} = state) do
    GPIO.write(gpio, 1)
    {:reply, :ok, %{state | state: :on}}
  end
  
  def handle_call(:off, _from, %{gpio: gpio} = state) do
    GPIO.write(gpio, 0)
    {:reply, :ok, %{state | state: :off}}
  end
  
  def handle_call(:toggle, _from, %{gpio: gpio, state: current} = state) do
    case current do
      :on  -> GPIO.write(gpio, 0); {:reply, :ok, %{state | state: :off}}
      :off -> GPIO.write(gpio, 1); {:reply, :ok, %{state | state: :on}}
    end
  end
  
  def terminate(_reason, %{gpio: gpio}) do
    GPIO.write(gpio, 0)
    GPIO.close(gpio)
  end
end

defmodule MyDevice.Button do
  use GenServer
  
  alias Circuits.GPIO
  
  @button_pin 18
  
  def start_link(handler), do: GenServer.start_link(__MODULE__, handler, name: __MODULE__)
  
  def init(handler) do
    {:ok, gpio} = GPIO.open(@button_pin, :input, pull_mode: :pullup)
    GPIO.set_interrupts(gpio, :both)
    {:ok, %{gpio: gpio, handler: handler, last_value: GPIO.read(gpio)}}
  end
  
  def handle_info({:circuits_gpio, _pin, _timestamp, value}, %{handler: handler} = state) do
    if value != state.last_value do
      case value do
        0 -> handler.(:pressed)
        1 -> handler.(:released)
      end
    end
    {:noreply, %{state | last_value: value}}
  end
end
```

---

## Step 404: I2C Sensors

```elixir
# Read temperature from BMP280 sensor via I2C
defmodule MyDevice.BMP280 do
  alias Circuits.I2C
  
  @i2c_bus "i2c-1"
  @bmp280_addr 0x76
  
  # Registers
  @ctrl_meas  0xF4
  @temp_xlsb  0xFC
  @calib_data 0x88
  
  def start do
    {:ok, i2c} = I2C.open(@i2c_bus)
    
    # Set normal mode, temperature + pressure oversampling
    I2C.write(i2c, @bmp280_addr, <<@ctrl_meas, 0xB7>>)
    
    {:ok, i2c}
  end
  
  def read_temperature(i2c) do
    # Read calibration data
    {:ok, calib_raw} = I2C.write_read(i2c, @bmp280_addr, <<@calib_data>>, 24)
    calib = parse_calibration(calib_raw)
    
    # Read raw temperature
    {:ok, temp_raw} = I2C.write_read(i2c, @bmp280_addr, <<@temp_xlsb>>, 3)
    <<msb, lsb, xlsb>> = temp_raw
    adc_t = (msb <<< 12) ||| (lsb <<< 4) ||| (xlsb >>> 4)
    
    compensate_temperature(adc_t, calib)
  end
  
  defp compensate_temperature(adc_t, %{dig_t1: t1, dig_t2: t2, dig_t3: t3}) do
    var1 = (adc_t / 16384.0 - t1 / 1024.0) * t2
    var2 = (adc_t / 131072.0 - t1 / 8192.0) * (adc_t / 131072.0 - t1 / 8192.0) * t3
    
    t_fine = round(var1 + var2)
    temp_celsius = (var1 + var2) / 5120.0
    
    {Float.round(temp_celsius, 2), t_fine}
  end
  
  defp parse_calibration(<<
    t1::little-16, t2::little-signed-16, t3::little-signed-16,
    _rest::binary
  >>) do
    %{dig_t1: t1, dig_t2: t2, dig_t3: t3}
  end
end
```

---

## Step 405: UART Communication

```elixir
# GPS over UART
defmodule MyDevice.GPS do
  use GenServer
  
  alias Circuits.UART
  
  @serial_port "ttyS0"
  @baud_rate 9600
  
  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)
  
  def get_position, do: GenServer.call(__MODULE__, :get_position)
  
  def init(_) do
    {:ok, uart} = UART.start_link()
    UART.open(uart, @serial_port,
      speed: @baud_rate,
      active: true,
      framing: {UART.Framing.Line, separator: "\r\n"}
    )
    
    {:ok, %{uart: uart, position: nil}}
  end
  
  def handle_call(:get_position, _from, %{position: pos} = state) do
    {:reply, pos, state}
  end
  
  def handle_info({:circuits_uart, _port, line}, state) do
    state = case parse_nmea(line) do
      {:ok, position} -> %{state | position: position}
      :skip           -> state
    end
    {:noreply, state}
  end
  
  defp parse_nmea("$GPRMC," <> rest) do
    parts = String.split(rest, ",")
    
    case parts do
      [_time, "A", lat, lat_dir, lon, lon_dir | _] ->
        {:ok, %{
          lat: parse_coordinate(lat, lat_dir),
          lon: parse_coordinate(lon, lon_dir)
        }}
      _ -> :skip
    end
  end
  defp parse_nmea(_), do: :skip
  
  defp parse_coordinate(coord, dir) do
    {degrees, minutes_str} = coord |> String.split_at(2)
    {degrees, ""} = Integer.parse(degrees)
    {minutes, ""} = Float.parse(minutes_str)
    
    dd = degrees + minutes / 60
    if dir in ["S", "W"], do: -dd, else: dd
  end
end
```

---

## Step 406: VintageNet (Networking)

```elixir
# mix.exs: {:vintage_net, "~> 0.13"}, {:vintage_net_wifi, "~> 0.12"}

defmodule MyDevice.Network do
  def configure_wifi(ssid, password) do
    VintageNet.configure("wlan0", %{
      type: VintageNetWiFi,
      vintage_net_wifi: %{
        networks: [
          %{
            key_mgmt: :wpa_psk,
            ssid: ssid,
            psk: password
          }
        ]
      },
      ipv4: %{method: :dhcp}
    })
  end
  
  def configure_ethernet do
    VintageNet.configure("eth0", %{
      type: VintageNetEthernet,
      ipv4: %{method: :dhcp}
    })
  end
  
  def get_ip(interface \\ "eth0") do
    case VintageNet.get(["interface", interface, "addresses"]) do
      [%{address: address} | _] -> {:ok, :inet.ntoa(address)}
      _ -> {:error, :no_ip}
    end
  end
  
  def online? do
    case VintageNet.get(["connection"]) do
      :internet -> true
      _         -> false
    end
  end
  
  def watch_connection do
    VintageNet.subscribe(["connection"])
    receive do
      {VintageNet, ["connection"], _old, :internet, _meta} ->
        IO.puts("Connected to internet!")
    end
  end
end
```

---

## Step 407: OTA Updates

```elixir
# mix.exs: {:nerves_hub_link, "~> 2.0"}

# config/target.exs
config :nerves_hub_link,
  host: "devices.nerveshub.org",
  org: "my-org",
  product: "my-iot-device"

defmodule MyDevice.Updater do
  def check_for_update do
    case NervesHubLink.check_for_update() do
      {:ok, %{firmware_url: url, firmware_meta: meta}} ->
        IO.puts("Update available: #{meta.version}")
        {:ok, :update_available}
      
      {:ok, :no_update} ->
        {:ok, :up_to_date}
      
      {:error, reason} ->
        {:error, reason}
    end
  end
  
  def update_now do
    NervesHubLink.update()
    # Device will reboot into new firmware
  end
end

# Deploy firmware
# mix firmware          # build firmware image
# mix upload            # deploy to device via SSH
# mix nerves_hub.publish # push to NervesHub for OTA

# Manual OTA
defmodule MyDevice.FirmwareUploader do
  def apply_firmware(firmware_path) do
    firmware_path
    |> Nerves.Runtime.KV.get_all_active()
    |> then(fn _ -> :nerves_runtime.reboot_to_firmware(firmware_path) end)
  end
end
```

---

## Step 408: Phoenix ใน Nerves

```elixir
# Lightweight web UI on device
defmodule MyDeviceWeb.DashboardController do
  use MyDeviceWeb, :controller
  
  def index(conn, _params) do
    render(conn, :index,
      temp:     MyDevice.BMP280.read_temperature(get_i2c()),
      position: MyDevice.GPS.get_position(),
      ip:       MyDevice.Network.get_ip(),
      uptime:   get_uptime()
    )
  end
  
  defp get_uptime do
    {uptime, _} = :erlang.statistics(:wall_clock)
    uptime |> div(1000)  # seconds
  end
end

# Real-time sensor data via LiveView
defmodule MyDeviceWeb.SensorLive do
  use MyDeviceWeb, :live_view
  
  @update_interval 1_000
  
  def mount(_params, _session, socket) do
    if connected?(socket), do: :timer.send_interval(@update_interval, :update)
    {:ok, assign(socket, readings: get_readings())}
  end
  
  def handle_info(:update, socket) do
    {:noreply, assign(socket, readings: get_readings())}
  end
  
  def render(assigns) do
    ~H"""
    <div class="sensor-dashboard">
      <h1>IoT Device Dashboard</h1>
      <p>Temperature: <%= @readings.temperature %>°C</p>
      <p>GPS: <%= @readings.lat %>, <%= @readings.lon %></p>
    </div>
    """
  end
  
  defp get_readings do
    {temp, _} = MyDevice.BMP280.read_temperature(Process.get(:i2c))
    pos = MyDevice.GPS.get_position()
    %{temperature: temp, lat: pos && pos.lat, lon: pos && pos.lon}
  end
end
```

---

## Step 409: Edge ML ด้วย Nx

```elixir
defmodule MyDevice.EdgeML do
  @moduledoc "Run ML inference on device"
  
  # Load pre-trained model at startup
  def setup do
    model_path = Application.app_dir(:my_iot_device, "priv/models/anomaly_detector.axon")
    
    {model, params} = Axon.deserialize(File.read!(model_path))
    
    # Warm up
    sample = Nx.zeros({1, 10})
    Axon.predict(model, params, sample)
    
    {model, params}
  end
  
  # Detect anomalies in sensor data
  def detect_anomaly({model, params}, sensor_readings) do
    input = sensor_readings
      |> Enum.map(&elem(&1, 1))  # extract values
      |> Nx.tensor({:f, 32})
      |> Nx.new_axis(0)          # add batch dimension
    
    prediction = Axon.predict(model, params, input)
    anomaly_score = prediction[0][0] |> Nx.to_number()
    
    cond do
      anomaly_score > 0.9 -> {:anomaly, :critical, anomaly_score}
      anomaly_score > 0.7 -> {:anomaly, :warning, anomaly_score}
      true                -> {:normal, anomaly_score}
    end
  end
end
```

---

## Step 410: Complete IoT Application

```elixir
defmodule MyDevice.Application do
  use Application
  
  def start(_type, _args) do
    children = [
      {Registry, keys: :unique, name: MyDevice.Registry},
      MyDevice.Network,
      MyDevice.BMP280,
      MyDevice.GPS,
      {MyDevice.Button, &on_button_press/1},
      MyDevice.LED,
      MyDevice.DataLogger,
      MyDevice.CloudSync,
      MyDeviceWeb.Endpoint
    ]
    
    Supervisor.start_link(children, strategy: :one_for_one)
  end
  
  defp on_button_press(:pressed) do
    MyDevice.LED.toggle()
    reading = MyDevice.BMP280.read_temperature(get_i2c())
    MyDevice.DataLogger.log(:button_press, reading)
  end
  defp on_button_press(:released), do: :ok
end

defmodule MyDevice.DataLogger do
  use GenServer
  
  @db_path "/data/sensor_readings.db"
  
  def log(event, data) do
    GenServer.cast(__MODULE__, {:log, event, data})
  end
  
  def init(_) do
    {:ok, db} = Exqlite.open_db(@db_path)
    setup_db(db)
    {:ok, %{db: db}}
  end
  
  def handle_cast({:log, event, data}, %{db: db} = state) do
    Exqlite.execute(db,
      "INSERT INTO readings (event, data, ts) VALUES (?, ?, ?)",
      [to_string(event), Jason.encode!(data), System.os_time(:second)]
    )
    {:noreply, state}
  end
  
  defp setup_db(db) do
    Exqlite.execute(db, """
      CREATE TABLE IF NOT EXISTS readings (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        event TEXT NOT NULL,
        data TEXT NOT NULL,
        ts INTEGER NOT NULL
      )
    """)
  end
end
```

---

## สรุป Part 37

✅ **Step 401** - Nerves overview  
✅ **Step 402** - Setup + project structure  
✅ **Step 403** - GPIO (LED, Button)  
✅ **Step 404** - I2C sensors (BMP280)  
✅ **Step 405** - UART (GPS)  
✅ **Step 406** - VintageNet networking  
✅ **Step 407** - OTA updates  
✅ **Step 408** - Phoenix on device  
✅ **Step 409** - Edge ML  
✅ **Step 410** - Complete IoT app  

➡️ [Part 38: Security และ Hardening](./part-38-security.md)
