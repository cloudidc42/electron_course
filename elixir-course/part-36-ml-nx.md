# Part 36: Machine Learning ด้วย Nx และ Axon (Steps 391-410)

## Step 391: Nx (Numerical Elixir)

```elixir
# mix.exs
{:nx, "~> 0.7"},
{:exla, "~> 0.7"},   # XLA backend (GPU/CPU acceleration)
{:axon, "~> 0.6"}    # neural networks

# config/config.exs
config :nx, default_backend: EXLA.Backend

# Tensor basics
import Nx

# Create tensors
t1 = Nx.tensor([1, 2, 3, 4])              # vector
t2 = Nx.tensor([[1, 2], [3, 4]])          # matrix
t3 = Nx.tensor([[[1, 2], [3, 4]]])        # 3D tensor

# Tensor info
Nx.shape(t2)    # => {2, 2}
Nx.rank(t2)     # => 2
Nx.type(t2)     # => {:s, 64}  (signed int 64)
Nx.size(t2)     # => 4

# Type casting
Nx.as_type(t2, {:f, 32})  # float32

# Operations
Nx.add(t1, 10)      # broadcast
Nx.multiply(t1, t2) # element-wise (if shapes match)
Nx.dot(t2, t2)      # matrix multiply

# Aggregations
Nx.sum(t2)                  # sum all
Nx.sum(t2, axes: [0])       # sum along axis 0
Nx.mean(t2)
Nx.argmax(t1)               # index of max value
```

---

## Step 392: Numerical Operations

```elixir
defmodule Statistics do
  import Nx
  
  def normalize(tensor) do
    mean = Nx.mean(tensor)
    std  = Nx.standard_deviation(tensor)
    Nx.divide(Nx.subtract(tensor, mean), std)
  end
  
  def correlation(x, y) do
    n    = Nx.size(x) |> Nx.tensor()
    xm   = Nx.subtract(x, Nx.mean(x))
    ym   = Nx.subtract(y, Nx.mean(y))
    
    num  = Nx.sum(Nx.multiply(xm, ym))
    denom = Nx.sqrt(Nx.multiply(Nx.sum(Nx.power(xm, 2)), Nx.sum(Nx.power(ym, 2))))
    
    Nx.divide(num, denom)
  end
  
  def euclidean_distance(v1, v2) do
    v1
    |> Nx.subtract(v2)
    |> Nx.power(2)
    |> Nx.sum()
    |> Nx.sqrt()
  end
  
  # Matrix operations
  def pca(data, n_components) do
    # 1. Center data
    mean = Nx.mean(data, axes: [0])
    centered = Nx.subtract(data, mean)
    
    # 2. Compute covariance matrix
    n = Nx.axis_size(data, 0) |> Nx.tensor({:f, 64})
    cov = Nx.divide(Nx.dot(Nx.transpose(centered), centered), n)
    
    # 3. SVD decomposition
    {_u, s, vt} = Nx.LinAlg.svd(cov)
    
    # 4. Take top n_components
    Nx.slice_along_axis(vt, 0, n_components, axis: 0)
  end
end
```

---

## Step 393: Defn (JIT-Compiled Functions)

```elixir
defmodule MyML do
  import Nx.Defn
  
  # defn compiles to optimized code (XLA, CUDA, etc.)
  defn sigmoid(x) do
    Nx.divide(1, Nx.add(1, Nx.exp(Nx.negate(x))))
  end
  
  defn relu(x) do
    Nx.max(x, 0)
  end
  
  defn softmax(x) do
    e = Nx.exp(x)
    Nx.divide(e, Nx.sum(e))
  end
  
  defn linear_forward(x, w, b) do
    Nx.add(Nx.dot(x, w), b)
  end
  
  defn mse_loss(predictions, targets) do
    diff = Nx.subtract(predictions, targets)
    Nx.mean(Nx.power(diff, 2))
  end
  
  defn cross_entropy_loss(logits, labels) do
    log_softmax = logits
      |> softmax()
      |> Nx.log()
    
    Nx.negate(Nx.mean(Nx.multiply(labels, log_softmax)))
  end
  
  # Gradient computation
  def compute_gradient(x, w, b, y) do
    grad_fn = Nx.Defn.grad(fn {w, b} ->
      y_pred = linear_forward(x, w, b)
      mse_loss(y_pred, y)
    end)
    
    grad_fn.({w, b})
  end
end
```

---

## Step 394: Axon Neural Network

```elixir
defmodule MyApp.NeuralNet do
  # Define model architecture
  def build_classifier(input_size, hidden_size, output_size) do
    Axon.input("features", shape: {nil, input_size})
    |> Axon.dense(hidden_size)
    |> Axon.relu()
    |> Axon.dropout(rate: 0.3)
    |> Axon.dense(hidden_size)
    |> Axon.relu()
    |> Axon.dense(output_size)
    |> Axon.softmax()
  end
  
  # Build regression model
  def build_regressor(input_size) do
    Axon.input("features", shape: {nil, input_size})
    |> Axon.dense(64, activation: :relu)
    |> Axon.batch_norm()
    |> Axon.dense(32, activation: :relu)
    |> Axon.dense(1)
  end
  
  # Display model summary
  def summary(model, input_size) do
    template = Nx.template({1, input_size}, :f32)
    Axon.Display.as_table(model, Axon.Templates.input(template))
  end
end
```

---

## Step 395: Training

```elixir
defmodule MyApp.Trainer do
  def train(model, train_data, opts \\ []) do
    epochs     = Keyword.get(opts, :epochs, 10)
    batch_size = Keyword.get(opts, :batch_size, 32)
    lr         = Keyword.get(opts, :lr, 0.001)
    
    {init_fn, step_fn} = Axon.Loop.train_step(model,
      Axon.Losses.categorical_cross_entropy([], reduction: :mean),
      Axon.Optimizers.adam(lr)
    )
    
    model
    |> Axon.Loop.trainer(:categorical_cross_entropy, :adam)
    |> Axon.Loop.metric(:accuracy)
    |> Axon.Loop.run(train_data, %{}, epochs: epochs, batch_size: batch_size)
  end
  
  def evaluate(model, params, test_data) do
    model
    |> Axon.Loop.evaluator()
    |> Axon.Loop.metric(:accuracy)
    |> Axon.Loop.run(test_data, params)
  end
  
  def predict(model, params, input) do
    {_init_fn, predict_fn} = Axon.build(model)
    predict_fn.(params, %{"features" => input})
  end
end
```

---

## Step 396: MNIST Classifier

```elixir
defmodule MyApp.MNIST do
  @moduledoc "Train digit classifier on MNIST dataset"
  
  def run do
    IO.puts("Loading MNIST data...")
    {train, test} = load_data()
    
    IO.puts("Building model...")
    model = build_model()
    
    IO.puts("Training...")
    params = train_model(model, train)
    
    IO.puts("Evaluating...")
    accuracy = evaluate_model(model, params, test)
    
    IO.puts("Test accuracy: #{Float.round(accuracy * 100, 2)}%")
    
    {model, params}
  end
  
  defp load_data do
    # Assumes MNIST downloaded to priv/data/
    # Returns batches of {images, labels}
    train_images = File.read!("priv/data/train-images")
    train_labels = File.read!("priv/data/train-labels")
    
    images = parse_images(train_images)
    labels = parse_labels(train_labels)
    
    data = Enum.zip(images, labels)
    split = round(length(data) * 0.8)
    
    {Enum.take(data, split), Enum.drop(data, split)}
  end
  
  defp build_model do
    Axon.input("images", shape: {nil, 784})
    |> Axon.dense(256, activation: :relu)
    |> Axon.dropout(rate: 0.5)
    |> Axon.dense(128, activation: :relu)
    |> Axon.dense(10, activation: :softmax)
  end
  
  defp train_model(model, data) do
    model
    |> Axon.Loop.trainer(:categorical_cross_entropy, :adam)
    |> Axon.Loop.metric(:accuracy)
    |> Axon.Loop.run(data, %{}, epochs: 5)
  end
  
  defp evaluate_model(model, params, test_data) do
    results = model
      |> Axon.Loop.evaluator()
      |> Axon.Loop.metric(:accuracy)
      |> Axon.Loop.run(test_data, params)
    
    results[:accuracy]
  end
end
```

---

## Step 397: Transfer Learning ด้วย Bumblebee

```elixir
# mix.exs
{:bumblebee, "~> 0.5"},
{:exla, "~> 0.7"}

defmodule MyApp.TextClassifier do
  @moduledoc "Sentiment analysis using BERT"
  
  def setup do
    {:ok, bert}      = Bumblebee.load_model({:hf, "bert-base-uncased"})
    {:ok, tokenizer} = Bumblebee.load_tokenizer({:hf, "bert-base-uncased"})
    
    serving = Bumblebee.Text.text_classification(bert, tokenizer,
      top_k: 1,
      compile: [batch_size: 4, sequence_length: 512],
      defn_options: [compiler: EXLA]
    )
    
    Nx.Serving.run(serving, [])  # warm up
    
    {serving, tokenizer}
  end
  
  def predict({serving, _tokenizer}, texts) when is_list(texts) do
    Nx.Serving.run(serving, texts)
  end
  
  def predict({serving, _tokenizer}, text) do
    Nx.Serving.run(serving, text)
  end
end

defmodule MyApp.ImageClassifier do
  def setup do
    {:ok, resnet} = Bumblebee.load_model({:hf, "microsoft/resnet-50"})
    {:ok, featurizer} = Bumblebee.load_featurizer({:hf, "microsoft/resnet-50"})
    
    serving = Bumblebee.Vision.image_classification(resnet, featurizer,
      top_k: 5,
      compile: [batch_size: 4],
      defn_options: [compiler: EXLA]
    )
    
    serving
  end
  
  def classify(serving, image_path) do
    image = StbImage.read_file!(image_path)
    Nx.Serving.run(serving, image)
  end
end
```

---

## Step 398: Recommendation System

```elixir
defmodule MyApp.Recommender do
  import Nx.Defn
  
  # Collaborative filtering
  def train(user_item_matrix, n_factors \\ 20, epochs \\ 100, lr \\ 0.01) do
    {n_users, n_items} = Nx.shape(user_item_matrix)
    
    # Initialize embeddings
    user_emb  = Nx.random_normal({n_users, n_factors}) |> Nx.multiply(0.01)
    item_emb  = Nx.random_normal({n_items, n_factors}) |> Nx.multiply(0.01)
    user_bias = Nx.zeros({n_users, 1})
    item_bias = Nx.zeros({1, n_items})
    
    mask = Nx.not_equal(user_item_matrix, 0)
    
    params = {user_emb, item_emb, user_bias, item_bias}
    
    Enum.reduce(1..epochs, params, fn epoch, {ue, ie, ub, ib} ->
      {loss, grads} = compute_loss_and_grads(ue, ie, ub, ib, user_item_matrix, mask)
      
      if rem(epoch, 10) == 0 do
        IO.puts("Epoch #{epoch}, Loss: #{Nx.to_number(loss)}")
      end
      
      update_params({ue, ie, ub, ib}, grads, lr)
    end)
  end
  
  defn compute_predictions(user_emb, item_emb, user_bias, item_bias) do
    Nx.add(
      Nx.add(Nx.dot(user_emb, Nx.transpose(item_emb)), user_bias),
      item_bias
    )
  end
  
  def recommend(params, user_id, top_n \\ 10) do
    {user_emb, item_emb, user_bias, item_bias} = params
    
    scores = compute_predictions(user_emb, item_emb, user_bias, item_bias)
    user_scores = scores[user_id]
    
    user_scores
    |> Nx.argsort(direction: :desc)
    |> Nx.slice([0], [top_n])
    |> Nx.to_flat_list()
  end
end
```

---

## Step 399: Model Serving

```elixir
defmodule MyApp.MLServer do
  use GenServer
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def predict(input) do
    GenServer.call(__MODULE__, {:predict, input}, 10_000)
  end
  
  def init(_opts) do
    IO.puts("Loading ML models...")
    
    # Load and warm up models
    sentiment_model = load_sentiment_model()
    image_classifier = load_image_classifier()
    
    IO.puts("ML models ready")
    
    {:ok, %{
      sentiment: sentiment_model,
      image:     image_classifier
    }}
  end
  
  def handle_call({:predict, %{type: :sentiment, text: text}}, _from, state) do
    result = Nx.Serving.run(state.sentiment, text)
    {:reply, {:ok, result}, state}
  end
  
  def handle_call({:predict, %{type: :image, data: data}}, _from, state) do
    result = Nx.Serving.run(state.image, data)
    {:reply, {:ok, result}, state}
  end
  
  defp load_sentiment_model do
    {:ok, model} = Bumblebee.load_model({:hf, "distilbert-base-uncased-finetuned-sst-2-english"})
    {:ok, tokenizer} = Bumblebee.load_tokenizer({:hf, "distilbert-base-uncased"})
    
    Bumblebee.Text.text_classification(model, tokenizer,
      compile: [batch_size: 8, sequence_length: 128],
      defn_options: [compiler: EXLA]
    )
  end
  
  defp load_image_classifier, do: nil  # placeholder
end

# HTTP endpoint
defmodule MyAppWeb.MLController do
  use MyAppWeb, :controller
  
  def sentiment(conn, %{"text" => text}) do
    case MyApp.MLServer.predict(%{type: :sentiment, text: text}) do
      {:ok, %{predictions: [%{label: label, score: score} | _]}} ->
        json(conn, %{label: label, confidence: score})
      {:error, reason} ->
        conn |> put_status(:internal_server_error) |> json(%{error: reason})
    end
  end
end
```

---

## Step 400: ML Pipeline

```elixir
defmodule MyApp.MLPipeline do
  @moduledoc "End-to-end ML pipeline: ingest → preprocess → train → evaluate → deploy"
  
  def run(dataset_path) do
    dataset_path
    |> load_data()
    |> preprocess()
    |> split_train_test(0.8)
    |> train_and_evaluate()
    |> maybe_deploy()
  end
  
  defp load_data(path) do
    path
    |> File.stream!()
    |> CSV.decode!(headers: true)
    |> Enum.to_list()
  end
  
  defp preprocess(data) do
    features = Enum.map(data, fn row ->
      [row["age"], row["income"], row["score"]]
      |> Enum.map(&String.to_float/1)
    end)
    
    labels = Enum.map(data, fn row ->
      if row["churn"] == "1", do: 1, else: 0
    end)
    
    x = features |> Nx.tensor({:f, 32})
    y = labels   |> Nx.tensor({:s, 32})
    
    # Normalize features
    {x_mean, x_std} = {Nx.mean(x, axes: [0]), Nx.standard_deviation(x, axes: [0])}
    x_normalized = Nx.divide(Nx.subtract(x, x_mean), Nx.add(x_std, 1.0e-8))
    
    {x_normalized, y}
  end
  
  defp split_train_test({x, y}, ratio) do
    n = Nx.axis_size(x, 0)
    split = round(n * ratio)
    
    x_train = Nx.slice_along_axis(x, 0, split, axis: 0)
    x_test  = Nx.slice_along_axis(x, split, n - split, axis: 0)
    y_train = Nx.slice_along_axis(y, 0, split, axis: 0)
    y_test  = Nx.slice_along_axis(y, split, n - split, axis: 0)
    
    {{x_train, y_train}, {x_test, y_test}}
  end
  
  defp train_and_evaluate({{x_train, y_train}, {x_test, y_test}}) do
    model = build_model(Nx.axis_size(x_train, 1))
    
    train_data = [{x_train, y_train}]
    params = MyApp.Trainer.train(model, train_data, epochs: 20)
    
    test_data = [{x_test, y_test}]
    accuracy = MyApp.Trainer.evaluate(model, params, test_data)
    
    IO.puts("Model accuracy: #{Float.round(accuracy * 100, 2)}%")
    
    {model, params, accuracy}
  end
  
  defp maybe_deploy({model, params, accuracy}) do
    if accuracy > 0.85 do
      IO.puts("Deploying model with #{accuracy} accuracy...")
      MyApp.ModelRegistry.deploy(model, params, %{accuracy: accuracy})
      {:deployed, accuracy}
    else
      IO.puts("Model accuracy #{accuracy} below threshold, not deploying")
      {:not_deployed, accuracy}
    end
  end
  
  defp build_model(input_size) do
    Axon.input("features", shape: {nil, input_size})
    |> Axon.dense(64, activation: :relu)
    |> Axon.dropout(rate: 0.3)
    |> Axon.dense(32, activation: :relu)
    |> Axon.dense(2, activation: :softmax)
  end
end
```

---

## สรุป Part 36

✅ **Step 391** - Nx tensors  
✅ **Step 392** - Numerical operations  
✅ **Step 393** - Defn JIT compilation  
✅ **Step 394** - Axon neural networks  
✅ **Step 395** - Training loops  
✅ **Step 396** - MNIST classifier  
✅ **Step 397** - Transfer learning (Bumblebee)  
✅ **Step 398** - Recommendation system  
✅ **Step 399** - Model serving  
✅ **Step 400** - ML pipeline  

➡️ [Part 37: Nerves - Embedded Systems](./part-37-nerves.md)
