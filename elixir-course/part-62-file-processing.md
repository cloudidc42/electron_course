# Part 62: File Processing & Storage (Steps 671-690)

## Step 671: File Upload with Phoenix LiveView

```elixir
defmodule MyAppWeb.UploadLive do
  use MyAppWeb, :live_view

  @max_file_size 10 * 1024 * 1024  # 10 MB

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(:uploads_complete, [])
     |> allow_upload(:document,
        accept:     ~w(.pdf .doc .docx .txt),
        max_entries: 5,
        max_file_size: @max_file_size,
        auto_upload: false
       )
     |> allow_upload(:image,
        accept:     ~w(image/*),
        max_entries: 10,
        max_file_size: 5 * 1024 * 1024,
        auto_upload: true  # upload as soon as file selected
       )}
  end

  def handle_event("validate", _params, socket) do
    {:noreply, socket}
  end

  def handle_event("upload", _params, socket) do
    uploaded_files =
      consume_uploaded_entries(socket, :document, fn %{path: tmp_path}, entry ->
        dest = Path.join([:code.priv_dir(:my_app), "uploads", entry.client_name])
        File.cp!(tmp_path, dest)
        {:ok, %{name: entry.client_name, size: entry.client_size, url: "/uploads/#{entry.client_name}"}}
      end)

    {:noreply, update(socket, :uploads_complete, &(&1 ++ uploaded_files))}
  end

  def render(assigns) do
    ~H"""
    <form phx-submit="upload" phx-change="validate">
      <.live_file_input upload={@uploads.document} />
      
      <%= for entry <- @uploads.document.entries do %>
        <div>
          <span><%= entry.client_name %></span>
          <span><%= entry.progress %>%</span>
          <%= for err <- upload_errors(@uploads.document, entry) do %>
            <span class="error"><%= error_to_string(err) %></span>
          <% end %>
          <button type="button" phx-click="cancel-upload" phx-value-ref={entry.ref}>✕</button>
        </div>
      <% end %>
      
      <button type="submit">Upload</button>
    </form>
    """
  end

  defp error_to_string(:too_large),       do: "File too large (max 10MB)"
  defp error_to_string(:not_accepted),    do: "File type not allowed"
  defp error_to_string(:too_many_files),  do: "Too many files (max 5)"
end
```

---

## Step 672: S3 Direct Upload

```elixir
# mix.exs: {:ex_aws, "~> 2.5"}, {:ex_aws_s3, "~> 2.4"}, {:hackney, "~> 1.18"}

defmodule MyApp.Storage.S3 do
  @bucket System.get_env("AWS_S3_BUCKET")

  def presigned_url(key, opts \\ []) do
    config  = ExAws.Config.new(:s3)
    expires = Keyword.get(opts, :expires, 3600)

    {:ok, url} = ExAws.S3.presigned_url(config, :put, @bucket, key,
      expires_in:  expires,
      query_params: [{"Content-Type", Keyword.get(opts, :content_type, "application/octet-stream")}]
    )
    url
  end

  def upload(local_path, s3_key, opts \\ []) do
    content_type = Keyword.get(opts, :content_type, "application/octet-stream")
    
    local_path
    |> ExAws.S3.Upload.stream_file()
    |> ExAws.S3.upload(@bucket, s3_key,
       content_type: content_type,
       acl:          Keyword.get(opts, :acl, :private)
      )
    |> ExAws.request()
  end

  def upload_binary(data, s3_key, opts \\ []) do
    ExAws.S3.put_object(@bucket, s3_key, data,
      content_type: Keyword.get(opts, :content_type, "application/octet-stream")
    )
    |> ExAws.request()
  end

  def download(s3_key) do
    ExAws.S3.get_object(@bucket, s3_key)
    |> ExAws.request()
    |> case do
      {:ok, %{body: body}} -> {:ok, body}
      error -> error
    end
  end

  def delete(s3_key) do
    ExAws.S3.delete_object(@bucket, s3_key)
    |> ExAws.request()
  end

  def public_url(s3_key) do
    "https://#{@bucket}.s3.amazonaws.com/#{s3_key}"
  end

  def cdn_url(s3_key) do
    cdn = System.get_env("CDN_URL", public_url(s3_key))
    "#{cdn}/#{s3_key}"
  end
end
```

---

## Step 673: LiveView + S3 Direct Upload (Presigned)

```elixir
defmodule MyAppWeb.DirectUploadLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> allow_upload(:avatar,
        accept: ~w(image/jpeg image/png image/webp),
        max_entries: 1,
        max_file_size: 5_000_000,
        external: &presign_upload/2
       )}
  end

  # Called once per file to get a presigned S3 URL
  defp presign_upload(entry, socket) do
    key     = "uploads/#{socket.assigns.current_user.id}/#{entry.uuid}/#{entry.client_name}"
    url     = MyApp.Storage.S3.presigned_url(key, content_type: entry.client_type)
    
    {:ok, %{uploader: "S3", key: key, url: url}, socket}
  end

  def handle_event("save", _params, socket) do
    [{:ok, %{key: s3_key}}] = consume_uploaded_entries(socket, :avatar, fn _meta, _entry ->
      {:ok, %{key: "already uploaded"}}
    end)

    # Save s3_key to user profile
    MyApp.Accounts.update_avatar(socket.assigns.current_user, s3_key)

    {:noreply, socket}
  end
end
```

---

## Step 674: Image Processing Pipeline

```elixir
defmodule MyApp.ImagePipeline do
  @sizes %{
    thumbnail: {150, 150},
    small:     {320, 320},
    medium:    {800, 600},
    large:     {1200, 900}
  }

  def process_upload(upload_path, user_id) do
    base_key  = "images/#{user_id}/#{unique_id()}"
    
    results = Enum.map(@sizes, fn {size_name, {w, h}} ->
      key       = "#{base_key}/#{size_name}.webp"
      dest_path = "/tmp/#{unique_id()}.webp"
      
      with :ok <- resize_image(upload_path, dest_path, w, h),
           {:ok, _} <- MyApp.Storage.S3.upload(dest_path, key, content_type: "image/webp") do
        File.rm(dest_path)
        {:ok, {size_name, MyApp.Storage.S3.cdn_url(key)}}
      end
    end)

    case Enum.find(results, &match?({:error, _}, &1)) do
      nil ->
        urls = for {:ok, {name, url}} <- results, into: %{}, do: {name, url}
        {:ok, urls}
      {:error, reason} ->
        {:error, reason}
    end
  end

  defp resize_image(src, dst, width, height) do
    # Using Mogrify (ImageMagick wrapper)
    Mogrify.open(src)
    |> Mogrify.resize_to_limit("#{width}x#{height}")
    |> Mogrify.format("webp")
    |> Mogrify.custom("quality", "85")
    |> Mogrify.save(path: dst)
    
    :ok
  rescue
    e -> {:error, e}
  end

  defp unique_id, do: :crypto.strong_rand_bytes(16) |> Base.url_encode64(padding: false)
end
```

---

## Step 675: File Scanning (Antivirus)

```elixir
defmodule MyApp.FileScanner do
  # ClamAV integration for virus scanning

  def scan(file_path) do
    case System.cmd("clamdscan", ["--no-summary", file_path]) do
      {output, 0} when output =~ "OK" ->
        {:ok, :clean}
      {output, 1} ->
        threat = parse_threat(output)
        {:ok, :infected, threat}
      {_output, _code} ->
        # ClamAV error
        {:error, :scan_failed}
    end
  end

  defp parse_threat(output) do
    case Regex.run(~r/FOUND: (.+)$/, output) do
      [_, threat] -> String.trim(threat)
      nil         -> "Unknown threat"
    end
  end

  # Integration in upload pipeline
  def safe_upload(tmp_path, s3_key) do
    case scan(tmp_path) do
      {:ok, :clean} ->
        MyApp.Storage.S3.upload(tmp_path, s3_key)
      {:ok, :infected, threat} ->
        Logger.warning("Malware detected: #{threat} in #{s3_key}")
        {:error, {:malware_detected, threat}}
      {:error, reason} ->
        Logger.error("Scan failed: #{inspect(reason)}")
        # Decide: reject or allow without scan
        {:error, :scan_failed}
    end
  end
end
```

---

## Step 676: CSV Processing

```elixir
defmodule MyApp.CSVImporter do
  def import_products(file_path) do
    file_path
    |> File.stream!()
    |> CSV.decode!(headers: true)
    |> Stream.map(&parse_product/1)
    |> Stream.chunk_every(100)
    |> Enum.reduce({0, []}, fn batch, {count, errors} ->
      case insert_batch(batch) do
        {:ok, inserted}   -> {count + length(inserted), errors}
        {:error, reasons} -> {count, errors ++ reasons}
      end
    end)
    |> then(fn {count, errors} ->
      {:ok, %{imported: count, errors: length(errors), error_details: errors}}
    end)
  end

  defp parse_product(row) do
    %{
      name:        row["name"],
      description: row["description"],
      price:       Decimal.new(row["price"] || "0"),
      category:    row["category"],
      stock:       String.to_integer(row["stock"] || "0")
    }
  end

  defp insert_batch(products) do
    {count, _} = MyApp.Repo.insert_all(
      MyApp.Catalog.Product,
      Enum.map(products, &Map.put(&1, :inserted_at, DateTime.utc_now())),
      on_conflict: {:replace, [:description, :price, :stock]},
      conflict_target: :name
    )
    {:ok, Range.new(1, count) |> Enum.to_list()}
  end

  # Export to CSV
  def export_orders(orders) do
    headers = ["id", "user_email", "total", "status", "created_at"]

    rows = Enum.map(orders, fn o ->
      [o.id, o.user.email, Decimal.to_string(o.total), o.status, DateTime.to_iso8601(o.inserted_at)]
    end)

    [headers | rows]
    |> CSV.encode()
    |> Enum.to_list()
    |> IO.iodata_to_binary()
  end
end
```

---

## Step 677: Excel Processing

```elixir
# mix.exs: {:xlsxir, "~> 1.6"} or {:elixlsx, "~> 0.5"}

defmodule MyApp.ExcelProcessor do
  def read_xlsx(file_path) do
    {:ok, table} = Xlsxir.extract(file_path, 0)
    
    [header | rows] = Xlsxir.get_list(table)
    headers = Enum.map(header, &String.downcase/1)
    
    data = Enum.map(rows, fn row ->
      headers
      |> Enum.zip(row)
      |> Enum.into(%{})
    end)
    
    Xlsxir.close(table)
    {:ok, data}
  end

  def write_xlsx(data, headers, output_path) do
    rows = Enum.map(data, fn row ->
      Enum.map(headers, &Map.get(row, &1, ""))
    end)

    sheet = %Elixlsx.Sheet{
      name: "Data",
      rows: [headers | rows]
    }

    workbook = %Elixlsx.Workbook{sheets: [sheet]}
    
    {:ok, {_, binary}} = Elixlsx.write_to_memory(workbook, output_path)
    File.write!(output_path, binary)
    {:ok, output_path}
  end

  # Generate report
  def generate_sales_report(from, to) do
    orders = MyApp.Orders.list_completed(from: from, to: to)

    headers = ["Order ID", "Customer", "Total", "Items", "Date"]
    data = Enum.map(orders, fn o ->
      %{
        "Order ID"  => o.id,
        "Customer"  => o.user.email,
        "Total"     => Decimal.to_string(o.total),
        "Items"     => length(o.items),
        "Date"      => Date.to_string(DateTime.to_date(o.inserted_at))
      }
    end)

    output = "/tmp/sales_report_#{Date.to_string(Date.utc_today())}.xlsx"
    write_xlsx(data, headers, output)
  end
end
```

---

## Step 678: Background File Processing

```elixir
defmodule MyApp.Workers.FileProcessingWorker do
  use Oban.Worker, queue: :file_processing, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"file_id" => file_id, "type" => type}}) do
    file = MyApp.Files.get!(file_id)
    
    case type do
      "image"    -> process_image(file)
      "csv"      -> process_csv(file)
      "document" -> process_document(file)
    end
  end

  defp process_image(file) do
    tmp = download_to_tmp(file.s3_key)
    
    {:ok, urls} = MyApp.ImagePipeline.process_upload(tmp, file.user_id)
    
    file
    |> MyApp.Files.File.changeset(%{processed: true, variants: urls})
    |> MyApp.Repo.update!()
    
    File.rm(tmp)
    :ok
  end

  defp process_csv(file) do
    tmp = download_to_tmp(file.s3_key)
    
    {:ok, result} = MyApp.CSVImporter.import_products(tmp)
    
    file
    |> MyApp.Files.File.changeset(%{
      processed: true,
      metadata: %{rows_imported: result.imported, errors: result.errors}
    })
    |> MyApp.Repo.update!()
    
    File.rm(tmp)
    :ok
  end

  defp download_to_tmp(s3_key) do
    {:ok, body} = MyApp.Storage.S3.download(s3_key)
    tmp = "/tmp/#{:erlang.unique_integer([:positive])}"
    File.write!(tmp, body)
    tmp
  end

  defp process_document(file) do
    # Extract text for full-text indexing
    tmp = download_to_tmp(file.s3_key)
    
    text = case Path.extname(file.name) do
      ".pdf" -> extract_pdf_text(tmp)
      ".txt" -> File.read!(tmp)
      _      -> ""
    end
    
    file
    |> MyApp.Files.File.changeset(%{processed: true, extracted_text: text})
    |> MyApp.Repo.update!()
    
    File.rm(tmp)
    :ok
  end

  defp extract_pdf_text(path) do
    {text, 0} = System.cmd("pdftotext", [path, "-"])
    text
  end
end
```

---

## Step 679: File Metadata & Schema

```elixir
defmodule MyApp.Files.File do
  use Ecto.Schema
  import Ecto.Changeset

  schema "files" do
    field :name,           :string
    field :s3_key,         :string
    field :content_type,   :string
    field :size,           :integer
    field :checksum,       :string
    field :processed,      :boolean, default: false
    field :variants,       :map,     default: %{}
    field :metadata,       :map,     default: %{}
    field :extracted_text, :string
    
    belongs_to :user,     MyApp.Accounts.User
    belongs_to :resource, MyApp.Resource, polymorphic: true
    
    timestamps()
  end

  def changeset(file, attrs) do
    file
    |> cast(attrs, [:name, :s3_key, :content_type, :size, :checksum, :processed, :variants, :metadata])
    |> validate_required([:name, :s3_key, :content_type])
    |> validate_length(:name, max: 255)
    |> validate_number(:size, greater_than: 0)
  end

  def compute_checksum(data) do
    :crypto.hash(:sha256, data) |> Base.encode16(case: :lower)
  end
end
```

---

## Step 680: Resumable Upload (TUS Protocol)

```elixir
defmodule MyApp.TusUpload do
  # TUS = resumable upload protocol (https://tus.io)
  # mix.exs: {:tus_plug, "~> 1.0"}

  # In your router.ex:
  # scope "/files" do
  #   forward "/upload", TusPlug, storage: MyApp.TusStorage
  # end

  defmodule MyApp.TusStorage do
    @behaviour TusPlug.Storage

    @impl true
    def init_upload(upload_id, length, _metadata) do
      :ets.insert(:tus_uploads, {upload_id, %{offset: 0, length: length}})
      {:ok, upload_id}
    end

    @impl true
    def write_chunk(upload_id, data, offset) do
      tmp_path = "/tmp/tus_#{upload_id}"
      File.write!(tmp_path, data, [:append])
      
      new_offset = offset + byte_size(data)
      :ets.update_element(:tus_uploads, upload_id, {2, %{offset: new_offset}})
      
      {:ok, new_offset}
    end

    @impl true
    def complete(upload_id) do
      tmp_path = "/tmp/tus_#{upload_id}"
      s3_key   = "uploads/#{upload_id}"
      
      {:ok, _} = MyApp.Storage.S3.upload(tmp_path, s3_key)
      File.rm(tmp_path)
      :ets.delete(:tus_uploads, upload_id)
      
      {:ok, s3_key}
    end
  end
end
```

---

## สรุป Part 62

✅ **Step 671** - LiveView file upload  
✅ **Step 672** - S3 integration  
✅ **Step 673** - S3 direct upload (presigned)  
✅ **Step 674** - Image processing pipeline  
✅ **Step 675** - File scanning (AV)  
✅ **Step 676** - CSV import/export  
✅ **Step 677** - Excel processing  
✅ **Step 678** - Background file processing  
✅ **Step 679** - File metadata schema  
✅ **Step 680** - Resumable uploads (TUS)  

➡️ [Part 63: AI/LLM Integration](./part-63-ai-llm.md)
