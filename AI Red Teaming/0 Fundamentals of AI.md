# Introduction to Machine Learning
`Artificial Intelligence` (`AI`) is a broad field focused on developing intelligent systems capable of performing tasks that typically require human intelligence.
	`Natural Language Processing` (`NLP`)
	`Computer Vision`
	`Robotics`

`Machine Learning` (`ML`) is a subfield of AI that focuses on enabling systems to learn from data and improve their performance on specific tasks without explicit programming.
	`Supervised Learning`: The algorithm learns from labeled data
		Image classification
		Spam detection
	`Unsupervised Learning`: The algorithm learns from unlabeled data.
		Anomaly detection
		Customer segmentation
	`Reinforcement Learning`: The algorithm learns through trial and error by interacting with an environment and receiving feedback.
		Games
		Robotics

`Deep Learning` (`DL`) is a subfield of ML that uses neural networks with multiple layers to learn and extract features from complex data.
	Characteristics:
		`Hierarchical Feature Learning`: DL models can learn hierarchical data representations
		`End-to-End Learning`: DL models can be trained end-to-end, meaning they can directly map raw input data to desired outputs
		`Scalability`: DL models can scale well with large datasets
	Common Types:
		`Convolutional Neural Networks` (`CNNs`): Specialized for image and video data
		`Recurrent Neural Networks` (`RNNs`): Designed for sequential data like text and speech, RNNs have loops that allow information to persist across time steps.
		`Transformers`: A recent advancement in DL, transformers are particularly effective for natural language processing tasks.

AI
	ML
		Neural Nets
			Deep Learning
### Math Refresher
```
╔══════════════════════════════════════════════════════════════════════════════╗
║                        MATH REFERENCE CHEAT SHEET                            ║
╚══════════════════════════════════════════════════════════════════════════════╝

─── BASIC ARITHMETIC ─────────────────────────────────────────────────────────

  *   Multiplication          3 * 4 = 12
  /   Division                10 / 2 = 5
  +   Addition                5 + 3 = 8
  -   Subtraction             9 - 4 = 5

─── ALGEBRAIC NOTATIONS ──────────────────────────────────────────────────────

  x_t     Subscript / index       x_t = value of x at time step t
          # A way to label a variable at a specific position in a sequence.
          # Think of it like an array index: x_0, x_1, x_2, ...
          # Common in time series, diffusion steps, RNNs, etc.

  x^n     Superscript / power     x^2 = x * x
          # Raises x to the nth power. Used in polynomials, loss functions,
          # growth/decay models, and anywhere you see squared error.

  ||v||   Euclidean (L2) Norm     sqrt(v_1^2 + v_2^2 + ... + v_n^2)
          # Measures the "straight-line" length of a vector.
          # Like finding the hypotenuse in n dimensions.

  ||v||_1 L1 Norm (Manhattan)     |v_1| + |v_2| + ... + |v_n|
          # Sum of absolute values. Called "Manhattan" because it measures
          # distance along grid lines (like city blocks).
          # Used in L1 regularization (Lasso) to encourage sparsity.

  ||v||_∞ L-inf Norm              max(|v_1|, |v_2|, ..., |v_n|)
          # The single largest absolute value in the vector.
          # Used in adversarial ML (e.g., L-inf perturbation budgets).

  Σ       Summation               Σ_{i=1}^{n} a_i = a_1 + a_2 + ... + a_n
          # Compact notation for "add up all these terms."
          # The bottom (i=1) is the start, the top (n) is the end.
          # Shows up everywhere: means, losses, series, etc.

─── LOGARITHMS & EXPONENTIALS ────────────────────────────────────────────────

  log2(x)   Log base 2            log2(8) = 3
  ln(x)     Natural log (base e)  ln(e^2) = 2
  e^x       Exponential (base e)  e^2 ≈ 7.389
  2^x       Exponential (base 2)  2^3 = 8

─── MATRIX & VECTOR OPERATIONS ──────────────────────────────────────────────

  A * v     Matrix-vector mult     [[1,2],[3,4]] * [5,6] = [17, 39]
            # Each row of A is dot-producted with v to produce one output value.
            # Fundamental to neural networks: every linear layer does this.
            # output = weights * input + bias

  A * B     Matrix-matrix mult     [[1,2],[3,4]] * [[5,6],[7,8]] = [[19,22],[43,50]]
            # Each element (i,j) in the result is the dot product of
            # row i of A and column j of B.
            # Used in layer-to-layer transformations in deep learning.

  A^T       Transpose              Swap rows ↔ columns
            [[1,2],[3,4]]^T = [[1,3],[2,4]]
            # Flips a matrix over its diagonal. Row 0 becomes column 0, etc.
            # Used in dot products (v^T * w), backprop, and data reshaping.

  A^{-1}    Inverse                A * A^{-1} = I (identity matrix)
            [[1,2],[3,4]]^{-1} = [[-2,1],[1.5,-0.5]]
            # The matrix equivalent of dividing by a number.
            # If A * x = b, then x = A^{-1} * b.
            # Only exists when det(A) != 0.

  det(A)    Determinant            det([[1,2],[3,4]]) = 1*4 - 2*3 = -2
            # A single scalar that summarizes key properties of a matrix.
            # det = 0 → matrix is singular (not invertible, rows are dependent).
            # det != 0 → matrix is invertible.
            # Also relates to how a transformation scales area/volume.

  tr(A)     Trace                  Sum of main diagonal elements
            tr([[1,2],[3,4]]) = 1 + 4 = 5
            # Quick shortcut: the trace equals the sum of eigenvalues.
            # Used in matrix calculus and some loss function derivations.

─── SET THEORY ───────────────────────────────────────────────────────────────

  |S|       Cardinality            |{1,2,3,4,5}| = 5
            # Simply "how many elements are in this set?"
            # Used in probability (e.g., |sample space|) and counting problems.

  A ∪ B     Union                  {1,2,3} ∪ {3,4,5} = {1,2,3,4,5}
            # Everything in A, B, or both. Think logical OR.
            # Duplicates are collapsed (3 appears once).

  A ∩ B     Intersection           {1,2,3} ∩ {3,4,5} = {3}
            # Only elements that appear in BOTH sets. Think logical AND.
            # Used in data filtering, confusion matrix overlap, IoU, etc.

  A^c       Complement             U={1..5}, A={1,2,3} → A^c = {4,5}
            # Everything NOT in A (relative to some universal set U).
            # P(A^c) = 1 - P(A). Handy for "probability of not happening."

─── COMPARISON OPERATORS ─────────────────────────────────────────────────────

  >=   Greater than or equal to    a >= b
  <=   Less than or equal to       a <= b
  ==   Equality                    a == b
  !=   Inequality                  a != b

─── EIGENVALUES & EIGENVECTORS ───────────────────────────────────────────────

  λ         Eigenvalue             A * v = λ * v
            # A scalar that tells you HOW MUCH the eigenvector is stretched
            # or squished when matrix A is applied.
            # Large λ → that direction captures a lot of variance.
            # In PCA, you pick the top-k eigenvalues to reduce dimensions.

  v         Eigenvector            A non-zero vector satisfying A * v = λ * v
            # The DIRECTION that doesn't change when A is applied —
            # it only gets scaled by λ.
            # In PCA, eigenvectors of the covariance matrix are the
            # principal components (axes of maximum variance in data).

─── FUNCTIONS & OPERATORS ────────────────────────────────────────────────────

  max(...)   Maximum               max(4, 7, 2) = 7
             # Returns the largest value. Used in ReLU (max(0, x)),
             # argmax for classification, and optimization objectives.

  min(...)   Minimum               min(4, 7, 2) = 2
             # Returns the smallest value. Used in min-max scaling,
             # clipping gradients, and optimization (minimizing loss).

  1 / x      Reciprocal            1 / 5 = 0.2
             # Flips a value. Used in learning rates, normalization
             # (e.g., 1/n for averaging), and inverse weighting.

  ...        Ellipsis              a_1 + a_2 + ... + a_n
             # "Pattern continues." Shorthand so you don't have to
             # write out every single term in a long sequence.

─── PROBABILITY & STATISTICS ─────────────────────────────────────────────────

  f(x)       Function notation     f(x) = x^2 + 2x + 1
             # "f of x" — a named rule that maps input x to an output.
             # The backbone of every model: input → function → prediction.

  P(x | y)   Conditional prob      Probability of x given y
              P(Output | Input)
             # "What's the chance of x, assuming y already happened?"
             # The basis of Bayes' theorem, classifiers, and generative models.
             # P(A|B) = P(A ∩ B) / P(B)

  E[X]       Expectation           E[X] = Σ x_i * P(x_i)
             # The weighted average — each outcome times its probability.
             # The "center of mass" of a distribution.
             # In ML, expected loss = what we're trying to minimize.

  Var(X)     Variance              Var(X) = E[(X - E[X])^2]
             # How spread out values are from the mean.
             # High variance → data is scattered. Low → tightly clustered.
             # Var = 0 means every value is identical.

  σ(X)       Standard deviation    σ(X) = sqrt(Var(X))
             # Same idea as variance but in the original units (not squared).
             # ~68% of data falls within ±1σ of the mean (normal dist).
             # Easier to interpret than variance.

  Cov(X,Y)   Covariance            Cov(X,Y) = E[(X - E[X])(Y - E[Y])]
             # Do X and Y move together (positive), opposite (negative),
             # or independently (≈0)?
             # Building block of the covariance matrix used in PCA.

  ρ(X,Y)     Correlation           ρ(X,Y) = Cov(X,Y) / (σ(X) * σ(Y))
             # Covariance normalized to [-1, 1].
             #  +1 = perfect positive linear relationship
             #   0 = no linear relationship
             #  -1 = perfect negative linear relationship
             # Easier to compare across different variable scales.

══════════════════════════════════════════════════════════════════════════════
```

# Supervised Learning Algorithms
`Supervised learning` algorithms form the cornerstone of many `Machine Learning` (`ML`) applications, enabling systems to learn from labeled data and make accurate predictions.

How it works:
	Fed large data sets of labeled examples, then train a model to predict labels for new, unseen examples.

Two Types of Learning Problems:
	`Classification`: In classification problems, the goal is to predict a categorical label.
	`Regression`: In regression problems, the goal is to predict a continuous value.

**Core Concepts:**

| Concept            | Description                                                                                                                               |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `Training Data`    | Labeled dataset used to train the ML model                                                                                                |
| `Features`         | Measurable properties of data that serve as input for the model                                                                           |
| `Labels`           | Known outcomes associated with each data point in a training set                                                                          |
| `Model`            | Mathematical representation of the relationship between features and labels                                                               |
| `Training`         | Process of feeding training data to the algorithm and adjusting model parameters                                                          |
| `Prediction`       | Model predicting outcomes on new, unseen data                                                                                             |
| `Inference`        | Using a trained model to derive insights, estimate parameters, and understand relationships between variables                             |
| `Evaluation`       | Assessing the model's performance using metrics such as accuracy, precision, recall, and F1-score (harmonic mean of precision and recall) |
| `Generalization`   | Model's ability to accurately predict outcomes for new, unseen data not used in training                                                  |
| `Overfitting`      | Model learns data too well, including noise and outliers, resulting in poor generalization on new data                                    |
| `Underfitting`     | Model is too simple to capture underlying patterns of the data                                                                            |
| `Cross-Validation` | Technique used to assess how well a model will generalize to an independent dataset                                                       |
| `Regularization`   | Technique used to prevent overfitting by adding a penalty term to the loss function; discourages overly complex patterns                  |
### Linear Regression
`Linear Regression` is a fundamental `supervised learning` algorithm that predicts a continuous target variable by establishing a linear relationship between the target and one or more predictor variables

### Logistic Regression
### Decision Trees
### Naive Bayes
### Support Vector Machines

# Unsupervised Learning Algorithms
# Reinforcement Learning Algorithms
# Introduction to Deep Learning
# Introduction to Generative AI
