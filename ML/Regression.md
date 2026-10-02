
![[1-Simple+Linear+Regression (2).pdf]]

# Simple Linear Regression: A Comprehensive Mathematical Treatment

## 1. Introduction and Foundational Concepts

### 1.1 What is Regression?

Regression analysis is a fundamental statistical and machine learning technique used to model and analyze the relationship between a dependent variable (also called the response variable or target) and one or more independent variables (also called predictors, features, or explanatory variables). The goal is to understand how the dependent variable changes when the independent variables vary, and to use this understanding to make predictions.

Simple Linear Regression specifically deals with the case where we have:

- **One independent variable** (x)
- **One dependent variable** (y)
- A **linear relationship** between them

This makes it the simplest form of regression, yet it provides the foundation for understanding more complex regression techniques.

### 1.2 The Supervised Learning Context

Linear regression falls under the category of **supervised learning**, where we learn from labeled training data. In supervised learning:

1. We have a dataset of input-output pairs: {(x⁽¹⁾, y⁽¹⁾), (x⁽²⁾, y⁽²⁾), ..., (x⁽ᵐ⁾, y⁽ᵐ⁾)}
2. We use this data to learn a function h (hypothesis) that maps inputs to outputs
3. We can then use this learned function to predict outputs for new, unseen inputs

The superscript notation (i) denotes the i-th training example, where i ranges from 1 to m (the number of training samples).

### 1.3 The Linear Model

The core assumption in simple linear regression is that the relationship between x and y can be modeled by a straight line. Mathematically, we express this as:

**y = θ₀ + θ₁x**

or equivalently in the hypothesis notation:

**h_θ(x) = θ₀ + θ₁x**

where:

- **h_θ(x)** is our hypothesis function (predicted value)
- **θ₀** (theta-zero) is the **intercept** - the value of y when x = 0
- **θ₁** (theta-one) is the **slope** or **coefficient** - the rate of change of y with respect to x
- **x** is the input feature
- **θ** collectively refers to the parameters [θ₀, θ₁]

**Alternative Notations:**

- Traditional mathematics: y = mx + c (where m is slope, c is intercept)
- Statistics: y = β₀ + β₁x (where β represents parameters)
- Machine Learning: h_θ(x) = θ₀ + θ₁x (emphasizing it's a learned hypothesis)

For predictions, we often use ŷ (y-hat) to denote predicted values: **ŷ = θ₀ + θ₁x**

## 2. The Concept of the "Best Fit" Line

### 2.1 What Does "Best Fit" Mean?

Given a set of data points, there are infinitely many lines we could draw. The question becomes: which line best represents the relationship in the data?

The "best fit" line is the one that minimizes the discrepancy between the actual observed values (y⁽ⁱ⁾) and the predicted values (ŷ⁽ⁱ⁾ or h_θ(x⁽ⁱ⁾)).

### 2.2 Residuals and Errors

For each training example i, we define the **residual** (or error) as:

**e⁽ⁱ⁾ = y⁽ⁱ⁾ - ŷ⁽ⁱ⁾ = y⁽ⁱ⁾ - h_θ(x⁽ⁱ⁾) = y⁽ⁱ⁾ - (θ₀ + θ₁x⁽ⁱ⁾)**

This represents the vertical distance between the actual data point and the predicted point on our line.

**Geometric Interpretation:**

- Positive residual (e⁽ⁱ⁾ > 0): the actual point lies above the line
- Negative residual (e⁽ⁱ⁾ < 0): the actual point lies below the line
- Zero residual (e⁽ⁱ⁾ = 0): the point lies exactly on the line

The best fit line should make these residuals as small as possible overall.

## 3. The Cost Function: Mean Squared Error

### 3.1 Why Not Just Sum the Errors?

A naive approach might be to minimize the sum of all residuals:

Σ e⁽ⁱ⁾ = Σ (y⁽ⁱ⁾ - ŷ⁽ⁱ⁾)

However, this has a critical flaw: positive and negative errors would cancel each other out. A line that has large errors on both sides could appear perfect under this metric.

### 3.2 Why Square the Errors?

We use **squared errors** for several important reasons:

1. **Eliminates sign issues**: Squaring ensures all errors are positive, preventing cancellation
2. **Penalizes large errors more heavily**: A residual of 2 contributes 4 to the cost, but a residual of 4 contributes 16 - this quadratic penalty discourages large deviations
3. **Mathematical convenience**: The squared function is differentiable everywhere and has nice calculus properties
4. **Statistical justification**: Under certain assumptions (normally distributed errors), minimizing squared errors is equivalent to maximum likelihood estimation
5. **Geometric interpretation**: Related to Euclidean distance in error space

### 3.3 Deriving the Mean Squared Error Cost Function

Starting with squared errors for all m training examples:

**Sum of Squared Errors (SSE)** = Σᵢ₌₁ᵐ (y⁽ⁱ⁾ - h_θ(x⁽ⁱ⁾))²

To make this independent of sample size, we take the average:

**Mean Squared Error (MSE)** = (1/m) Σᵢ₌₁ᵐ (y⁽ⁱ⁾ - h_θ(x⁽ⁱ⁾))²

For mathematical convenience in calculus (it simplifies derivatives), we often include a factor of 1/2:

**J(θ₀, θ₁) = (1/2m) Σᵢ₌₁ᵐ (h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾)²**

Expanding the hypothesis function:

**J(θ₀, θ₁) = (1/2m) Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾)²**

This is our **cost function** (also called loss function or objective function). It measures how well our parameters θ₀ and θ₁ fit the training data.

**Key Properties:**

- J(θ₀, θ₁) ≥ 0 always (since it's a sum of squared terms)
- J(θ₀, θ₁) = 0 only if our model perfectly predicts all training examples
- The function is convex (bowl-shaped) in the parameter space, guaranteeing a unique global minimum

### 3.4 Related Error Metrics

**Root Mean Squared Error (RMSE):** RMSE = √[1/m Σᵢ₌₁ᵐ (y⁽ⁱ⁾ - ŷ⁽ⁱ⁾)²]

This has the same units as y, making it more interpretable.

**Mean Absolute Error (MAE):** MAE = (1/m) Σᵢ₌₁ᵐ |y⁽ⁱ⁾ - ŷ⁽ⁱ⁾|

Less sensitive to outliers but not differentiable at zero.

## 4. Finding Optimal Parameters: The Analytical Solution

### 4.1 The Optimization Problem

Our goal is to find the parameter values θ₀* and θ₁* that minimize the cost function:

__(θ₀_, θ₁_) = argmin J(θ₀, θ₁)** θ₀,θ₁

This is an unconstrained optimization problem. Since J is a smooth, convex function, we can use calculus to find the minimum.

### 4.2 Calculus Approach: Setting Partial Derivatives to Zero

At the minimum of a function, all partial derivatives equal zero. Therefore:

∂J/∂θ₀ = 0 ∂J/∂θ₁ = 0

Let's derive these conditions step by step.

### 4.3 Computing ∂J/∂θ₀

Starting with: J(θ₀, θ₁) = (1/2m) Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾)²

Taking the partial derivative with respect to θ₀:

∂J/∂θ₀ = ∂/∂θ₀ [(1/2m) Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾)²]

= (1/2m) Σᵢ₌₁ᵐ ∂/∂θ₀ [(θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾)²]

Using the chain rule: ∂/∂θ₀[f²] = 2f · ∂f/∂θ₀

= (1/2m) Σᵢ₌₁ᵐ 2(θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾) · ∂/∂θ₀[θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾]

Since ∂/∂θ₀[θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾] = 1:

= (1/2m) Σᵢ₌₁ᵐ 2(θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾) · 1

= (1/m) Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾)

**∂J/∂θ₀ = (1/m) Σᵢ₌₁ᵐ (h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾)**

### 4.4 Computing ∂J/∂θ₁

Similarly, for θ₁:

∂J/∂θ₁ = ∂/∂θ₁ [(1/2m) Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾)²]

= (1/2m) Σᵢ₌₁ᵐ 2(θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾) · ∂/∂θ₁[θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾]

Since ∂/∂θ₁[θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾] = x⁽ⁱ⁾:

= (1/m) Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾) · x⁽ⁱ⁾

**∂J/∂θ₁ = (1/m) Σᵢ₌₁ᵐ (h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾) · x⁽ⁱ⁾**

### 4.5 The Normal Equations

Setting both partial derivatives to zero:

**(1) (1/m) Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾) = 0**

**(2) (1/m) Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾) · x⁽ⁱ⁾ = 0**

From equation (1), multiplying by m:

Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾) = 0

Expanding: Σᵢ₌₁ᵐ θ₀ + Σᵢ₌₁ᵐ θ₁x⁽ⁱ⁾ - Σᵢ₌₁ᵐ y⁽ⁱ⁾ = 0

mθ₀ + θ₁Σᵢ₌₁ᵐ x⁽ⁱ⁾ - Σᵢ₌₁ᵐ y⁽ⁱ⁾ = 0

Let's denote:

- x̄ = (1/m)Σᵢ₌₁ᵐ x⁽ⁱ⁾ (mean of x)
- ȳ = (1/m)Σᵢ₌₁ᵐ y⁽ⁱ⁾ (mean of y)

Then: mθ₀ + θ₁(mx̄) = mȳ

**θ₀ = ȳ - θ₁x̄** ... (Equation A)

This reveals an important property: **the best fit line always passes through the point (x̄, ȳ)**, the centroid of the data.

From equation (2), multiplying by m:

Σᵢ₌₁ᵐ (θ₀ + θ₁x⁽ⁱ⁾ - y⁽ⁱ⁾) · x⁽ⁱ⁾ = 0

Expanding: θ₀Σᵢ₌₁ᵐ x⁽ⁱ⁾ + θ₁Σᵢ₌₁ᵐ (x⁽ⁱ⁾)² - Σᵢ₌₁ᵐ x⁽ⁱ⁾y⁽ⁱ⁾ = 0

Substituting θ₀ = ȳ - θ₁x̄ from Equation A:

(ȳ - θ₁x̄)Σᵢ₌₁ᵐ x⁽ⁱ⁾ + θ₁Σᵢ₌₁ᵐ (x⁽ⁱ⁾)² - Σᵢ₌₁ᵐ x⁽ⁱ⁾y⁽ⁱ⁾ = 0

ȳ(mx̄) - θ₁x̄(mx̄) + θ₁Σᵢ₌₁ᵐ (x⁽ⁱ⁾)² - Σᵢ₌₁ᵐ x⁽ⁱ⁾y⁽ⁱ⁾ = 0

mȳx̄ - mθ₁x̄² + θ₁Σᵢ₌₁ᵐ (x⁽ⁱ⁾)² = Σᵢ₌₁ᵐ x⁽ⁱ⁾y⁽ⁱ⁾

θ₁[Σᵢ₌₁ᵐ (x⁽ⁱ⁾)² - mx̄²] = Σᵢ₌₁ᵐ x⁽ⁱ⁾y⁽ⁱ⁾ - mȳx̄

Now, note that: Σᵢ₌₁ᵐ (x⁽ⁱ⁾)² - mx̄² = Σᵢ₌₁ᵐ (x⁽ⁱ⁾)² - m(Σᵢx⁽ⁱ⁾/m)² = Σᵢ₌₁ᵐ (x⁽ⁱ⁾)² - (Σᵢx⁽ⁱ⁾)²/m = [mΣᵢ(x⁽ⁱ⁾)² - (Σᵢx⁽ⁱ⁾)²]/m = Σᵢ₌₁ᵐ (x⁽ⁱ⁾ - x̄)²

And: Σᵢ₌₁ᵐ x⁽ⁱ⁾y⁽ⁱ⁾ - mȳx̄ = Σᵢ₌₁ᵐ x⁽ⁱ⁾y⁽ⁱ⁾ - (Σᵢy⁽ⁱ⁾)(Σᵢx⁽ⁱ⁾)/m = Σᵢ₌₁ᵐ (x⁽ⁱ⁾ - x̄)(y⁽ⁱ⁾ - ȳ)

Therefore:

**θ₁ = Σᵢ₌₁ᵐ (x⁽ⁱ⁾ - x̄)(y⁽ⁱ⁾ - ȳ) / Σᵢ₌₁ᵐ (x⁽ⁱ⁾ - x̄)²**

And from Equation A:

**θ₀ = ȳ - θ₁x̄**

These are the **closed-form solutions** (also called the Normal Equations) for simple linear regression.

### 4.6 Geometric and Statistical Interpretation

The formula for θ₁ can also be written as:

**θ₁ = Cov(x,y) / Var(x) = σₓᵧ / σₓ²**

where:

- Cov(x,y) = (1/m)Σᵢ(x⁽ⁱ⁾ - x̄)(y⁽ⁱ⁾ - ȳ) is the covariance between x and y
- Var(x) = (1/m)Σᵢ(x⁽ⁱ⁾ - x̄)² is the variance of x

This connects linear regression to correlation: θ₁ = r · (σᵧ/σₓ), where r is the correlation coefficient.

**Interpretation:**

- θ₁ measures how much y changes per unit change in x
- θ₀ ensures the line passes through (x̄, ȳ)
- The slope depends on how x and y co-vary relative to the spread in x

## 5. Gradient Descent: The Iterative Optimization Approach

### 5.1 Motivation for Gradient Descent

While the analytical solution (Normal Equations) works perfectly for simple linear regression, gradient descent becomes essential when:

1. The number of features is very large (computing matrix inverse becomes expensive)
2. We're dealing with non-linear models where closed-form solutions don't exist
3. We're working with massive datasets where iterative methods are more memory-efficient

Gradient descent is a general optimization algorithm that forms the backbone of training neural networks and many other machine learning models.

### 5.2 The Intuition Behind Gradient Descent

Imagine you're standing on a hill in thick fog and want to reach the valley (minimum). You can't see far, but you can feel the slope beneath your feet. A natural strategy is:

1. Determine which direction slopes downward most steeply
2. Take a step in that direction
3. Repeat until you reach the bottom

This is exactly what gradient descent does in the parameter space.

**Key Concept: The Gradient**

The gradient ∇J is a vector of partial derivatives:

∇J = [∂J/∂θ₀] [∂J/∂θ₁]

The gradient points in the direction of steepest **ascent**. To minimize J, we move in the opposite direction: **-∇J**.

### 5.3 The Gradient Descent Algorithm

**Algorithm:**

```
1. Initialize parameters: θ₀, θ₁ (often to 0 or small random values)
2. Repeat until convergence {
     θ₀ := θ₀ - α · (∂J/∂θ₀)
     θ₁ := θ₁ - α · (∂J/∂θ₁)
   }
```

where:

- **α** (alpha) is the **learning rate** - controls the step size
- **:=** denotes assignment (update)
- Both parameters must be updated **simultaneously**

### 5.4 Computing the Gradients

We already derived these in Section 4.3 and 4.4:

**∂J/∂θ₀ = (1/m) Σᵢ₌₁ᵐ (h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾)**

**∂J/∂θ₁ = (1/m) Σᵢ₌₁ᵐ (h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾) · x⁽ⁱ⁾**

Substituting into the update rules:

**θ₀ := θ₀ - α · (1/m) Σᵢ₌₁ᵐ (h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾)**

**θ₁ := θ₁ - α · (1/m) Σᵢ₌₁ᵐ (h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾) · x⁽ⁱ⁾**

### 5.5 Detailed Gradient Descent Steps

Let's see how this works with a concrete example from the images:

**Initial Setup:**

```
Training data: (1,1), (2,2), (3,3)
Initialize: θ₀ = 0, θ₁ = 0
Learning rate: α = 0.01
```

**Iteration 0:** h_θ(x) = 0 + 0·x = 0 for all x

Calculate gradients:

- ∂J/∂θ₀ = (1/3)[(0-1) + (0-2) + (0-3)] = -6/3 = -2
- ∂J/∂θ₁ = (1/3)[(0-1)·1 + (0-2)·2 + (0-3)·3] = (1/3)[-1-4-9] = -14/3

Update:

- θ₀ := 0 - 0.01·(-2) = 0.02
- θ₁ := 0 - 0.01·(-14/3) ≈ 0.047

**Iteration 1:** h_θ(x) = 0.02 + 0.047x

Calculate new gradients and repeat...

The cost J(θ₀, θ₁) decreases with each iteration, following a curve down toward the minimum.

### 5.6 The Learning Rate α

The learning rate is a crucial hyperparameter:

**If α is too small:**

- Gradient descent will converge very slowly
- Requires many iterations
- But guaranteed to make progress (for convex functions)

**If α is too large:**

- May overshoot the minimum
- Could diverge or oscillate
- Never converges

**If α is just right:**

- Converges efficiently
- Reaches minimum in reasonable time

**Adaptive learning rates**: In practice, techniques like learning rate decay or adaptive methods (Adam, RMSprop) adjust α during training.

### 5.7 Convergence Criteria

How do we know when to stop? Common criteria:

1. **Cost threshold**: Stop when J(θ) < ε for some small ε
2. **Change threshold**: Stop when |J^(t) - J^(t-1)| < ε
3. **Gradient threshold**: Stop when ||∇J|| < ε
4. **Maximum iterations**: Stop after a fixed number of iterations

### 5.8 Variants of Gradient Descent

**Batch Gradient Descent (what we described):**

- Uses all m training examples in each iteration
- Update rule uses full sum: Σᵢ₌₁ᵐ
- Guaranteed convergence to global minimum for convex functions
- Can be slow for large datasets

**Stochastic Gradient Descent (SGD):**

- Uses one random training example per iteration
- Much faster iterations
- Noisier updates, but often faster overall convergence
- May not reach exact minimum but oscillates near it

**Mini-batch Gradient Descent:**

- Uses a small batch (e.g., 32, 64, 256) of training examples
- Balance between batch and stochastic
- Most commonly used in practice
- Allows vectorization for computational efficiency

### 5.9 Visualizing the Cost Function

For simple linear regression, J(θ₀, θ₁) forms a convex paraboloid (bowl shape) in 3D. The contour plot shows concentric ellipses, with the minimum at the center. Gradient descent traces a path from the initial point toward this center, with each step perpendicular to the contour lines.

## 6. Model Evaluation and Performance Metrics

### 6.1 The Coefficient of Determination (R²)

R² (R-squared) is the most common metric for evaluating regression models. It represents the proportion of variance in the dependent variable that is predictable from the independent variable.

**Definition:**

**R² = 1 - (RSS/TSS) = 1 - Σᵢ(y⁽ⁱ⁾ - ŷ⁽ⁱ⁾)² / Σᵢ(y⁽ⁱ⁾ - ȳ)²**

where:

- **RSS** (Residual Sum of Squares) = Σᵢ(y⁽ⁱ⁾ - ŷ⁽ⁱ⁾)² - unexplained variance
- **TSS** (Total Sum of Squares) = Σᵢ(y⁽ⁱ⁾ - ȳ)² - total variance

**Derivation and Interpretation:**

The total sum of squares can be decomposed: TSS = ESS + RSS

where:

- **ESS** (Explained Sum of Squares) = Σᵢ(ŷ⁽ⁱ⁾ - ȳ)² - variance explained by the model

Therefore: R² = ESS/TSS = (TSS - RSS)/TSS = 1 - RSS/TSS

**Properties:**

- R² ∈ [0, 1] for models with intercept
- R² = 1: perfect fit (all points on the line)
- R² = 0: model no better than predicting the mean ȳ
- R² close to 1: most variance explained
- R² can be negative for models without intercept or when predictions are very poor

**Connection to Correlation:** For simple linear regression: R² = r², where r is the Pearson correlation coefficient between x and y.

### 6.2 Residual Analysis

Examining residuals (actual - predicted) is crucial for validating model assumptions.

**Residual Plots:**

1. **Residuals vs. Fitted Values**: Should show random scatter around zero
    - Patterns indicate non-linearity or heteroscedasticity
2. **Residuals vs. Independent Variable**: Should show random scatter
    - Patterns suggest relationship isn't linear
3. **Histogram of Residuals**: Should approximate a normal distribution
    - Skewness or heavy tails indicate assumption violations
4. **Q-Q Plot**: Quantile-quantile plot against normal distribution
    - Points should lie on diagonal line
    - Deviations indicate non-normality

### 6.3 Standard Error of the Estimate

The **standard error of regression (SER)** or **residual standard error** estimates the standard deviation of the residuals:

**σ̂ = √[RSS/(m-2)] = √[Σᵢ(y⁽ⁱ⁾ - ŷ⁽ⁱ⁾)²/(m-2)]**

The denominator is m-2 (degrees of freedom) because we estimated two parameters (θ₀ and θ₁).

This gives us the typical size of prediction errors in the same units as y.

### 6.4 Confidence Intervals for Parameters

The estimated parameters θ̂₀ and θ̂₁ are random variables (they would vary with different samples). We can construct confidence intervals:

**Standard Error of θ̂₁:** SE(θ̂₁) = σ̂ / √[Σᵢ(x⁽ⁱ⁾ - x̄)²]

**Standard Error of θ̂₀:** SE(θ̂₀) = σ̂ · √[1/m + x̄²/Σᵢ(x⁽ⁱ⁾ - x̄)²]

**95% Confidence Intervals:**

- θ₁: θ̂₁ ± t₀.₀₂₅,ₘ₋₂ · SE(θ̂₁)
- θ₀: θ̂₀ ± t₀.₀₂₅,ₘ₋₂ · SE(θ̂₀)

where t₀.₀₂₅,ₘ₋₂ is the critical value from the t-distribution with m-2 degrees of freedom.

### 6.5 Prediction Intervals vs. Confidence Intervals

**Confidence Interval for E[y|x]**: Estimates the mean response at a given x

- Width: depends on how far x is from x̄
- Narrower than prediction intervals

**Prediction Interval for y|x**: Estimates an individual observation at a given x

- Must account for both parameter uncertainty and inherent variability
- Wider than confidence intervals
- Formula: ŷ ± t · σ̂√[1 + 1/m + (x-x̄)²/Σᵢ(x⁽ⁱ⁾-x̄)²]

### 6.6 Hypothesis Testing

**Testing if slope is zero (no relationship):**

Null hypothesis: H₀: θ₁ = 0 Alternative: H₁: θ₁ ≠ 0

Test statistic: t = θ̂₁ / SE(θ̂₁)

Under H₀, this follows a t-distribution with m-2 degrees of freedom. If |t| > critical value, we reject H₀ and conclude there is a significant linear relationship.

## 7. Assumptions of Linear Regression

Linear regression relies on several key assumptions. Violations can lead to biased estimates, inefficient predictions, or invalid inference.

### 7.1 Linearity

**Assumption**: The relationship between x and y is linear.

Mathematically: E[y|x] = θ₀ + θ₁x

**Check**: Scatter plot of y vs. x, residual plots **Violation**: If relationship is curved, predictions will be systematically biased **Solution**: Transform variables (log, square root, etc.) or use non-linear models

### 7.2 Independence of Errors

**Assumption**: The residuals e⁽ⁱ⁾ are independent of each other.

Mathematically: Cov(e⁽ⁱ⁾, e⁽ʲ⁾) = 0 for i ≠ j

**Check**: Plot residuals in sequence (for time series), Durbin-Watson test **Violation**: Autocorrelation leads to underestimated standard errors **Common cause**: Time series data, spatial data **Solution**: Use time series methods, add lagged variables, or model the correlation structure

### 7.3 Homoscedasticity (Constant Variance)

**Assumption**: The variance of residuals is constant across all values of x.

Mathematically: Var(e|x) = σ² (constant)

**Check**: Plot residuals vs. fitted values (should show equal spread), Breusch-Pagan test **Violation (Heteroscedasticity)**:

- Confidence intervals and p-values become unreliable
- Efficiency loss (not optimal use of data) **Common pattern**: Variance increases with x (cone shape in residual plot) **Solution**: Transform y (log, square root), use weighted least squares, or use robust standard errors

### 7.4 Normality of Residuals

**Assumption**: Residuals are normally distributed.

Mathematically: e⁽ⁱ⁾ ~ N(0, σ²)

**Check**: Histogram of residuals, Q-Q plot, Shapiro-Wilk test **Violation**: Primarily affects inference (confidence intervals, hypothesis tests)

- Predictions remain unbiased
- Less critical for large samples (Central Limit Theorem) **Solution**: Transform variables, use robust methods, or rely on large-sample asymptotics

### 7.5 No Multicollinearity (For Multiple Regression)

While not directly applicable to simple linear regression, when we extend to multiple features, we require:

**Assumption**: Independent variables are not highly correlated with each other.

**Check**: Correlation matrix, Variance Inflation Factor (VIF) **Violation**: Parameter estimates become unstable, large standard errors **Solution**: Remove redundant variables, principal component analysis, or regularization

### 7.6 Zero Conditional Mean

**Assumption**: E[e|x] = 0

This means no systematic errors - the residuals average to zero at every value of x.

This assumption is automatically satisfied if we include an intercept and the relationship is truly linear.

## 8. Extension to Multiple Linear Regression

### 8.1 Motivation and Notation

Real-world problems often involve multiple predictors. For example, predicting house prices might depend on:

- Size (x₁)
- Number of bedrooms (x₂)
- Age (x₃)
- Location (x₄)

**Multiple Linear Regression Model:**

**h_θ(x) = θ₀ + θ₁x₁ + θ₂x₂ + ... + θₙxₙ**

where n is the number of features.

**Vector Notation:**

Let x = [1, x₁, x₂, ..., xₙ]ᵀ (we include 1 for the intercept term) Let θ = [θ₀, θ₁, θ₂, ..., θₙ]ᵀ

Then: **h_θ(x) = θᵀx** = θ₀·1 + θ₁x₁ + θ₂x₂ + ... + θₙxₙ

### 8.2 Matrix Formulation

For m training examples with n features:

**Design Matrix X** (m × (n+1)):

```
X = [1  x₁⁽¹⁾  x₂⁽¹⁾  ...  xₙ⁽¹⁾]
    [1  x₁⁽²⁾  x₂⁽²⁾  ...  xₙ⁽²⁾]
    [⋮    ⋮      ⋮    ⋱    ⋮   ]
    [1  x₁⁽ᵐ⁾  x₂⁽ᵐ⁾  ...  xₙ⁽ᵐ⁾]
```

**Parameter Vector θ** ((n+1) × 1):

```
θ = [θ₀]
    [θ₁]
    [θ₂]
    [⋮ ]
    [θₙ]
```

**Response Vector y** (m × 1):

```
y = [y⁽¹⁾]
    [y⁽²⁾]
    [ ⋮  ]
    [y⁽ᵐ⁾]
```

**Predictions**: ŷ = Xθ

### 8.3 Cost Function in Matrix Form

**J(θ) = (1/2m)(Xθ - y)ᵀ(Xθ - y)**

Expanding: J(θ) = (1/2m)[θᵀXᵀXθ - 2θᵀXᵀy + yᵀy]

### 8.4 Deriving the Normal Equation

To minimize J(θ), we take the gradient with respect to θ and set it to zero:

∇_θJ = (1/m)[XᵀXθ - Xᵀy] = 0

Multiplying by m: XᵀXθ - Xᵀy = 0

**XᵀXθ = Xᵀy**

Assuming XᵀX is invertible:

**θ = (XᵀX)⁻¹Xᵀy**

This is the **Normal Equation** for multiple linear regression - a closed-form solution.

**Derivation Details:**

The gradient of J(θ) with respect to θ:

∂J/∂θ = ∂/∂θ[(1/2m)(θᵀXᵀXθ - 2θᵀXᵀy + yᵀy)]

Using matrix calculus rules:

- ∂/∂θ(θᵀAθ) = 2Aθ (for symmetric A)
- ∂/∂θ(θᵀb) = b

We get: ∂J/∂θ = (1/2m)[2XᵀXθ - 2Xᵀy] = (1/m)[XᵀXθ - Xᵀy]

Setting to zero and solving gives the Normal Equation.

### 8.5 Gradient Descent for Multiple Features

**Update Rule (Vectorized):**

**θ := θ - (α/m)Xᵀ(Xθ - y)**

Component-wise: **θⱼ := θⱼ - (α/m) Σᵢ₌₁ᵐ (h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾)xⱼ⁽ⁱ⁾** for j = 0, 1, ..., n

(where x₀⁽ⁱ⁾ = 1 for all i)

### 8.6 Normal Equation vs. Gradient Descent

**Normal Equation Advantages:**

- No need to choose α
- No iteration required
- Gives exact answer in one computation

**Normal Equation Disadvantages:**

- Need to compute (XᵀX)⁻¹ which is O(n³)
- Doesn't work if XᵀX is not invertible (singular)
- Slow for large n (n > 10,000)

**Gradient Descent Advantages:**

- Works well even with large n
- Works for other models beyond linear regression
- O(kn²) where k is number of iterations

**Gradient Descent Disadvantages:**

- Need to choose α
- Requires many iterations
- Need to preprocess features (scaling)

## 9. Practical Considerations

### 9.1 Feature Scaling and Normalization

When features have different scales, gradient descent can be slow or oscillate.

**Problem**: If x₁ ∈ [0, 1000] and x₂ ∈ [0, 1], the cost function contours become elongated ellipses.

**Solutions:**

**Min-Max Scaling** (Normalization): x' = (x - min(x)) / (max(x) - min(x)) Result: x' ∈ [0, 1]

**Z-score Standardization**: x' = (x - μₓ) / σₓ Result: x' has mean 0 and standard deviation 1

**Why it helps**: Makes the cost function more spherical, allowing gradient descent to converge faster with larger learning rates.

### 9.2 Handling Outliers

Outliers can dramatically affect the best fit line because squared errors heavily penalize large deviations.

**Detection:**

- Visual inspection (scatter plots)
- Residual analysis (points with large |e⁽ⁱ⁾|)
- Statistical tests (Cook's distance, leverage)

**Strategies:**

1. **Remove if erroneous**: Data entry errors, measurement errors
2. **Keep if legitimate**: Real data points, even if extreme
3. **Transform**: Log transformation can reduce impact
4. **Robust regression**: Use methods less sensitive to outliers (e.g., RANSAC, Huber loss)

### 9.3 Overfitting vs. Underfitting

**Underfitting** (High Bias):

- Model too simple for the data
- High training error
- High test error
- Example: Using linear regression for highly non-linear data

**Overfitting** (High Variance):

- Model too complex for the data
- Low training error
- High test error
- Captures noise instead of signal
- Less common in simple linear regression (more relevant with polynomial features)

**Solutions to Overfitting:**

- Get more training data
- Reduce feature complexity
- Regularization (Ridge, Lasso)
- Cross-validation for model selection

**The Bias-Variance Tradeoff:** Total Error = Bias² + Variance + Irreducible Error

We seek the model complexity that minimizes this total error.

### 9.4 When Linear Regression is Appropriate

**Good fit when:**

- Relationship is approximately linear
- Continuous numeric response variable
- Assumptions are reasonably met
- Interpretability is important
- Need fast predictions

**Poor fit when:**

- Relationship is highly non-linear
- Response is categorical (use logistic regression)
- Assumptions seriously violated
- Complex interactions between features
- Need highest possible prediction accuracy (ensemble methods might be better)

### 9.5 Computational Complexity

**Normal Equation:**

- Time: O(n³) for matrix inversion + O(mn²) for matrix multiplications
- Space: O(n²) for storing XᵀX
- Total: O(n²(m + n))

**Gradient Descent:**

- Time per iteration: O(mn)
- Space: O(n) for storing θ
- Total: O(kmn) where k is number of iterations

For small n and m: Normal Equation For large n or m: Gradient Descent For very large m: Stochastic/Mini-batch Gradient Descent

## 10. Connection to Maximum Likelihood Estimation

### 10.1 Probabilistic Interpretation

Linear regression can be derived from a probabilistic perspective. Assume:

**y⁽ⁱ⁾ = θ₀ + θ₁x⁽ⁱ⁾ + ε⁽ⁱ⁾**

where ε⁽ⁱ⁾ ~ N(0, σ²) are independent, normally distributed errors.

This means: **y⁽ⁱ⁾ | x⁽ⁱ⁾ ~ N(θ₀ + θ₁x⁽ⁱ⁾, σ²)**

The probability density: p(y⁽ⁱ⁾ | x⁽ⁱ⁾; θ) = (1/√(2πσ²)) exp[-(y⁽ⁱ⁾ - θ₀ - θ₁x⁽ⁱ⁾)² / (2σ²)]

### 10.2 Maximum Likelihood Estimation

The **likelihood** of observing all the data:

L(θ) = ∏ᵢ₌₁ᵐ p(y⁽ⁱ⁾ | x⁽ⁱ⁾; θ)

The **log-likelihood**:

ℓ(θ) = log L(θ) = Σᵢ₌₁ᵐ log p(y⁽ⁱ⁾ | x⁽ⁱ⁾; θ)

= Σᵢ₌₁ᵐ [log(1/√(2πσ²)) - (y⁽ⁱ⁾ - θ₀ - θ₁x⁽ⁱ⁾)²/(2σ²)]

= -m log(√(2πσ²)) - (1/2σ²)Σᵢ₌₁ᵐ (y⁽ⁱ⁾ - θ₀ - θ₁x⁽ⁱ⁾)²

**Maximizing ℓ(θ)** with respect to θ is equivalent to **minimizing Σᵢ(y⁽ⁱ⁾ - θ₀ - θ₁x⁽ⁱ⁾)²**, which is exactly our least squares objective!

**Conclusion**: Under the assumption of normally distributed errors, minimizing MSE is equivalent to maximum likelihood estimation.

### 10.3 Relationship to Bayesian Inference

With appropriate priors on θ, linear regression can be formulated in a Bayesian framework, leading to Ridge regression (L2 regularization) as a special case.

## 11. Advanced Topics and Extensions

### 11.1 Regularization

To prevent overfitting and handle multicollinearity:

**Ridge Regression (L2)**: J(θ) = (1/2m)Σᵢ(h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾)² + (λ/2m)Σⱼ₌₁ⁿ θⱼ²

**Lasso Regression (L1)**: J(θ) = (1/2m)Σᵢ(h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾)² + (λ/2m)Σⱼ₌₁ⁿ |θⱼ|

**Elastic Net** (combination of L1 and L2)

### 11.2 Polynomial Regression

Extend to non-linear relationships while maintaining linearity in parameters:

h_θ(x) = θ₀ + θ₁x + θ₂x² + θ₃x³ + ...

This is still "linear regression" because it's linear in the parameters θ.

### 11.3 Robust Regression

Alternative loss functions less sensitive to outliers:

- Huber loss
- Quantile regression
- RANSAC (Random Sample Consensus)

### 11.4 Weighted Least Squares

When error variance is not constant, assign weights inversely proportional to variance:

J(θ) = (1/2m)Σᵢ wᵢ(h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾)²

### 11.5 Generalized Linear Models

Extend beyond continuous responses:

- Logistic regression (binary classification)
- Poisson regression (count data)
- Exponential family distributions

## 12. Summary and Key Takeaways

### 12.1 Core Concepts

1. **Simple linear regression** models the relationship between one predictor and one response with a straight line
2. The **best fit line** minimizes the sum of squared residuals
3. Parameters can be found analytically (Normal Equations) or iteratively (Gradient Descent)
4. Model evaluation uses R², residual analysis, and hypothesis tests
5. Key assumptions: linearity, independence, homoscedasticity, normality

### 12.2 Mathematical Framework

- **Model**: y = θ₀ + θ₁x + ε
- **Cost Function**: J(θ) = (1/2m)Σᵢ(h_θ(x⁽ⁱ⁾) - y⁽ⁱ⁾)²
- **Optimal Parameters**: θ₁ = Cov(x,y)/Var(x), θ₀ = ȳ - θ₁x̄
- **Matrix Form**: θ = (XᵀX)⁻¹Xᵀy

### 12.3 Practical Insights

- Feature scaling improves gradient descent convergence
- Check assumptions through residual analysis
- Balance model complexity to avoid underfitting/overfitting
- Choose between analytical and iterative solutions based on problem size
- Linear regression provides interpretable, fast, and often effective baseline models

### 12.4 Connections to Broader ML

- Gradient descent is fundamental to training neural networks
- The bias-variance tradeoff applies to all models
- Regularization techniques extend to deep learning
- Linear regression principles generalize to logistic regression, GLMs, and beyond

Linear regression, despite its simplicity, embodies fundamental principles that permeate all of machine learning: learning from data, optimizing objective functions, balancing complexity with generalization, and making probabilistic predictions. Mastering these concepts provides a solid foundation for understanding more sophisticated models.