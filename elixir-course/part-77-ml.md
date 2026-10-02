# Part 77: Machine Learning & Data Science (Steps 831-850)

## Step 831: Nx - Numerical Elixir

```elixir
# mix.exs: {:nx, "~> 0.7"}, {:exla, "~> 0.7"}

defmodule MyApp.ML.Basics do
  import Nx

  def tensor_operations do
    # Create tensors
    a = Nx.tensor([1, 2, 3, 4, 5])
    b = Nx.tensor([[1, 2], [3, 4], [5, 6]])

    # Operations
    sum     = Nx.sum(a)               # 15
    mean    = Nx.mean(a)              # 3.0
    product = Nx.multiply(a, 2)       # [2, 4, 6, 8, 10]
    dot     = Nx.dot(a, a)            # 55

    # Matrix operations
    transposed   = Nx.transpose(b)             # [[1,3,5],[2,4,6]]
    matmul_input = Nx.tensor([[1, 2, 3]])
    matmul       = Nx.dot(matmul_input, b)     # [[22, 28]]

    # GPU acceleration with EXLA
    Nx.default_backend(EXLA.Backend)
    large_tensor = Nx.random_uniform({1000, 1000})
    result = Nx.dot(large_tensor, large_tensor)  # runs on GPU if available

    %{sum: sum, mean: mean, product: product}
  end

  def normalize(tensor) do
    mean = Nx.mean(tensor)
    std  = Nx.standard_deviation(tensor)
    Nx.divide(Nx.subtract(tensor, mean), std)
  end

  def sigmoid(x) do
    Nx.divide(1, Nx.add(1, Nx.exp(Nx.negate(x))))
  end

  def relu(x) do
    Nx.max(x, 0)
  end
end
```

---

## Step 832: Axon Neural Networks

```elixir
# mix.exs: {:axon, "~> 0.6"}

defmodule MyApp.ML.NeuralNet do
  def build_model(input_shape, num_classes) do
    Axon.input("input", shape: input_shape)
    |> Axon.dense(128)
    |> Axon.batch_norm()
    |> Axon.relu()
    |> Axon.dropout(rate: 0.3)
    |> Axon.dense(64)
    |> Axon.relu()
    |> Axon.dense(num_classes)
    |> Axon.softmax()
  end

  def train(model, train_data, opts \\ []) do
    epochs    = Keyword.get(opts, :epochs, 10)
    optimizer = Axon.Optimizers.adam(0.001)
    loss      = :categorical_cross_entropy

    model
    |> Axon.Loop.trainer(loss, optimizer)
    |> Axon.Loop.metric(:accuracy)
    |> Axon.Loop.run(train_data, %{}, epochs: epochs, compiler: EXLA)
  end

  def predict(model, params, input) do
    Axon.predict(model, params, input)
  end
end

# Example: image classifier
defmodule MyApp.ML.ImageClassifier do
  def cnn_model do
    Axon.input("image", shape: {nil, 3, 32, 32})
    |> Axon.conv(32, kernel_size: {3, 3}, padding: :same)
    |> Axon.batch_norm()
    |> Axon.relu()
    |> Axon.max_pool(kernel_size: {2, 2})
    |> Axon.conv(64, kernel_size: {3, 3}, padding: :same)
    |> Axon.batch_norm()
    |> Axon.relu()
    |> Axon.max_pool(kernel_size: {2, 2})
    |> Axon.flatten()
    |> Axon.dense(256)
    |> Axon.dropout(rate: 0.5)
    |> Axon.dense(10)
    |> Axon.softmax()
  end
end
```

---

## Step 833: Bumblebee (HuggingFace models)

```elixir
# mix.exs: {:bumblebee, "~> 0.5"}, {:nx, "~> 0.7"}, {:exla, "~> 0.7"}

defmodule MyApp.ML.TextClassifier do
  def load_sentiment_model do
    {:ok, model_info} = Bumblebee.load_model({:hf, "distilbert-base-uncased-finetuned-sst-2-english"})
    {:ok, tokenizer}  = Bumblebee.load_tokenizer({:hf, "distilbert-base-uncased"})

    Bumblebee.Text.text_classification(model_info, tokenizer,
      compile: [batch_size: 1, sequence_length: 64],
      defn_options: [compiler: EXLA]
    )
  end

  def classify_sentiment(text, serving) do
    result = Nx.Serving.run(serving, text)
    
    %{
      label:      result.predictions |> Enum.max_by(& &1.score) |> Map.get(:label),
      confidence: result.predictions |> Enum.max_by(& &1.score) |> Map.get(:score)
    }
  end
end

defmodule MyApp.ML.Embeddings do
  # Generate text embeddings with Bumblebee

  def load_embedding_model do
    {:ok, model_info} = Bumblebee.load_model({:hf, "sentence-transformers/all-MiniLM-L6-v2"})
    {:ok, tokenizer}  = Bumblebee.load_tokenizer({:hf, "sentence-transformers/all-MiniLM-L6-v2"})

    Bumblebee.Text.TextEmbedding.text_embedding(model_info, tokenizer,
      compile: [batch_size: 4, sequence_length: 128],
      defn_options: [compiler: EXLA]
    )
  end

  def embed(texts, serving) when is_list(texts) do
    Nx.Serving.run(serving, texts)
    |> Map.get(:embedding)
    |> Nx.to_list()
  end

  def embed(text, serving) do
    embed([text], serving) |> List.first()
  end
end
```

---

## Step 834: Scholar (ML Algorithms)

```elixir
# mix.exs: {:scholar, "~> 0.3"}

defmodule MyApp.ML.Clustering do
  def kmeans_cluster(data, k \\ 3) do
    tensor = Nx.tensor(data)
    
    # K-Means clustering
    {centroids, labels} = Scholar.Cluster.KMeans.fit(tensor, num_clusters: k)
    
    %{
      centroids: Nx.to_list(centroids),
      labels:    Nx.to_list(labels),
      k:         k
    }
  end

  def pca_reduce(data, n_components \\ 2) do
    tensor = Nx.tensor(data)
    
    model = Scholar.Decomposition.PCA.fit(tensor, num_components: n_components)
    reduced = Scholar.Decomposition.PCA.transform(model, tensor)
    
    Nx.to_list(reduced)
  end
end

defmodule MyApp.ML.Regression do
  def linear_regression(x, y) do
    x_tensor = Nx.tensor(x)
    y_tensor = Nx.tensor(y)
    
    model = Scholar.Linear.LinearRegression.fit(x_tensor, y_tensor)
    
    %{
      coefficients: Nx.to_list(model.coefficients),
      intercept:    Nx.to_number(model.intercept),
      model:        model
    }
  end

  def predict(%{model: model}, x_new) do
    Scholar.Linear.LinearRegression.predict(model, Nx.tensor(x_new))
    |> Nx.to_list()
  end

  def r_squared(%{model: model}, x, y) do
    x_t     = Nx.tensor(x)
    y_t     = Nx.tensor(y)
    y_pred  = Scholar.Linear.LinearRegression.predict(model, x_t)
    
    Scholar.Metrics.Regression.r2_score(y_t, y_pred)
    |> Nx.to_number()
  end
end
```

---

## Step 835: Recommendation Engine

```elixir
defmodule MyApp.ML.Recommendations do
  # Collaborative filtering using matrix factorization

  def train(user_item_matrix, opts \\ []) do
    factors = Keyword.get(opts, :factors, 20)
    epochs  = Keyword.get(opts, :epochs, 50)
    lr      = Keyword.get(opts, :learning_rate, 0.01)

    matrix = Nx.tensor(user_item_matrix)
    {num_users, num_items} = Nx.shape(matrix)

    # Random initialization
    user_factors = Nx.random_normal({num_users, factors})
    item_factors = Nx.random_normal({num_items, factors})

    {user_f, item_f, _loss} =
      Enum.reduce(1..epochs, {user_factors, item_factors, nil}, fn _epoch, {uf, itf, _} ->
        predicted = Nx.dot(uf, Nx.transpose(itf))
        error     = Nx.subtract(matrix, predicted)
        
        # Gradient descent
        new_uf  = Nx.add(uf,  Nx.multiply(lr, Nx.dot(error, itf)))
        new_itf = Nx.add(itf, Nx.multiply(lr, Nx.dot(Nx.transpose(error), uf)))
        
        loss = error |> Nx.pow(2) |> Nx.mean() |> Nx.to_number()
        {new_uf, new_itf, loss}
      end)

    %{user_factors: user_f, item_factors: item_f}
  end

  def recommend(%{user_factors: uf, item_factors: itf}, user_id, n \\ 10) do
    user_vec  = Nx.slice(uf, [user_id, 0], [1, Nx.axis_size(uf, 1)])
    scores    = Nx.dot(user_vec, Nx.transpose(itf)) |> Nx.flatten()
    
    scores
    |> Nx.argsort(direction: :desc)
    |> Nx.slice([0], [n])
    |> Nx.to_list()
  end
end
```

---

## Step 836: Time Series Analysis

```elixir
defmodule MyApp.ML.TimeSeries do
  import Nx

  def moving_average(series, window) do
    tensor = Nx.tensor(series)
    n      = Nx.axis_size(tensor, 0)
    
    Enum.map(window..(n - 1), fn i ->
      Nx.slice(tensor, [i - window + 1], [window])
      |> Nx.mean()
      |> Nx.to_number()
    end)
  end

  def exponential_smoothing(series, alpha \\ 0.3) do
    [first | rest] = series
    
    Enum.reduce(rest, [first], fn value, [prev | _] = smoothed ->
      new = alpha * value + (1 - alpha) * prev
      [new | smoothed]
    end)
    |> Enum.reverse()
  end

  def detect_anomalies(series, threshold_std \\ 2.0) do
    tensor = Nx.tensor(series)
    mean   = Nx.mean(tensor) |> Nx.to_number()
    std    = Nx.standard_deviation(tensor) |> Nx.to_number()

    series
    |> Enum.with_index()
    |> Enum.filter(fn {value, _idx} ->
      abs(value - mean) > threshold_std * std
    end)
    |> Enum.map(fn {value, idx} ->
      %{index: idx, value: value, z_score: (value - mean) / std}
    end)
  end

  def seasonal_decompose(series, period) do
    n = length(series)

    # Trend: moving average
    trend = moving_average(series, period)

    # Seasonal component
    seasonal = Enum.chunk_every(series, period)
      |> Enum.zip()
      |> Enum.map(fn season_values ->
        Tuple.to_list(season_values)
        |> Enum.sum()
        |> Kernel./(tuple_size(season_values))
      end)

    %{trend: trend, seasonal: seasonal}
  end
end
```

---

## Step 837: Feature Engineering

```elixir
defmodule MyApp.ML.Features do
  def encode_categorical(values) do
    unique = Enum.uniq(values)
    index  = Enum.with_index(unique) |> Map.new()
    
    encoded = Enum.map(values, &Map.fetch!(index, &1))
    {encoded, index}
  end

  def one_hot_encode(values) do
    {encoded, index} = encode_categorical(values)
    n_classes = map_size(index)
    
    Enum.map(encoded, fn i ->
      Nx.broadcast(0, {n_classes})
      |> Nx.put_slice([i], Nx.tensor([1]))
      |> Nx.to_list()
    end)
  end

  def normalize_minmax(values) do
    min_v = Enum.min(values)
    max_v = Enum.max(values)
    range = max_v - min_v

    if range == 0 do
      Enum.map(values, fn _ -> 0.0 end)
    else
      Enum.map(values, fn v -> (v - min_v) / range end)
    end
  end

  def standardize(values) do
    n    = length(values)
    mean = Enum.sum(values) / n
    std  = :math.sqrt(Enum.reduce(values, 0, fn v, acc ->
      acc + :math.pow(v - mean, 2)
    end) / n)

    if std == 0 do
      Enum.map(values, fn _ -> 0.0 end)
    else
      Enum.map(values, fn v -> (v - mean) / std end)
    end
  end

  def polynomial_features(x, degree \\ 2) do
    Enum.flat_map(1..degree, fn d ->
      Enum.map(x, fn v -> :math.pow(v, d) end)
    end)
  end
end
```

---

## Step 838: Model Serving

```elixir
defmodule MyApp.ML.ModelServer do
  use GenServer

  def start_link(model_path) do
    GenServer.start_link(__MODULE__, model_path, name: __MODULE__)
  end

  def predict(input) do
    GenServer.call(__MODULE__, {:predict, input}, 30_000)
  end

  def reload do
    GenServer.cast(__MODULE__, :reload)
  end

  def init(model_path) do
    {:ok, model, params} = load_model(model_path)
    {:ok, %{model: model, params: params, path: model_path}}
  end

  def handle_call({:predict, input}, _from, %{model: model, params: params} = state) do
    result = Axon.predict(model, params, input)
    {:reply, {:ok, result}, state}
  end

  def handle_cast(:reload, %{path: path} = state) do
    case load_model(path) do
      {:ok, model, params} ->
        Logger.info("Model reloaded from #{path}")
        {:noreply, %{state | model: model, params: params}}
      {:error, reason} ->
        Logger.error("Model reload failed: #{inspect(reason)}")
        {:noreply, state}
    end
  end

  defp load_model(path) do
    case File.read(path) do
      {:ok, binary} ->
        {model, params} = :erlang.binary_to_term(binary)
        {:ok, model, params}
      {:error, reason} ->
        {:error, reason}
    end
  end
end

# HTTP endpoint for model predictions
defmodule MyAppWeb.MLController do
  use MyAppWeb, :controller

  def predict(conn, %{"input" => input}) do
    tensor = Nx.tensor(input)
    
    case MyApp.ML.ModelServer.predict(tensor) do
      {:ok, result} ->
        json(conn, %{prediction: Nx.to_list(result)})
      {:error, reason} ->
        conn
        |> put_status(500)
        |> json(%{error: inspect(reason)})
    end
  end
end
```

---

## Step 839: A/B Testing

```elixir
defmodule MyApp.ABTest do
  # Statistically sound A/B testing

  def assign_variant(user_id, experiment_name, variants \\ [:control, :treatment]) do
    hash = :erlang.phash2("#{experiment_name}:#{user_id}", 100)
    variant_index = rem(hash, length(variants))
    Enum.at(variants, variant_index)
  end

  def record_conversion(experiment, variant, user_id, converted) do
    Redix.command(:redix, [
      "HINCRBY",
      "ab:#{experiment}:#{variant}",
      if(converted, do: "conversions", else: "impressions"),
      1
    ])
  end

  def results(experiment) do
    variants = get_variants(experiment)
    
    Enum.map(variants, fn variant ->
      {:ok, data} = Redix.command(:redix, ["HGETALL", "ab:#{experiment}:#{variant}"])
      stats = Enum.chunk_every(data, 2) |> Enum.into(%{}, fn [k, v] -> {k, String.to_integer(v)} end)
      
      impressions = Map.get(stats, "impressions", 0)
      conversions = Map.get(stats, "conversions", 0)
      rate        = if impressions > 0, do: conversions / impressions, else: 0.0

      %{
        variant:     variant,
        impressions: impressions,
        conversions: conversions,
        rate:        rate
      }
    end)
  end

  def statistical_significance(control, treatment, confidence \\ 0.95) do
    # Z-test for proportions
    p1 = control.conversions / control.impressions
    p2 = treatment.conversions / treatment.impressions
    n1 = control.impressions
    n2 = treatment.impressions
    
    p_pooled = (control.conversions + treatment.conversions) / (n1 + n2)
    se = :math.sqrt(p_pooled * (1 - p_pooled) * (1/n1 + 1/n2))
    
    z_score = if se > 0, do: (p2 - p1) / se, else: 0

    # p-value from z-score (two-tailed)
    p_value = 2 * (1 - normal_cdf(abs(z_score)))
    
    %{
      z_score:     z_score,
      p_value:     p_value,
      significant: p_value < (1 - confidence),
      lift:        if(p1 > 0, do: (p2 - p1) / p1 * 100, else: 0)
    }
  end

  defp normal_cdf(x) do
    0.5 * (1 + :math.erf(x / :math.sqrt(2)))
  end

  defp get_variants(_experiment), do: [:control, :treatment]
end
```

---

## Step 840: Model Evaluation Metrics

```elixir
defmodule MyApp.ML.Metrics do
  import Nx

  def accuracy(y_true, y_pred) do
    correct = Nx.equal(Nx.tensor(y_true), Nx.tensor(y_pred))
    Nx.mean(correct) |> Nx.to_number()
  end

  def precision(y_true, y_pred) do
    tp = true_positives(y_true, y_pred)
    fp = false_positives(y_true, y_pred)
    if tp + fp > 0, do: tp / (tp + fp), else: 0.0
  end

  def recall(y_true, y_pred) do
    tp = true_positives(y_true, y_pred)
    fn_ = false_negatives(y_true, y_pred)
    if tp + fn_ > 0, do: tp / (tp + fn_), else: 0.0
  end

  def f1_score(y_true, y_pred) do
    p = precision(y_true, y_pred)
    r = recall(y_true, y_pred)
    if p + r > 0, do: 2 * p * r / (p + r), else: 0.0
  end

  def mean_absolute_error(y_true, y_pred) do
    t = Nx.tensor(y_true)
    p = Nx.tensor(y_pred)
    Nx.subtract(t, p) |> Nx.abs() |> Nx.mean() |> Nx.to_number()
  end

  def root_mean_squared_error(y_true, y_pred) do
    t = Nx.tensor(y_true)
    p = Nx.tensor(y_pred)
    Nx.subtract(t, p) |> Nx.pow(2) |> Nx.mean() |> Nx.sqrt() |> Nx.to_number()
  end

  defp true_positives(y_true, y_pred) do
    Enum.zip(y_true, y_pred)
    |> Enum.count(fn {t, p} -> t == 1 and p == 1 end)
  end

  defp false_positives(y_true, y_pred) do
    Enum.zip(y_true, y_pred)
    |> Enum.count(fn {t, p} -> t == 0 and p == 1 end)
  end

  defp false_negatives(y_true, y_pred) do
    Enum.zip(y_true, y_pred)
    |> Enum.count(fn {t, p} -> t == 1 and p == 0 end)
  end
end
```

---

## สรุป Part 77

✅ **Step 831** - Nx tensors  
✅ **Step 832** - Axon neural networks  
✅ **Step 833** - Bumblebee HuggingFace models  
✅ **Step 834** - Scholar ML algorithms  
✅ **Step 835** - Recommendation engine  
✅ **Step 836** - Time series analysis  
✅ **Step 837** - Feature engineering  
✅ **Step 838** - Model serving  
✅ **Step 839** - A/B testing  
✅ **Step 840** - Evaluation metrics  

➡️ [Part 78: Advanced Testing Strategies](./part-78-testing-advanced.md)
