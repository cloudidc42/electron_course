# Part 57: NIF & Native Integration (Steps 621-640)

## Step 621: What are NIFs?

```
NIF = Native Implemented Function
- C/C++/Rust code called directly from Elixir
- Runs in the BEAM scheduler thread (no context switch)
- Must complete quickly (<1ms) or it blocks the scheduler
- Used for: CPU-intensive computation, C library bindings

Use cases:
✅ Image processing (libvips, ImageMagick)
✅ Cryptography (libsodium, OpenSSL)
✅ Data compression (zstd, lz4)
✅ ML inference (ONNX, TensorRT)
✅ Regular expressions (re2)
✅ JSON parsing (simdjson)

Alternatives if NIF is too risky:
- Port: separate OS process, communicates via stdin/stdout
- Port driver: like NIF but with driver API, slightly safer
- Rustler: safe NIFs in Rust (recommended over C)
```

---

## Step 622: Simple NIF in C

```c
/* c_src/my_math.c */
#include <erl_nif.h>

static ERL_NIF_TERM add_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    int a, b;
    
    if (!enif_get_int(env, argv[0], &a) ||
        !enif_get_int(env, argv[1], &b)) {
        return enif_make_badarg(env);
    }
    
    return enif_make_int(env, a + b);
}

static ERL_NIF_TERM fibonacci_nif(ErlNifEnv* env, int argc, const ERL_NIF_TERM argv[]) {
    int n;
    if (!enif_get_int(env, argv[0], &n)) {
        return enif_make_badarg(env);
    }
    
    if (n <= 1) return enif_make_int(env, n);
    
    long long a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        long long c = a + b;
        a = b;
        b = c;
    }
    
    return enif_make_int64(env, b);
}

static ErlNifFunc nif_funcs[] = {
    {"add", 2, add_nif, 0},
    {"fibonacci", 1, fibonacci_nif, 0}
};

ERL_NIF_INIT(Elixir.MyApp.Math, nif_funcs, NULL, NULL, NULL, NULL)
```

```elixir
# lib/my_app/math.ex
defmodule MyApp.Math do
  @on_load :load_nif

  def load_nif do
    :erlang.load_nif(~c"#{:code.priv_dir(:my_app)}/my_math", 0)
  end

  # Fallback if NIF fails to load
  def add(_a, _b), do: raise("NIF not loaded")
  def fibonacci(_n), do: raise("NIF not loaded")
end

# Makefile (c_src/Makefile)
# CFLAGS = -O3 -Wall $(shell erl -noshell -noinput -eval "io:format(\"~s\", [code:lib_dir(erl_interface, include)]), halt()")
# all: $(MIX_APP_PATH)/priv/my_math.so
```

---

## Step 623: Rustler (Safe NIFs in Rust)

```toml
# native/my_nif/Cargo.toml
[package]
name    = "my_nif"
version = "0.1.0"
edition = "2021"

[lib]
name    = "my_nif"
crate-type = ["cdylib"]

[dependencies]
rustler = "0.31"
```

```rust
// native/my_nif/src/lib.rs
use rustler::{Atom, Encoder, Env, NifResult, Term};

#[rustler::nif]
fn add(a: i64, b: i64) -> i64 {
    a + b
}

#[rustler::nif]
fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => {
            let (mut a, mut b) = (0u64, 1u64);
            for _ in 2..=n {
                let c = a + b;
                a = b;
                b = c;
            }
            b
        }
    }
}

#[rustler::nif(schedule = "DirtyCpu")]  // For long-running NIFs
fn heavy_computation(data: Vec<u8>) -> Vec<u8> {
    // This runs on dirty schedulers, doesn't block BEAM
    data.iter().map(|&b| b.wrapping_add(1)).collect()
}

rustler::init!("Elixir.MyApp.RustNIF", [add, fibonacci, heavy_computation]);
```

```elixir
# lib/my_app/rust_nif.ex
defmodule MyApp.RustNIF do
  use Rustler, otp_app: :my_app, crate: "my_nif"

  def add(_a, _b),             do: :erlang.nif_error(:nif_not_loaded)
  def fibonacci(_n),           do: :erlang.nif_error(:nif_not_loaded)
  def heavy_computation(_data), do: :erlang.nif_error(:nif_not_loaded)
end
```

---

## Step 624: Port (External Process)

```elixir
defmodule MyApp.PythonPort do
  use GenServer

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def call(command, args) do
    GenServer.call(__MODULE__, {:call, command, args}, 30_000)
  end

  def init(_opts) do
    port = Port.open({:spawn, "python3 priv/worker.py"}, [
      :binary,
      :use_stdio,
      {:line, 65535}
    ])
    {:ok, %{port: port, pending: %{}}}
  end

  def handle_call({:call, command, args}, from, state) do
    id = System.unique_integer([:positive, :monotonic])
    msg = Jason.encode!(%{id: id, command: command, args: args})
    
    Port.command(state.port, msg <> "\n")
    
    {:noreply, put_in(state, [:pending, id], from)}
  end

  def handle_info({port, {:data, {:eol, line}}}, %{port: port} = state) do
    case Jason.decode(line) do
      {:ok, %{"id" => id, "result" => result}} ->
        case Map.pop(state.pending, id) do
          {nil, _} -> {:noreply, state}
          {from, pending} ->
            GenServer.reply(from, {:ok, result})
            {:noreply, %{state | pending: pending}}
        end
      _ ->
        {:noreply, state}
    end
  end

  def handle_info({port, {:exit_status, status}}, %{port: port} = state) do
    {:stop, {:port_exited, status}, state}
  end
end

# priv/worker.py
# import sys, json
# for line in sys.stdin:
#     req = json.loads(line)
#     result = process(req['command'], req['args'])
#     print(json.dumps({'id': req['id'], 'result': result}), flush=True)
```

---

## Step 625: ErlPort (Python/Ruby)

```elixir
# mix.exs: {:erlport, "~> 0.10"}

defmodule MyApp.PythonBridge do
  def start do
    {:ok, pid} = :python.start([
      {:python, ~c"python3"},
      {:python_path, ~c"#{:code.priv_dir(:my_app)}/python"}
    ])
    {:ok, pid}
  end

  def call_function(pid, module, function, args) do
    :python.call(pid, module, function, args)
  end

  def run_ml_prediction(pid, features) do
    :python.call(pid, :ml_model, :predict, [features])
  end
end

# priv/python/ml_model.py
# import pickle, numpy as np
#
# model = pickle.load(open('model.pkl', 'rb'))
#
# def predict(features):
#     X = np.array(features).reshape(1, -1)
#     return model.predict(X).tolist()
```

---

## Step 626: C Extension with erl_driver

```c
/* c_src/async_driver.c - Long-running work in separate thread */
#include "erl_driver.h"

typedef struct {
    ErlDrvPort port;
    ErlDrvTid  tid;
} DriverData;

static void async_work(void* async_data) {
    DriverData* dd = (DriverData*)async_data;
    
    // Simulate long work
    // This runs in a separate OS thread, not blocking BEAM
    char result[] = "done";
    
    ErlDrvTermData terms[] = {
        ERL_DRV_ATOM, driver_mk_atom("ok"),
        ERL_DRV_STRING, (ErlDrvTermData)result, strlen(result),
        ERL_DRV_TUPLE, 2
    };
    
    erl_drv_output_term(driver_caller(dd->port), terms, sizeof(terms)/sizeof(terms[0]));
}

static void driver_output(ErlDrvData drv_data, char *buf, ErlDrvSizeT len) {
    DriverData* dd = (DriverData*)drv_data;
    driver_async(dd->port, NULL, async_work, dd, NULL);
}
```

---

## Step 627: Zigler (Zig NIFs)

```zig
// native/my_zig/src/main.zig
const beam = @import("beam");
const std = @import("std");

pub fn add(env: beam.env, a: i64, b: i64) beam.term {
    return beam.make_i64(env, a + b);
}

pub fn sum_array(env: beam.env, arr: []const f64) beam.term {
    var total: f64 = 0;
    for (arr) |v| total += v;
    return beam.make_f64(env, total);
}
```

```elixir
defmodule MyApp.ZigNIF do
  use Zig,
    otp_app: :my_app,
    zig_version: "0.11.0"

  ~Z"""
  pub fn add(a: i64, b: i64) i64 {
    return a + b;
  }

  pub fn sum(slice: []const f64) f64 {
    var total: f64 = 0;
    for (slice) |v| total += v;
    return total;
  }
  """
end
```

---

## Step 628: Image Processing with libvips

```elixir
# mix.exs: {:image, "~> 0.38"}
# Uses libvips under the hood (must have libvips installed)

defmodule MyApp.ImageProcessor do
  alias Image

  def process_upload(input_path, output_path) do
    with {:ok, image} <- Image.open(input_path),
         {:ok, resized}  <- Image.resize(image, 800),
         {:ok, _}        <- Image.write(resized, "#{output_path}_800.webp", quality: 85),
         {:ok, thumb}    <- Image.thumbnail(image, 200),
         {:ok, _}        <- Image.write(thumb, "#{output_path}_thumb.webp", quality: 80) do
      {:ok, %{
        full_url:  "#{output_path}_800.webp",
        thumb_url: "#{output_path}_thumb.webp"
      }}
    end
  end

  def add_watermark(input_path, output_path, text) do
    with {:ok, image} <- Image.open(input_path),
         {:ok, overlay} <- Image.Text.text(text, font_size: 24, text_fill_color: :white),
         {:ok, watermarked} <- Image.compose(image, overlay, x: :right, y: :bottom, dx: -20, dy: -20),
         {:ok, _} <- Image.write(watermarked, output_path) do
      :ok
    end
  end

  def generate_avatar(name) do
    initials = name
      |> String.split()
      |> Enum.take(2)
      |> Enum.map(&String.first/1)
      |> Enum.join()
      |> String.upcase()

    color = :erlang.phash2(name, 0xFFFFFF)
    hex = Integer.to_string(color, 16) |> String.pad_leading(6, "0")

    Image.new!(200, 200, color: "##{hex}")
    |> Image.compose!(Image.Text.text!(initials, font_size: 72, text_fill_color: :white))
  end
end
```

---

## Step 629: PDF Generation

```elixir
# ChromicPDF - uses headless Chrome to generate PDFs
# mix.exs: {:chromic_pdf, "~> 1.13"}

defmodule MyApp.PDFGenerator do
  def generate_invoice(order) do
    html = Phoenix.View.render_to_string(
      MyAppWeb.InvoiceView,
      "invoice.html",
      order: order
    )

    ChromicPDF.print_to_pdf(
      {:html, html},
      print_to_pdf: %{
        format:           "A4",
        margin:           %{top: "20mm", right: "15mm", bottom: "20mm", left: "15mm"},
        printBackground:  true
      },
      output: fn pdf_binary ->
        File.write!("/tmp/invoice_#{order.id}.pdf", pdf_binary)
        {:ok, "/tmp/invoice_#{order.id}.pdf"}
      end
    )
  end

  # URL-based PDF
  def capture_url(url) do
    ChromicPDF.print_to_pdf(
      {:url, url},
      print_to_pdf: %{format: "A4"},
      output: fn binary -> {:ok, binary} end
    )
  end
end

# HTML template (priv/templates/invoice/invoice.html.eex)
# <!DOCTYPE html>
# <html>
# <head><style>body { font-family: Arial; }</style></head>
# <body>
#   <h1>Invoice #<%= @order.id %></h1>
#   ...
# </body>
# </html>
```

---

## Step 630: Interop with Node.js

```elixir
defmodule MyApp.NodeBridge do
  use GenServer

  @port_cmd "node priv/node/worker.js"

  def start_link(_), do: GenServer.start_link(__MODULE__, [], name: __MODULE__)

  def render_react(component, props) do
    GenServer.call(__MODULE__, {:render, component, props}, 10_000)
  end

  def init(_) do
    port = Port.open({:spawn, @port_cmd}, [:binary, :use_stdio, {:packet, 4}])
    {:ok, %{port: port, callers: %{}, next_id: 0}}
  end

  def handle_call({:render, component, props}, from, state) do
    id  = state.next_id
    msg = :erlang.term_to_binary(%{id: id, component: component, props: props})
    Port.command(state.port, msg)
    {:noreply, %{state | callers: Map.put(state.callers, id, from), next_id: id + 1}}
  end

  def handle_info({_port, {:data, data}}, state) do
    %{id: id, html: html} = :erlang.binary_to_term(data)
    {caller, callers} = Map.pop(state.callers, id)
    GenServer.reply(caller, {:ok, html})
    {:noreply, %{state | callers: callers}}
  end
end

# priv/node/worker.js
# const {renderToString} = require('react-dom/server')
# process.stdin.on('data', chunk => {
#   const {id, component, props} = erlpack.unpack(chunk)
#   const html = renderToString(React.createElement(components[component], props))
#   process.stdout.write(erlpack.pack({id, html}))
# })
```

---

## สรุป Part 57

✅ **Step 621** - NIF overview  
✅ **Step 622** - C NIF  
✅ **Step 623** - Rustler (Rust NIFs)  
✅ **Step 624** - Port (external process)  
✅ **Step 625** - ErlPort Python bridge  
✅ **Step 626** - Async driver  
✅ **Step 627** - Zigler (Zig NIFs)  
✅ **Step 628** - Image processing  
✅ **Step 629** - PDF generation  
✅ **Step 630** - Node.js interop  

➡️ [Part 58: BEAM Internals](./part-58-beam-internals.md)
