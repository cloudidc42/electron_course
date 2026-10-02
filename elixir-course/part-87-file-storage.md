# Part 87: File Handling & Storage (Steps 931-950)

## Step 931: File Upload Handling

```elixir
defmodule MyAppWeb.UploadController do
  use MyAppWeb, :controller

  @max_file_size 10 * 1024 * 1024  # 10MB
  @allowed_types ~w(image/jpeg image/png image/gif application/pdf)

  def create(conn, %{"file" => %Plug.Upload{} = upload}) do
    with :ok <- validate_file(upload),
         {:ok, url} <- MyApp.Storage.upload(upload) do
      json(conn, %{url: url, filename: upload.filename})
    else
      {:error, :too_large} ->
        conn |> put_status(413) |> json(%{error: "file_too_large"})
      {:error, :invalid_type} ->
        conn |> put_status(415) |> json(%{error: "unsupported_media_type"})
    end
  end

  defp validate_file(%Plug.Upload{content_type: type, path: path}) do
    with :ok <- validate_type(type),
         :ok <- validate_size(path) do
      :ok
    end
  end

  defp validate_type(type) do
    if type in @allowed_types, do: :ok, else: {:error, :invalid_type}
  end

  defp validate_size(path) do
    case File.stat(path) do
      {:ok, %{size: size}} when size <= @max_file_size -> :ok
      {:ok, _}                                          -> {:error, :too_large}
      {:error, _}                                       -> {:error, :invalid_type}
    end
  end
end
```

---

## Step 932: LiveView File Upload

```elixir
defmodule MyAppWeb.AvatarLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok,
     socket
     |> assign(:uploaded_files, [])
     |> allow_upload(:avatar,
          accept:     ~w(.jpg .jpeg .png),
          max_entries: 1,
          max_file_size: 5_000_000,  # 5MB
          auto_upload: true
        )}
  end

  def handle_event("validate", _params, socket) do
    {:noreply, socket}
  end

  def handle_event("save", _params, socket) do
    uploaded_files = consume_uploaded_entries(socket, :avatar, fn %{path: path}, entry ->
      dest = Path.join([:code.priv_dir(:my_app), "static", "uploads", entry.client_name])
      File.cp!(path, dest)
      {:ok, ~p"/uploads/#{entry.client_name}"}
    end)

    case uploaded_files do
      [url | _] ->
        MyApp.Accounts.update_avatar(socket.assigns.current_user, url)
        {:noreply, push_navigate(socket, to: ~p"/profile")}
      [] ->
        {:noreply, put_flash(socket, :error, "No file selected")}
    end
  end

  def render(assigns) do
    ~H"""
    <form phx-submit="save" phx-change="validate">
      <.live_file_input upload={@uploads.avatar} />
      <%= for entry <- @uploads.avatar.entries do %>
        <.live_img_preview entry={entry} width="150" />
        <progress value={entry.progress} max="100"><%= entry.progress %>%</progress>
        <%= for err <- upload_errors(@uploads.avatar, entry) do %>
          <p class="error"><%= error_to_string(err) %></p>
        <% end %>
      <% end %>
      <button type="submit">Upload</button>
    </form>
    """
  end
end
```

---

## Step 933: S3 / Object Storage

```elixir
# mix.exs: {:ex_aws_s3, "~> 2.5"}, {:ex_aws, "~> 2.5"}

defmodule MyApp.Storage.S3 do
  def upload(%Plug.Upload{path: path, filename: filename, content_type: type}) do
    key     = unique_key(filename)
    bucket  = System.get_env("S3_BUCKET")

    path
    |> ExAws.S3.Upload.stream_file()
    |> ExAws.S3.upload(bucket, key,
         content_type: type,
         acl:          :public_read,
         metadata:     %{"original-filename" => filename}
       )
    |> ExAws.request()
    |> case do
      {:ok, _} ->
        url = "https://#{bucket}.s3.amazonaws.com/#{key}"
        {:ok, url}
      {:error, reason} ->
        {:error, reason}
    end
  end

  def delete(url) do
    key    = URI.parse(url).path |> String.trim_leading("/")
    bucket = System.get_env("S3_BUCKET")
    ExAws.S3.delete_object(bucket, key) |> ExAws.request()
  end

  def presigned_url(key, expires_in \\ 3600) do
    bucket = System.get_env("S3_BUCKET")
    ExAws.S3.presigned_url(ExAws.Config.new(:s3), :get, bucket, key, expires_in: expires_in)
  end

  defp unique_key(filename) do
    ext  = Path.extname(filename)
    uuid = Ecto.UUID.generate()
    "uploads/#{uuid}#{ext}"
  end
end
```

---

## Step 934: Image Processing

```elixir
# mix.exs: {:image, "~> 0.54"}

defmodule MyApp.ImageProcessor do
  def process_avatar(source_path) do
    with {:ok, image}    <- Image.open(source_path),
         {:ok, resized}  <- Image.thumbnail(image, 200),
         {:ok, cropped}  <- Image.thumbnail(resized, 200, crop: :center),
         {:ok, _}        <- Image.write(cropped, output_path(source_path)) do
      {:ok, output_path(source_path)}
    end
  end

  def create_thumbnails(source_path, sizes \\ [100, 200, 400]) do
    with {:ok, image} <- Image.open(source_path) do
      Enum.reduce_while(sizes, {:ok, []}, fn size, {:ok, paths} ->
        thumb_path = thumb_path(source_path, size)
        case Image.thumbnail(image, size) do
          {:ok, thumb} ->
            Image.write(thumb, thumb_path)
            {:cont, {:ok, [{size, thumb_path} | paths]}}
          {:error, reason} ->
            {:halt, {:error, reason}}
        end
      end)
    end
  end

  def optimize(source_path) do
    with {:ok, image}     <- Image.open(source_path),
         {:ok, optimized} <- Image.write(image, source_path,
                               quality: 85,
                               strip_metadata: true) do
      {:ok, optimized}
    end
  end

  defp output_path(path) do
    ext  = Path.extname(path)
    base = Path.basename(path, ext)
    dir  = Path.dirname(path)
    Path.join(dir, "#{base}_processed#{ext}")
  end

  defp thumb_path(path, size) do
    ext  = Path.extname(path)
    base = Path.basename(path, ext)
    dir  = Path.dirname(path)
    Path.join(dir, "#{base}_#{size}w#{ext}")
  end
end
```

---

## Step 935: Video Processing

```elixir
defmodule MyApp.VideoProcessor do
  # Use ffmpeg via System.cmd

  def extract_thumbnail(video_path, timestamp \\ "00:00:05") do
    output = temp_path(".jpg")

    case System.cmd("ffmpeg", [
      "-i",     video_path,
      "-ss",    timestamp,
      "-vframes", "1",
      "-f",     "image2",
      output
    ], stderr_to_stdout: true) do
      {_, 0}    -> {:ok, output}
      {error, _} -> {:error, error}
    end
  end

  def transcode(source, format \\ "mp4") do
    output = temp_path(".#{format}")

    case System.cmd("ffmpeg", [
      "-i",     source,
      "-c:v",   "libx264",
      "-c:a",   "aac",
      "-movflags", "+faststart",
      output
    ], stderr_to_stdout: true) do
      {_, 0}     -> {:ok, output}
      {error, _} -> {:error, error}
    end
  end

  def get_duration(video_path) do
    case System.cmd("ffprobe", [
      "-v",           "quiet",
      "-print_format", "json",
      "-show_format",
      video_path
    ]) do
      {json, 0} ->
        duration = json |> Jason.decode!() |> get_in(["format", "duration"])
        {:ok, String.to_float(duration)}
      {error, _} ->
        {:error, error}
    end
  end

  defp temp_path(ext) do
    "/tmp/#{Ecto.UUID.generate()}#{ext}"
  end
end
```

---

## Step 936: Document Processing

```elixir
defmodule MyApp.DocumentProcessor do
  # PDF processing

  def extract_text(pdf_path) do
    case System.cmd("pdftotext", [pdf_path, "-"]) do
      {text, 0}  -> {:ok, text}
      {error, _} -> {:error, error}
    end
  end

  def page_count(pdf_path) do
    case System.cmd("pdfinfo", [pdf_path]) do
      {info, 0} ->
        pages = Regex.run(~r/Pages:\s+(\d+)/, info, capture: :all_but_first)
        {:ok, pages |> List.first() |> String.to_integer()}
      {error, _} ->
        {:error, error}
    end
  end

  # CSV processing with streaming
  def process_csv(file_path) do
    file_path
    |> File.stream!()
    |> CSV.decode!(headers: true)
    |> Stream.map(&normalize_row/1)
    |> Stream.reject(&is_nil/1)
  end

  def batch_import_csv(file_path) do
    process_csv(file_path)
    |> Stream.chunk_every(1000)
    |> Stream.each(fn batch ->
      MyApp.Repo.insert_all(MyApp.Record, batch,
        on_conflict: :replace_all,
        conflict_target: [:external_id]
      )
    end)
    |> Stream.run()
  end

  defp normalize_row(%{"name" => name, "email" => email}) when name != "" do
    %{name: name, email: String.downcase(email), inserted_at: DateTime.utc_now()}
  end
  defp normalize_row(_), do: nil
end
```

---

## Step 937: File Chunked Upload

```elixir
defmodule MyApp.ChunkedUpload do
  @chunk_ttl 3600  # 1 hour

  def init_upload(filename, total_chunks) do
    upload_id = Ecto.UUID.generate()
    
    Redix.command(:redix, ["HSET", upload_key(upload_id),
      "filename",     filename,
      "total_chunks", total_chunks,
      "received",     0
    ])
    Redix.command(:redix, ["EXPIRE", upload_key(upload_id), @chunk_ttl])
    
    {:ok, upload_id}
  end

  def upload_chunk(upload_id, chunk_number, data) do
    key = upload_key(upload_id)
    
    case Redix.command(:redix, ["HGET", key, "total_chunks"]) do
      {:ok, nil} -> {:error, :upload_not_found}
      {:ok, total} ->
        chunk_path = chunk_file_path(upload_id, chunk_number)
        File.write!(chunk_path, data)
        
        {:ok, received} = Redix.command(:redix, ["HINCRBY", key, "received", 1])
        
        if received == String.to_integer(total) do
          {:ok, :complete}
        else
          {:ok, :partial}
        end
    end
  end

  def assemble(upload_id) do
    key = upload_key(upload_id)
    {:ok, data} = Redix.command(:redix, ["HGETALL", key])
    info = Enum.chunk_every(data, 2) |> Map.new(fn [k, v] -> {k, v} end)
    
    total = String.to_integer(info["total_chunks"])
    final_path = final_file_path(upload_id, info["filename"])
    
    File.open!(final_path, [:write, :binary], fn file ->
      for i <- 0..(total - 1) do
        chunk = File.read!(chunk_file_path(upload_id, i))
        IO.binwrite(file, chunk)
        File.rm(chunk_file_path(upload_id, i))
      end
    end)
    
    Redix.command(:redix, ["DEL", key])
    {:ok, final_path}
  end

  defp upload_key(id),         do: "upload:#{id}"
  defp chunk_file_path(id, n), do: "/tmp/#{id}_chunk_#{n}"
  defp final_file_path(id, n), do: "/tmp/#{id}_#{n}"
end
```

---

## Step 938: Storage Backends

```elixir
defmodule MyApp.Storage do
  @behaviour MyApp.Storage.Adapter

  def upload(file) do
    adapter().upload(file)
  end

  def delete(url) do
    adapter().delete(url)
  end

  def url(key) do
    adapter().url(key)
  end

  defp adapter do
    Application.get_env(:my_app, :storage_adapter, MyApp.Storage.Local)
  end
end

defmodule MyApp.Storage.Local do
  @behaviour MyApp.Storage.Adapter

  @upload_dir "priv/static/uploads"

  def upload(%Plug.Upload{path: path, filename: filename}) do
    dest = Path.join(@upload_dir, unique_name(filename))
    File.cp!(path, dest)
    {:ok, "/uploads/#{Path.basename(dest)}"}
  end

  def delete(url) do
    path = Path.join("priv/static", url)
    File.rm(path)
    :ok
  end

  def url(key), do: "/uploads/#{key}"

  defp unique_name(filename) do
    ext = Path.extname(filename)
    "#{Ecto.UUID.generate()}#{ext}"
  end
end

defmodule MyApp.Storage.Cloudinary do
  @behaviour MyApp.Storage.Adapter

  def upload(%Plug.Upload{path: path, content_type: type}) do
    key = System.get_env("CLOUDINARY_API_KEY")
    secret = System.get_env("CLOUDINARY_API_SECRET")
    cloud  = System.get_env("CLOUDINARY_CLOUD_NAME")
    
    timestamp  = System.os_time(:second)
    signature  = sign(timestamp, key, secret)

    case Req.post("https://api.cloudinary.com/v1_1/#{cloud}/image/upload",
           form: [file: File.read!(path), api_key: key, timestamp: timestamp, signature: signature]) do
      {:ok, %{status: 200, body: body}} -> {:ok, body["secure_url"]}
      {:error, reason}                  -> {:error, reason}
    end
  end

  defp sign(timestamp, key, secret) do
    :crypto.mac(:hmac, :sha256, secret, "timestamp=#{timestamp}")
    |> Base.encode16(case: :lower)
  end

  def delete(_url), do: :ok
  def url(key),     do: key
end
```

---

## Step 939: File Streaming

```elixir
defmodule MyAppWeb.DownloadController do
  use MyAppWeb, :controller

  # Stream large file download
  def download(conn, %{"id" => id}) do
    attachment = MyApp.Attachments.get!(id)
    
    conn
    |> put_resp_content_type(attachment.content_type)
    |> put_resp_header("content-disposition", ~s(attachment; filename="#{attachment.filename}"))
    |> put_resp_header("content-length", to_string(attachment.size))
    |> send_chunked(200)
    |> stream_file(attachment.path)
  end

  defp stream_file(conn, path) do
    path
    |> File.stream!([], 4096)
    |> Enum.reduce_while(conn, fn chunk, conn ->
      case chunk(conn, chunk) do
        {:ok, conn}    -> {:cont, conn}
        {:error, :closed} -> {:halt, conn}
      end
    end)
  end

  # Range requests (for video streaming)
  def video(conn, %{"id" => id}) do
    video = MyApp.Videos.get!(id)
    file_size = File.stat!(video.path).size

    case get_req_header(conn, "range") do
      ["bytes=" <> range_spec] ->
        {start_byte, end_byte} = parse_range(range_spec, file_size)
        length = end_byte - start_byte + 1

        conn
        |> put_resp_content_type("video/mp4")
        |> put_resp_header("content-range", "bytes #{start_byte}-#{end_byte}/#{file_size}")
        |> put_resp_header("accept-ranges", "bytes")
        |> put_resp_header("content-length", to_string(length))
        |> send_file(206, video.path, start_byte, length)
      [] ->
        send_file(conn, 200, video.path)
    end
  end

  defp parse_range(spec, file_size) do
    [s, e] = String.split(spec, "-")
    start_byte = String.to_integer(s)
    end_byte   = if e == "", do: file_size - 1, else: String.to_integer(e)
    {start_byte, min(end_byte, file_size - 1)}
  end
end
```

---

## Step 940: File Virus Scanning

```elixir
defmodule MyApp.VirusScanner do
  # ClamAV integration for virus scanning

  def scan(file_path) do
    case System.cmd("clamdscan", ["--no-summary", file_path]) do
      {_, 0}  -> :clean
      {_, 1}  -> {:infected, "Virus found"}
      {error, 2} ->
        Logger.error("ClamAV error: #{error}")
        {:error, :scanner_error}
    end
  end

  def scan_upload(%Plug.Upload{path: path}) do
    case scan(path) do
      :clean         -> :ok
      {:infected, _} -> {:error, :virus_detected}
      {:error, e}    -> {:error, e}
    end
  end

  # Async scanning for large files
  def scan_async(file_path, callback) do
    Task.start(fn ->
      result = scan(file_path)
      callback.(result)
    end)
  end
end

# Plug to scan uploads
defmodule MyAppWeb.Plugs.VirusScan do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    uploads = conn.params
      |> Enum.filter(fn {_, v} -> match?(%Plug.Upload{}, v) end)
      |> Enum.map(fn {key, upload} -> {key, upload} end)

    case scan_all(uploads) do
      :ok ->
        conn
      {:error, :virus_detected} ->
        conn
        |> put_status(422)
        |> Phoenix.Controller.json(%{error: "virus_detected"})
        |> halt()
    end
  end

  defp scan_all([]), do: :ok
  defp scan_all([{_key, upload} | rest]) do
    case MyApp.VirusScanner.scan_upload(upload) do
      :ok             -> scan_all(rest)
      {:error, reason} -> {:error, reason}
    end
  end
end
```

---

## สรุป Part 87

✅ **Step 931** - File upload handling  
✅ **Step 932** - LiveView file upload  
✅ **Step 933** - S3 / object storage  
✅ **Step 934** - Image processing  
✅ **Step 935** - Video processing  
✅ **Step 936** - Document processing  
✅ **Step 937** - Chunked upload  
✅ **Step 938** - Storage backends  
✅ **Step 939** - File streaming  
✅ **Step 940** - Virus scanning  

➡️ [Part 88: Email & Notifications](./part-88-email.md)
