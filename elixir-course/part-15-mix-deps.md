# Part 15: Mix Projects และ Dependencies (Steps 161-180)

## Step 161: Mix Project Structure

```
my_app/
├── lib/
│   ├── my_app.ex          # Application module
│   └── my_app/
│       ├── worker.ex
│       └── utils.ex
├── test/
│   ├── test_helper.exs    # ExUnit setup
│   └── my_app/
│       └── worker_test.exs
├── config/
│   ├── config.exs         # Base config
│   ├── dev.exs            # Dev config
│   ├── test.exs           # Test config
│   └── prod.exs           # Production config
├── priv/
│   ├── static/            # Static assets
│   └── repo/migrations/   # DB migrations
├── mix.exs                # Project definition
├── mix.lock               # Locked dependency versions
└── .formatter.exs         # Code formatter config
```

---

## Step 162: mix.exs

```elixir
defmodule MyApp.MixProject do
  use Mix.Project
  
  def project do
    [
      app: :my_app,
      version: "1.0.0",
      elixir: "~> 1.15",
      start_permanent: Mix.env() == :prod,  # crash if App fails to start
      deps: deps(),
      
      # Release config
      releases: releases(),
      
      # Test
      test_coverage: [tool: ExCoveralls],
      preferred_cli_env: [
        coveralls: :test,
        "coveralls.detail": :test
      ],
      
      # Docs
      name: "My App",
      source_url: "https://github.com/myorg/my_app",
      docs: [
        main: "MyApp",
        extras: ["README.md"]
      ]
    ]
  end
  
  def application do
    [
      extra_applications: [:logger, :crypto],
      mod: {MyApp.Application, []}
    ]
  end
  
  defp deps do
    [
      # HTTP client
      {:req, "~> 0.4"},
      
      # Database
      {:ecto_sql, "~> 3.10"},
      {:postgrex, ">= 0.0.0"},
      
      # JSON
      {:jason, "~> 1.4"},
      
      # Phoenix (web framework)
      {:phoenix, "~> 1.7"},
      {:phoenix_ecto, "~> 4.4"},
      
      # Testing
      {:ex_doc,         "~> 0.31", only: :dev, runtime: false},
      {:excoveralls,    "~> 0.18", only: :test},
      {:mox,            "~> 1.1",  only: :test},
      {:faker,          "~> 0.18", only: :test},
      {:stream_data,    "~> 0.6",  only: [:dev, :test]},
      {:dialyxir,       "~> 1.4",  only: [:dev, :test], runtime: false},
      {:credo,          "~> 1.7",  only: [:dev, :test], runtime: false}
    ]
  end
  
  defp releases do
    [
      my_app: [
        include_executables_for: [:unix],
        steps: [:assemble, :tar]
      ]
    ]
  end
end
```

---

## Step 163: Mix Commands

```bash
# สร้าง project ใหม่
mix new my_app
mix new my_app --sup              # with Application + Supervisor
mix new my_app --umbrella         # umbrella project
mix phx.new my_web_app            # Phoenix project

# Dependencies
mix deps.get                      # ดาวน์โหลด deps
mix deps.update --all             # อัพเดท deps ทั้งหมด
mix deps.update jason             # อัพเดท dep เดียว
mix deps.compile                  # คอมไพล์ deps
mix deps.clean --all              # ลบ deps ทั้งหมด
mix deps.tree                     # แสดง dep tree

# Build/Run
mix compile                       # compile project
mix run                           # run application
mix run --no-halt                 # run and keep running
iex -S mix                        # IEx กับ project loaded

# Test
mix test                          # run ทุก tests
mix test test/my_test.exs         # run specific file
mix test --only unit              # run by tag
mix test --stale                  # only changed tests
mix test --cover                  # with coverage

# Code Quality
mix format                        # auto-format code
mix format --check-formatted      # CI check
mix credo                         # static analysis
mix dialyzer                      # type checking

# Docs
mix docs                          # generate docs

# Release
mix release                       # build release
mix release --overwrite           # overwrite existing
```

---

## Step 164: Configuration

```elixir
# config/config.exs - base config
import Config

config :my_app,
  env: config_env(),
  app_name: "My Application"

config :my_app, MyApp.Repo,
  adapter: Ecto.Adapters.Postgres,
  pool_size: 10

# Import environment specific config
import_config "#{config_env()}.exs"

# config/dev.exs
import Config

config :my_app, MyApp.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "my_app_dev",
  show_sensitive_data_on_connection_error: true,
  pool_size: 10

config :logger, :console,
  format: "[$level] $message\n"

# config/prod.exs
import Config

config :my_app, MyApp.Repo,
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
  ssl: true

config :logger, level: :info

# config/runtime.exs - evaluated at runtime (not compile time)
import Config

if config_env() == :prod do
  database_url = System.fetch_env!("DATABASE_URL")
  
  config :my_app, MyApp.Repo,
    url: database_url,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")
  
  secret_key_base = System.fetch_env!("SECRET_KEY_BASE")
  
  config :my_app, MyAppWeb.Endpoint,
    http: [ip: {0, 0, 0, 0}, port: String.to_integer(System.get_env("PORT") || "4000")],
    secret_key_base: secret_key_base
end
```

---

## Step 165: อ่าน Config ใน Application

```elixir
defmodule MyApp.Config do
  def get(key) do
    Application.get_env(:my_app, key)
  end
  
  def get(key, default) do
    Application.get_env(:my_app, key, default)
  end
  
  def fetch!(key) do
    Application.fetch_env!(:my_app, key)
  end
  
  def database_url do
    fetch!(:database_url)
  end
  
  def pool_size do
    get(:pool_size, 10)
  end
end

# ใน GenServer
defmodule MyService do
  use GenServer
  
  def start_link(opts) do
    config = Application.get_env(:my_app, __MODULE__, [])
    merged_opts = Keyword.merge(config, opts)
    GenServer.start_link(__MODULE__, merged_opts, name: __MODULE__)
  end
end
```

---

## Step 166: Mix Tasks

```elixir
# สร้าง custom Mix task
# lib/mix/tasks/my_task.ex

defmodule Mix.Tasks.MyApp.Seed do
  @moduledoc "Seeds the database with initial data"
  @shortdoc "Seeds database"
  
  use Mix.Task
  
  @impl Mix.Task
  def run(args) do
    Mix.Task.run("app.start")  # start the application
    
    case args do
      ["--env", env] -> seed_for_env(env)
      []             -> seed_for_env("dev")
      _              -> Mix.raise("Usage: mix my_app.seed [--env ENV]")
    end
  end
  
  defp seed_for_env("dev") do
    Mix.shell().info("Seeding development database...")
    
    users = [
      %{name: "Alice", email: "alice@example.com"},
      %{name: "Bob",   email: "bob@example.com"}
    ]
    
    Enum.each(users, fn user ->
      case MyApp.Accounts.create_user(user) do
        {:ok, u}    -> Mix.shell().info("Created: #{u.name}")
        {:error, e} -> Mix.shell().error("Failed: #{inspect(e)}")
      end
    end)
    
    Mix.shell().info("Done!")
  end
end

# Usage:
# mix my_app.seed
# mix my_app.seed --env prod
```

---

## Step 167: Umbrella Projects

```elixir
# Umbrella = หลาย apps ใน repo เดียว
# my_umbrella/
# ├── apps/
# │   ├── core/          # business logic
# │   ├── web/           # Phoenix web app
# │   ├── worker/        # background jobs
# │   └── api/           # REST API
# └── mix.exs            # umbrella mix.exs

# apps/core/mix.exs
defmodule Core.MixProject do
  use Mix.Project
  
  def project do
    [
      app: :core,
      version: "0.1.0",
      build_path: "../../_build",
      config_path: "../../config/config.exs",
      deps_path: "../../deps",
      lockfile: "../../mix.lock",
      elixir: "~> 1.15",
      deps: deps()
    ]
  end
  
  defp deps do
    [{:ecto_sql, "~> 3.10"}, {:postgrex, ">= 0.0.0"}]
  end
end

# apps/web/mix.exs
defmodule Web.MixProject do
  use Mix.Project
  
  def project do
    [
      app: :web,
      # ...
      deps: deps()
    ]
  end
  
  defp deps do
    [
      {:core, in_umbrella: true},  # depend on sibling app
      {:phoenix, "~> 1.7"}
    ]
  end
end
```

---

## Step 168: Release Configuration

```elixir
# mix.exs
defp releases do
  [
    my_app: [
      version: "1.0.0",
      applications: [runtime_tools: :permanent],
      include_executables_for: [:unix],
      config_providers: [
        {Config.Reader, {:system, "RELEASE_ROOT", "/config/runtime.exs"}}
      ],
      steps: [:assemble, :tar]
    ]
  ]
end

# สร้าง release
# MIX_ENV=prod mix release

# Run release
# _build/prod/rel/my_app/bin/my_app start
# _build/prod/rel/my_app/bin/my_app daemon
# _build/prod/rel/my_app/bin/my_app stop
# _build/prod/rel/my_app/bin/my_app remote   # remote IEx

# rel/overlays/ - extra files in release
# rel/overlays/config/runtime.exs
```

---

## Step 169: Hex.pm Packages

```bash
# ค้นหา packages
mix hex.search ecto

# Info เกี่ยวกับ package
mix hex.info jason

# publish package ของตัวเอง
mix hex.register        # register account
mix hex.publish         # publish
mix hex.publish docs    # publish docs only

# Private packages
mix hex.organization auth my_org

# Audit packages
mix hex.audit           # check for known vulnerabilities
```

### 169.1 สร้าง Hex Package

```elixir
# mix.exs
def project do
  [
    app: :my_library,
    version: "1.0.0",
    description: "A great library",
    package: package(),
    deps: deps()
  ]
end

defp package do
  [
    name: :my_library,
    licenses: ["MIT"],
    links: %{"GitHub" => "https://github.com/me/my_library"},
    maintainers: ["Your Name"],
    files: ~w(lib .formatter.exs mix.exs README.md CHANGELOG.md LICENSE)
  ]
end
```

---

## Step 170: Best Practices

```elixir
# 1. Pin dependencies ใน production
# mix.lock ต้อง commit เข้า git เสมอ

# 2. แยก deps ตาม environment
defp deps do
  [
    {:ecto, "~> 3.10"},                              # all envs
    {:ex_doc, only: :dev, runtime: false},           # dev only
    {:mox, "~> 1.1", only: :test}                   # test only
  ]
end

# 3. ใช้ version constraints อย่างระมัดระวัง
{:lib, "~> 1.0"}    # >= 1.0.0 and < 2.0.0 (patch + minor)
{:lib, ">= 1.0.0"}  # any version >= 1.0.0
{:lib, "== 1.0.0"}  # exact version

# 4. Check for vulnerabilities
# mix hex.audit

# 5. ใช้ formatter
# .formatter.exs
[
  inputs: ["{mix,.formatter}.exs", "{config,lib,test}/**/*.{ex,exs}"],
  import_deps: [:ecto, :phoenix]
]
```

---

## สรุป Part 15

✅ **Step 161** - Project structure  
✅ **Step 162** - mix.exs  
✅ **Step 163** - Mix commands  
✅ **Step 164** - Configuration files  
✅ **Step 165** - อ่าน config ใน code  
✅ **Step 166** - Custom Mix tasks  
✅ **Step 167** - Umbrella projects  
✅ **Step 168** - Release configuration  
✅ **Step 169** - Hex.pm packages  
✅ **Step 170** - Best practices  

➡️ [Part 16: Testing ด้วย ExUnit](./part-16-testing.md)
