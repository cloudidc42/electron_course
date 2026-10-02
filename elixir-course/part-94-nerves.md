# Part 94: Nerves Embedded Systems (Steps 1001-1010)

## Step 1001: Nerves Introduction

```elixir
# Nerves - Elixir for embedded systems (Raspberry Pi, BeagleBone, etc.)
# mix nerves.new my_firmware --target rpi4

# mix.exs
defp deps do
  [
    {:nerves,       "~> 1.10",  runtime: false},
    {:shoehorn,     "~> 0.9"},
    {:ring_logger,  "~> 0.11"},
    {:toolshed,     "~> 0.4"},
    {:nerves_runtime, "~> 0.13"},
    {:nerves_network, "~> 0.5"},
    # Target-specific
    {:nerves_system_rpi4, "~> 1.24", runtime: false, targets: :rpi4}
  ]
end

# lib/my_firmware/application.ex
defmodule MyFirmware.Application do
  use Application

  def start(_type, _args) do
    children = [
      MyFirmware.Sensor.Supervisor,
      MyFirmware.Network,
      MyFirmware.Dashboard
    ]

    opts = [strategy: :one_for_one, name: MyFirmware.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

---

## Step 1002: GPIO Control

```elixir
# mix.exs: {:circuits_gpio, "~> 1.2"}

defmodule MyFirmware.GPIO do
  alias Circuits.GPIO

  @led_pin    17  # GPIO 17
  @button_pin 27  # GPIO 27

  def start do
    {:ok, led}    = GPIO.open(@led_pin, :output)
    {:ok, button} = GPIO.open(@button_pin, :input, pull_mode: :pullup)
    
    GPIO.set_interrupts(button, :both)
    
    {led, button}
  end

  def blink(led, times \\ 3, interval \\ 500) do
    for _ <- 1..times do
      GPIO.write(led, 1)
      Process.sleep(interval)
      GPIO.write(led, 0)
      Process.sleep(interval)
    end
  end
end

# GenServer LED controller
defmodule MyFirmware.LED do
  use GenServer

  def start_link(pin) do
    GenServer.start_link(__MODULE__, pin, name: __MODULE__)
  end

  def on,  do: GenServer.cast(__MODULE__, :on)
  def off, do: GenServer.cast(__MODULE__, :off)
  def blink(times, interval), do: GenServer.call(__MODULE__, {:blink, times, interval})

  def init(pin) do
    {:ok, gpio} = Circuits.GPIO.open(pin, :output)
    {:ok, %{gpio: gpio, state: :off}}
  end

  def handle_cast(:on, state) do
    Circuits.GPIO.write(state.gpio, 1)
    {:noreply, %{state | state: :on}}
  end

  def handle_cast(:off, state) do
    Circuits.GPIO.write(state.gpio, 0)
    {:noreply, %{state | state: :off}}
  end

  def handle_call({:blink, times, interval}, _from, state) do
    for _ <- 1..times do
      Circuits.GPIO.write(state.gpio, 1)
      Process.sleep(interval)
      Circuits.GPIO.write(state.gpio, 0)
      Process.sleep(interval)
    end
    {:reply, :ok, state}
  end
end
```

---

## Step 1003: I2C Sensor Reading

```elixir
# mix.exs: {:circuits_i2c, "~> 2.0"}

defmodule MyFirmware.Sensors.BMP280 do
  # BMP280 Temperature/Pressure sensor over I2C
  alias Circuits.I2C

  @addr       0x76
  @reg_ctrl   0xF4
  @reg_data   0xF7
  @reg_calib  0x88

  def start do
    {:ok, i2c} = I2C.open("i2c-1")
    
    # Set oversampling
    I2C.write(i2c, @addr, <<@reg_ctrl, 0x27>>)
    
    calibration = read_calibration(i2c)
    
    {:ok, %{i2c: i2c, cal: calibration}}
  end

  def read_temperature(%{i2c: i2c, cal: cal}) do
    {:ok, raw} = I2C.write_read(i2c, @addr, <<@reg_data>>, 6)
    
    <<tp_msb, tp_lsb, tp_xlsb, _rest::binary>> = raw
    adc_t = (tp_msb <<< 12) + (tp_lsb <<< 4) + (tp_xlsb >>> 4)
    
    # Apply BMP280 compensation formula
    var1 = (adc_t / 16384.0 - cal.dig_t1 / 1024.0) * cal.dig_t2
    var2 = (adc_t / 131072.0 - cal.dig_t1 / 8192.0) *
           (adc_t / 131072.0 - cal.dig_t1 / 8192.0) * cal.dig_t3
    
    t_fine = var1 + var2
    temperature = t_fine / 5120.0
    
    {:ok, Float.round(temperature, 2)}
  end

  defp read_calibration(i2c) do
    {:ok, data} = I2C.write_read(i2c, @addr, <<@reg_calib>>, 24)
    
    <<dig_t1::unsigned-little-16, dig_t2::signed-little-16,
      dig_t3::signed-little-16, _rest::binary>> = data

    %{dig_t1: dig_t1, dig_t2: dig_t2, dig_t3: dig_t3}
  end
end
```

---

## Step 1004: Serial Communication

```elixir
# mix.exs: {:circuits_uart, "~> 1.5"}

defmodule MyFirmware.Serial do
  use GenServer
  alias Circuits.UART

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def send(data) do
    GenServer.call(__MODULE__, {:send, data})
  end

  def init(opts) do
    {:ok, uart} = UART.start_link()
    
    :ok = UART.open(uart, opts[:port] || "/dev/ttyUSB0",
      speed:      opts[:baud] || 9600,
      framing:    {UART.Framing.Line, separator: "\r\n"},
      active:     true
    )

    {:ok, %{uart: uart}}
  end

  def handle_call({:send, data}, _from, state) do
    result = UART.write(state.uart, data)
    {:reply, result, state}
  end

  def handle_info({:circuits_uart, _port, {:error, reason}}, state) do
    Logger.error("UART error: #{inspect(reason)}")
    {:noreply, state}
  end

  def handle_info({:circuits_uart, port, data}, state) do
    Logger.info("Received on #{port}: #{inspect(data)}")
    process_received(data)
    {:noreply, state}
  end

  defp process_received(data) do
    Phoenix.PubSub.broadcast(MyFirmware.PubSub, "serial:received", {data})
  end
end
```

---

## Step 1005: Network Configuration

```elixir
defmodule MyFirmware.Network do
  use GenServer
  require Logger

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    configure_network()
    {:ok, %{}}
  end

  defp configure_network do
    # WiFi configuration from application env
    ssid     = Application.get_env(:my_firmware, :wifi_ssid)
    password = Application.get_env(:my_firmware, :wifi_psk)

    VintageNet.configure("wlan0", %{
      type:       VintageNetWiFi,
      vintage_net_wifi: %{
        networks: [%{
          key_mgmt: :wpa_psk,
          ssid:     ssid,
          psk:      password
        }]
      },
      ipv4: %{method: :dhcp}
    })
  end

  def connected? do
    VintageNet.get(["interface", "wlan0", "connection"]) == :internet
  end

  def ip_address do
    VintageNet.get(["interface", "wlan0", "addresses"])
    |> Enum.find(& &1.family == :inet)
    |> case do
      nil -> nil
      addr -> addr.address |> :inet.ntoa() |> to_string()
    end
  end
end
```

---

## Step 1006: Data Collection & Cloud Sync

```elixir
defmodule MyFirmware.DataCollector do
  use GenServer

  @collect_interval 60_000  # every minute
  @sync_interval    300_000 # every 5 minutes

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    :timer.send_interval(@collect_interval, :collect)
    :timer.send_interval(@sync_interval,    :sync)
    {:ok, %{readings: []}}
  end

  def handle_info(:collect, state) do
    reading = %{
      timestamp:   DateTime.utc_now(),
      temperature: read_temperature(),
      humidity:    read_humidity(),
      pressure:    read_pressure()
    }

    Logger.info("Collected: #{inspect(reading)}")
    {:noreply, %{state | readings: [reading | state.readings]}}
  end

  def handle_info(:sync, %{readings: []} = state), do: {:noreply, state}
  def handle_info(:sync, state) do
    if MyFirmware.Network.connected?() do
      case sync_to_cloud(state.readings) do
        :ok ->
          Logger.info("Synced #{length(state.readings)} readings")
          {:noreply, %{state | readings: []}}
        {:error, reason} ->
          Logger.warning("Sync failed: #{inspect(reason)}, keeping readings")
          {:noreply, state}
      end
    else
      Logger.warning("No internet, skipping sync")
      {:noreply, state}
    end
  end

  defp sync_to_cloud(readings) do
    endpoint = Application.get_env(:my_firmware, :cloud_endpoint)
    api_key  = Application.get_env(:my_firmware, :api_key)
    
    case Req.post(endpoint,
           json:    %{device_id: node(), readings: readings},
           headers: [{"x-api-key", api_key}]) do
      {:ok, %{status: 200}} -> :ok
      {:error, reason}      -> {:error, reason}
    end
  end

  defp read_temperature, do: 25.0 + (:rand.uniform(100) - 50) / 100
  defp read_humidity,    do: 60.0 + (:rand.uniform(20) - 10) / 10
  defp read_pressure,    do: 1013.25 + (:rand.uniform(10) - 5) / 10
end
```

---

## Step 1007: OTA Updates

```elixir
defmodule MyFirmware.OTAUpdate do
  require Logger

  @firmware_server Application.compile_env(:my_firmware, :firmware_server)

  def check_and_update do
    current  = Nerves.Runtime.KV.get("nerves_fw_version")
    
    case fetch_latest_version() do
      {:ok, latest} when latest != current ->
        Logger.info("Update available: #{current} → #{latest}")
        perform_update(latest)
      {:ok, _} ->
        Logger.info("Firmware up to date: #{current}")
        :ok
      {:error, reason} ->
        Logger.warning("Cannot check for updates: #{inspect(reason)}")
        {:error, reason}
    end
  end

  defp fetch_latest_version do
    case Req.get("#{@firmware_server}/latest") do
      {:ok, %{status: 200, body: %{"version" => version}}} -> {:ok, version}
      {:error, reason} -> {:error, reason}
    end
  end

  defp perform_update(version) do
    url = "#{@firmware_server}/firmware/#{version}.fw"
    
    Logger.info("Downloading firmware #{version}...")
    
    case Nerves.Runtime.KV.put("fwup_block_device", "/dev/mmcblk0") do
      :ok ->
        Nerves.Runtime.FwupOpts.new()
        |> Nerves.Runtime.FwupOpts.put(:apply, url)
        |> Nerves.Runtime.apply_fwup()
        |> case do
          :ok ->
            Logger.info("Update applied, rebooting...")
            Nerves.Runtime.reboot()
          {:error, reason} ->
            Logger.error("Update failed: #{inspect(reason)}")
            {:error, reason}
        end
    end
  end
end
```

---

## Step 1008: Watchdog & Recovery

```elixir
defmodule MyFirmware.Watchdog do
  use GenServer

  @heartbeat_interval 30_000  # 30s
  @max_missed_beats   3

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    :timer.send_interval(@heartbeat_interval, :check)
    {:ok, %{missed: 0, services: %{}}}
  end

  def heartbeat(service_name) do
    GenServer.cast(__MODULE__, {:heartbeat, service_name})
  end

  def handle_cast({:heartbeat, service}, state) do
    services = Map.put(state.services, service, System.monotonic_time(:millisecond))
    {:noreply, %{state | services: services}}
  end

  def handle_info(:check, state) do
    now = System.monotonic_time(:millisecond)
    
    dead_services = state.services
      |> Enum.filter(fn {_name, last} ->
        now - last > @heartbeat_interval * @max_missed_beats
      end)
      |> Keyword.keys()

    if length(dead_services) > 0 do
      Logger.error("Dead services: #{inspect(dead_services)}, restarting...")
      Enum.each(dead_services, &restart_service/1)
    end

    {:noreply, state}
  end

  defp restart_service(service) do
    case GenServer.whereis(service) do
      nil -> :ok  # Already dead
      pid ->
        Process.exit(pid, :kill)
        Logger.warning("Restarted #{service}")
    end
  end
end

# Hardware watchdog
defmodule MyFirmware.HWWatchdog do
  use GenServer

  @feed_interval 10_000  # Must feed every 10s

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    {:ok, wdt} = File.open("/dev/watchdog", [:write])
    :timer.send_interval(@feed_interval, :feed)
    {:ok, %{wdt: wdt}}
  end

  def handle_info(:feed, %{wdt: wdt} = state) do
    IO.write(wdt, "1")
    {:noreply, state}
  end
end
```

---

## Step 1009: Local Web Interface

```elixir
# Nerves device web UI via Phoenix
defmodule MyFirmwareWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :my_firmware

  socket "/live",  Phoenix.LiveView.Socket
  socket "/socket", MyFirmwareWeb.UserSocket

  plug Plug.Static, at: "/", from: :my_firmware
  plug MyFirmwareWeb.Router
end

defmodule MyFirmwareWeb.DashboardLive do
  use Phoenix.LiveView

  def mount(_params, _session, socket) do
    if connected?(socket) do
      :timer.send_interval(2000, :refresh)
      Phoenix.PubSub.subscribe(MyFirmware.PubSub, "serial:received")
    end

    {:ok, assign(socket, data: get_latest_data())}
  end

  def handle_info(:refresh, socket) do
    {:noreply, assign(socket, :data, get_latest_data())}
  end

  def handle_info({"serial:received", data}, socket) do
    {:noreply, update(socket, :data, &Map.put(&1, :last_serial, data))}
  end

  defp get_latest_data do
    %{
      temperature:  get_temperature(),
      uptime:       :erlang.statistics(:wall_clock) |> elem(0) |> div(1000),
      memory:       :erlang.memory(:total),
      ip:           MyFirmware.Network.ip_address(),
      connected:    MyFirmware.Network.connected?()
    }
  end

  defp get_temperature do
    case MyFirmware.Sensors.BMP280.read_temperature(%{}) do
      {:ok, temp} -> temp
      _           -> nil
    end
  end
end
```

---

## Step 1010: Device Management

```elixir
defmodule MyFirmware.DeviceManager do
  use GenServer

  @server_url Application.compile_env(:my_firmware, :device_server)

  def start_link(_) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    register_device()
    :timer.send_interval(60_000, :heartbeat)
    {:ok, %{}}
  end

  def handle_info(:heartbeat, state) do
    send_heartbeat()
    {:noreply, state}
  end

  defp register_device do
    info = %{
      device_id:   device_id(),
      version:     Nerves.Runtime.KV.get("nerves_fw_version"),
      platform:    Nerves.Runtime.KV.get("nerves_fw_platform"),
      ip_address:  MyFirmware.Network.ip_address()
    }

    Req.post("#{@server_url}/devices/register", json: info)
  end

  defp send_heartbeat do
    stats = %{
      device_id:  device_id(),
      uptime:     :erlang.statistics(:wall_clock) |> elem(0),
      memory:     :erlang.memory(:total),
      cpu:        get_cpu_temp()
    }

    Req.post("#{@server_url}/devices/heartbeat", json: stats)
  end

  defp device_id do
    Nerves.Runtime.serial_number() || "#{node()}"
  end

  defp get_cpu_temp do
    case File.read("/sys/class/thermal/thermal_zone0/temp") do
      {:ok, temp} -> String.trim(temp) |> String.to_integer() |> Kernel./(1000)
      _           -> nil
    end
  end
end
```

---

## สรุป Part 94

✅ **Step 1001** - Nerves introduction  
✅ **Step 1002** - GPIO control  
✅ **Step 1003** - I2C sensor reading  
✅ **Step 1004** - Serial communication  
✅ **Step 1005** - Network configuration  
✅ **Step 1006** - Data collection & cloud sync  
✅ **Step 1007** - OTA updates  
✅ **Step 1008** - Watchdog & recovery  
✅ **Step 1009** - Local web interface  
✅ **Step 1010** - Device management  

➡️ [Part 95: LiveBook & Data Science](./part-95-livebook.md)
