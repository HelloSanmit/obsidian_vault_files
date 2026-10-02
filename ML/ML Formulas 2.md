
# Supervised Learning — Fully Explicit, Derivation-Safe Notes

---

## Global, Fully Typed Notation (Canonical)

- $n$ : number of training samples
    
- $m$ : input dimensionality
    
- $k$ : number of classes (classification)
    
- $\mathbf{x}^{(i)} \in \mathbb{R}^m$ : $i$-th input
    
- $y^{(i)} \in \mathbb{R}$ (regression) or ${1,\dots,k}$ (classification)
    
- $\mathbf{X} \in \mathbb{R}^{n \times m}$ : design matrix (rows = samples)
    
- $\mathbf{y} \in \mathbb{R}^n$
    
- $\mathbf{w} \in \mathbb{R}^m$, $b \in \mathbb{R}$
    

Augmented form (optional):

$$  
\tilde{\mathbf{x}} =  
\begin{bmatrix}  
\mathbf{x} \  
1  
\end{bmatrix},  
\quad  
\tilde{\mathbf{w}} =  
\begin{bmatrix}  
\mathbf{w} \  
b  
\end{bmatrix}  
$$

- $\alpha$ : learning rate
    
- $\lambda$ : regularization strength
    
- All gradients are column vectors
    
- All expectations are w.r.t. the data distribution unless stated
    

---

# Linear Regression

## Model

$$  
h_\theta(\mathbf{x}) = \mathbf{w}^T \mathbf{x} + b  
$$

Linear **in parameters**, not necessarily in features.

---

## Loss (Mean Squared Error)

$$  
J(\mathbf{w}, b)  
= \frac{1}{2n}  
\sum_{i=1}^n  
\left(  
\mathbf{w}^T \mathbf{x}^{(i)} + b - y^{(i)}  
\right)^2  
$$

The $\tfrac{1}{2}$ exists purely for gradient convenience.

---

## Matrix Form

$$  
J(\mathbf{w})  
= \frac{1}{2n}  
|\mathbf{X}\mathbf{w} - \mathbf{y}|_2^2  
$$

---

## Normal Equation (Correct Domain)

$$  
\hat{\mathbf{w}}  
= (\mathbf{X}^T \mathbf{X})^{-1}  
\mathbf{X}^T \mathbf{y}  
$$

Valid **only if**:

- $\mathbf{X}^T \mathbf{X}$ is invertible
    
- typically requires $n \ge m$ and full column rank
    

More generally:

$$  
\hat{\mathbf{w}} = \mathbf{X}^+ \mathbf{y}  
$$

where $\mathbf{X}^+$ is the Moore–Penrose pseudoinverse.

---

# Gradient Descent (Linear Regression)

## Gradients

$$  
\nabla_{\mathbf{w}} J  
= \frac{1}{n}  
\mathbf{X}^T (\mathbf{X}\mathbf{w} - \mathbf{y})  
$$

$$  
\frac{\partial J}{\partial b}  
= \frac{1}{n}  
\sum_{i=1}^n  
(\hat{y}^{(i)} - y^{(i)})  
$$

---

## Updates

$$  
\mathbf{w}_{t+1}  
= \mathbf{w}_t

- \alpha \nabla_{\mathbf{w}} J  
    $$
    

Stochastic gradient descent replaces the full sum with a single sample.

---

# Regularization

## Ridge Regression (L2)

$$  
J  
= \frac{1}{2n}  
|\mathbf{X}\mathbf{w} - \mathbf{y}|_2^2

- \frac{\lambda}{2n}  
    |\mathbf{w}|_2^2  
    $$
    

Closed form:

# $$  
\hat{\mathbf{w}}

(\mathbf{X}^T \mathbf{X}

- \lambda \mathbf{I})^{-1}  
    \mathbf{X}^T \mathbf{y}  
    $$
    

---

## Lasso Regression (L1)

$$  
J  
= \frac{1}{2n}  
|\mathbf{X}\mathbf{w} - \mathbf{y}|_2^2

- \frac{\lambda}{n}  
    |\mathbf{w}|_1  
    $$
    

- No closed-form solution
    
- Solved via coordinate descent or proximal methods
    

---

# Logistic Regression

## Model

$$  
P(y=1 \mid \mathbf{x})  
= \sigma(\mathbf{w}^T \mathbf{x} + b)  
$$

$$  
\sigma(z)  
= \frac{1}{1 + e^{-z}}  
$$

---

## Loss (Negative Log-Likelihood)

$$  
J  
= -\frac{1}{n}  
\sum_{i=1}^n  
\Big[  
y^{(i)} \log p^{(i)}

- (1 - y^{(i)}) \log (1 - p^{(i)})  
    \Big]  
    $$
    

where

$$  
p^{(i)} = \sigma(\mathbf{w}^T \mathbf{x}^{(i)} + b)  
$$

---

## Gradient

$$  
\nabla_{\mathbf{w}} J  
= \frac{1}{n}  
\mathbf{X}^T (\mathbf{p} - \mathbf{y})  
$$

---

# Softmax Regression (Multinomial Logistic)

## Model

# $$  
P(y=c \mid \mathbf{x})

\frac{  
e^{\mathbf{w}_c^T \mathbf{x}}  
}{  
\sum_{j=1}^k  
e^{\mathbf{w}_j^T \mathbf{x}}  
}  
$$

- $\mathbf{W} \in \mathbb{R}^{k \times m}$
    

---

## Loss (Cross-Entropy)

$$  
J  
= -\frac{1}{n}  
\sum_{i=1}^n  
\sum_{c=1}^k  
y_c^{(i)} \log p_c^{(i)}  
$$

Labels $y_c^{(i)}$ are one-hot encoded.

---

# Support Vector Machines

## Primal (Soft Margin)

$$  
\min_{\mathbf{w}, b, \boldsymbol{\xi}}  
;  
\frac{1}{2}  
|\mathbf{w}|_2^2

- C \sum_{i=1}^n \xi_i  
    $$
    

Subject to:

$$  
y^{(i)}(\mathbf{w}^T \mathbf{x}^{(i)} + b)  
\ge 1 - \xi_i,  
\quad  
\xi_i \ge 0  
$$

---

## Hinge Loss Form

# $$  
J(\mathbf{w})

\frac{1}{2}  
|\mathbf{w}|_2^2

- C  
    \sum_{i=1}^n  
    \max!\big(0,,  
    1 - y^{(i)} f(\mathbf{x}^{(i)})  
    \big)  
    $$
    

---

# Neural Networks

## Layer Dimensions

- $\mathbf{W}^{[l]} \in \mathbb{R}^{d_l \times d_{l-1}}$
    
- $\mathbf{b}^{[l]} \in \mathbb{R}^{d_l}$
    

---

## Forward Pass

# $$  
\mathbf{z}^{[l]}

\mathbf{W}^{[l]}  
\mathbf{a}^{[l-1]}

- \mathbf{b}^{[l]}  
    $$
    

# $$  
\mathbf{a}^{[l]}

g^{[l]}(\mathbf{z}^{[l]})  
$$

---

## Backpropagation

# $$  
\delta^{[L]}

\mathbf{a}^{[L]} - \mathbf{y}  
$$

# $$  
\delta^{[l]}

(\mathbf{W}^{[l+1]T}  
\delta^{[l+1]})  
\odot g'(\mathbf{z}^{[l]})  
$$

---

## Gradients

# $$  
\frac{\partial J}  
{\partial \mathbf{W}^{[l]}}

\frac{1}{n}  
\delta^{[l]}  
\mathbf{a}^{[l-1]T}  
$$

---

# Optimization Algorithms

## Adam (Fully Correct)

# $$  
\mathbf{m}_t

\beta_1 \mathbf{m}_{t-1}

- (1 - \beta_1)\mathbf{g}_t  
    $$
    

# $$  
\mathbf{v}_t

\beta_2 \mathbf{v}_{t-1}

- (1 - \beta_2)\mathbf{g}_t^2  
    $$
    

Bias correction:

# $$
\hat{\mathbf{m}}_t
=
\frac{\mathbf{m}_t}
{1 - \beta_1^{\,t}},

\hat{\mathbf{v}}_t
=
\frac{\mathbf{v}_t}
{1 - \beta_2^{\,t}}
$$


Update:

# $$
\theta_{t+1}
=
\theta_t
-
\alpha
\frac{\hat{\mathbf{m}}_t}
{\sqrt{\hat{\mathbf{v}}_t} + \epsilon}
$$


---

# Decision Trees

## Entropy

# $$  
H(S)

- \sum_{c=1}^k  
    p_c \log_2 p_c  
    $$
    

---

## Information Gain

# $$  
IG(S, A)
=
H(S)
-
\sum_{v}
\frac{|S_v|}{|S|}
\, H(S_v)
$$

---

# Bias–Variance Decomposition

(Regression, squared loss)

# $$  
\mathbb{E}  
\big[  
(y - \hat{f}(x))^2  
\big]

\underbrace{  
(\mathbb{E}[\hat{f}(x)] - f(x))^2  
}_{\text{bias}^2}  
+  
\underbrace{  
\mathbb{E}  
[  
(\hat{f}(x)

- \mathbb{E}\hat{f}(x))^2  
    ]  
    }_{\text{variance}}
    

\underbrace{  
\sigma^2  
}_{\text{irreducible noise}}  
$$

This decomposition does **not** hold unchanged for classification.

---

# Cross-Validation

## K-Fold

# $$  
CV_k

\frac{1}{k}  
\sum_{j=1}^k  
\frac{1}{|V_j|}  
\sum_{i \in V_j}  
L(y^{(i)}, \hat{y}^{(i)})  
$$

---

## Leave-One-Out (LOOCV)

# $$  
CV_{\text{LOO}}
=
\frac{1}{n}
\sum_{i=1}^{n}
\left(
y^{(i)}
-
\hat{y}^{(i)}_{-i}
\right)^2
$$
 

