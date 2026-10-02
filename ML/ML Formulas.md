
# Mathematical Formulas in Supervised Learning

## Notation

- $n$ = number of training examples
- $m$ = number of features
- $\mathbf{x}^{(i)} \in \mathbb{R}^m$ = input features for example $i$
- $y^{(i)}$ = output/target for example $i$
- $\hat{y}^{(i)}$ = predicted output for example $i$
- $\mathbf{w} \in \mathbb{R}^m$ = weight vector
- $b$ = bias term
- $\theta$ = parameter vector (often $[\mathbf{w}; b]$)
- $\alpha$ = learning rate
- $\lambda$ = regularization parameter
- $k$ = number of classes (classification)

---

## Linear Regression

**Hypothesis:** $$h_\theta(\mathbf{x}) = \mathbf{w}^T\mathbf{x} + b = \sum_{j=1}^{m} w_j x_j + b$$

**Mean Squared Error (MSE) Loss:** $$J(\theta) = \frac{1}{2n} \sum_{i=1}^{n} (h_\theta(\mathbf{x}^{(i)}) - y^{(i)})^2$$

**Root Mean Squared Error (RMSE):** $$RMSE = \sqrt{\frac{1}{n} \sum_{i=1}^{n} (h_\theta(\mathbf{x}^{(i)}) - y^{(i)})^2}$$

**Mean Absolute Error (MAE):** $$MAE = \frac{1}{n} \sum_{i=1}^{n} |h_\theta(\mathbf{x}^{(i)}) - y^{(i)}|$$

**R-squared (Coefficient of Determination):** $$R^2 = 1 - \frac{\sum_{i=1}^{n}(y^{(i)} - \hat{y}^{(i)})^2}{\sum_{i=1}^{n}(y^{(i)} - \bar{y})^2}$$

where $\bar{y} = \frac{1}{n}\sum_{i=1}^{n} y^{(i)}$

**Normal Equation (Closed-form solution):** $$\theta = (\mathbf{X}^T\mathbf{X})^{-1}\mathbf{X}^T\mathbf{y}$$

---

## Gradient Descent

**Parameter Update Rule:** $$\theta_j := \theta_j - \alpha \frac{\partial J(\theta)}{\partial \theta_j}$$

**Batch Gradient Descent:** $$w_j := w_j - \alpha \frac{1}{n} \sum_{i=1}^{n} (h_\theta(\mathbf{x}^{(i)}) - y^{(i)}) x_j^{(i)}$$

**Stochastic Gradient Descent (SGD):** $$w_j := w_j - \alpha (h_\theta(\mathbf{x}^{(i)}) - y^{(i)}) x_j^{(i)}$$

**Mini-batch Gradient Descent:** $$w_j := w_j - \alpha \frac{1}{|B|} \sum_{i \in B} (h_\theta(\mathbf{x}^{(i)}) - y^{(i)}) x_j^{(i)}$$

where $B$ is a mini-batch of examples

---

## Regularization

**Ridge Regression (L2 Regularization):** $$J(\theta) = \frac{1}{2n} \sum_{i=1}^{n} (h_\theta(\mathbf{x}^{(i)}) - y^{(i)})^2 + \frac{\lambda}{2n} \sum_{j=1}^{m} w_j^2$$

**Lasso Regression (L1 Regularization):** $$J(\theta) = \frac{1}{2n} \sum_{i=1}^{n} (h_\theta(\mathbf{x}^{(i)}) - y^{(i)})^2 + \frac{\lambda}{n} \sum_{j=1}^{m} |w_j|$$

**Elastic Net:** $$J(\theta) = \frac{1}{2n} \sum_{i=1}^{n} (h_\theta(\mathbf{x}^{(i)}) - y^{(i)})^2 + \lambda_1 \sum_{j=1}^{m} |w_j| + \frac{\lambda_2}{2} \sum_{j=1}^{m} w_j^2$$

---

## Logistic Regression

**Sigmoid Function:** $$\sigma(z) = \frac{1}{1 + e^{-z}}$$

**Hypothesis (Binary Classification):** $$h_\theta(\mathbf{x}) = \sigma(\mathbf{w}^T\mathbf{x} + b) = \frac{1}{1 + e^{-(\mathbf{w}^T\mathbf{x} + b)}}$$

**Binary Cross-Entropy Loss (Log Loss):** $$J(\theta) = -\frac{1}{n} \sum_{i=1}^{n} [y^{(i)} \log(h_\theta(\mathbf{x}^{(i)})) + (1-y^{(i)}) \log(1-h_\theta(\mathbf{x}^{(i)}))]$$

**Gradient for Logistic Regression:** $$\frac{\partial J}{\partial w_j} = \frac{1}{n} \sum_{i=1}^{n} (h_\theta(\mathbf{x}^{(i)}) - y^{(i)}) x_j^{(i)}$$

**Regularized Logistic Regression:** $$J(\theta) = -\frac{1}{n} \sum_{i=1}^{n} [y^{(i)} \log(h_\theta(\mathbf{x}^{(i)})) + (1-y^{(i)}) \log(1-h_\theta(\mathbf{x}^{(i)}))] + \frac{\lambda}{2n} \sum_{j=1}^{m} w_j^2$$

---

## Softmax Regression (Multinomial Logistic Regression)

**Softmax Function:** $$P(y=c|\mathbf{x}) = \frac{e^{\mathbf{w}_c^T\mathbf{x}}}{\sum_{j=1}^{k} e^{\mathbf{w}_j^T\mathbf{x}}}$$

**Cross-Entropy Loss (Multi-class):** $$J(\theta) = -\frac{1}{n} \sum_{i=1}^{n} \sum_{c=1}^{k} \mathbb{1}{y^{(i)}=c} \log P(y=c|\mathbf{x}^{(i)})$$

---

## Support Vector Machines (SVM)

**Linear SVM (Hard Margin):** $$\min_{\mathbf{w}, b} \frac{1}{2} ||\mathbf{w}||^2$$ subject to: $y^{(i)}(\mathbf{w}^T\mathbf{x}^{(i)} + b) \geq 1$ for all $i$

**Linear SVM (Soft Margin):** $$\min_{\mathbf{w}, b, \xi} \frac{1}{2} ||\mathbf{w}||^2 + C \sum_{i=1}^{n} \xi_i$$ subject to: $y^{(i)}(\mathbf{w}^T\mathbf{x}^{(i)} + b) \geq 1 - \xi_i$ and $\xi_i \geq 0$

**Hinge Loss:** $$L(y, \hat{y}) = \max(0, 1 - y \cdot \hat{y})$$

**Total Loss (SVM):** $$J(\mathbf{w}) = C \sum_{i=1}^{n} \max(0, 1 - y^{(i)}(\mathbf{w}^T\mathbf{x}^{(i)} + b)) + \frac{1}{2}||\mathbf{w}||^2$$

**Kernel Trick:** $$K(\mathbf{x}^{(i)}, \mathbf{x}^{(j)}) = \phi(\mathbf{x}^{(i)})^T \phi(\mathbf{x}^{(j)})$$

**RBF (Gaussian) Kernel:** $$K(\mathbf{x}^{(i)}, \mathbf{x}^{(j)}) = \exp\left(-\frac{||\mathbf{x}^{(i)} - \mathbf{x}^{(j)}||^2}{2\sigma^2}\right)$$

**Polynomial Kernel:** $$K(\mathbf{x}^{(i)}, \mathbf{x}^{(j)}) = (\mathbf{x}^{(i)T}\mathbf{x}^{(j)} + c)^d$$

---

## Neural Networks

**Linear Transformation:** $$\mathbf{z}^{[l]} = \mathbf{W}^{[l]}\mathbf{a}^{[l-1]} + \mathbf{b}^{[l]}$$

**Activation Functions:**

- ReLU: $a = \max(0, z)$
- Leaky ReLU: $a = \max(\alpha z, z)$ where $\alpha$ is small (e.g., 0.01)
- Tanh: $a = \tanh(z) = \frac{e^z - e^{-z}}{e^z + e^{-z}}$
- Sigmoid: $a = \sigma(z) = \frac{1}{1+e^{-z}}$

**Forward Propagation:** $$\mathbf{a}^{[l]} = g^{[l]}(\mathbf{z}^{[l]})$$

where $g^{[l]}$ is the activation function at layer $l$

**Cross-Entropy Loss (Neural Networks):** $$J = -\frac{1}{n} \sum_{i=1}^{n} \sum_{c=1}^{k} y_c^{(i)} \log(\hat{y}_c^{(i)})$$

**Backpropagation (Output Layer):** $$\delta^{[L]} = \mathbf{a}^{[L]} - \mathbf{y}$$

**Backpropagation (Hidden Layers):** $$\delta^{[l]} = (\mathbf{W}^{[l+1]T}\delta^{[l+1]}) \odot g'^{[l]}(\mathbf{z}^{[l]})$$

where $\odot$ denotes element-wise multiplication

**Gradient Computation:** $$\frac{\partial J}{\partial \mathbf{W}^{[l]}} = \frac{1}{n} \delta^{[l]} \mathbf{a}^{[l-1]T}$$

$$\frac{\partial J}{\partial \mathbf{b}^{[l]}} = \frac{1}{n} \sum_{i=1}^{n} \delta^{[l](https://claude.ai/chat/i)}$$

**Dropout:** $$\mathbf{a}^{[l]} = \mathbf{a}^{[l]} \odot \mathbf{d}^{[l]}$$

where $\mathbf{d}^{[l]} \sim \text{Bernoulli}(p)$

---

## Optimization Algorithms

**Momentum:** $$\mathbf{v}_t = \beta \mathbf{v}_{t-1} + (1-\beta) \nabla J(\theta_t)$$ $$\theta_{t+1} = \theta_t - \alpha \mathbf{v}_t$$

**RMSprop:** $$\mathbf{s}_t = \beta \mathbf{s}_{t-1} + (1-\beta) (\nabla J(\theta_t))^2$$ $$\theta_{t+1} = \theta_t - \alpha \frac{\nabla J(\theta_t)}{\sqrt{\mathbf{s}_t + \epsilon}}$$

**Adam:** $$\mathbf{m}_t = \beta_1 \mathbf{m}_{t-1} + (1-\beta_1) \nabla J(\theta_t)$$ $$\mathbf{v}_t = \beta_2 \mathbf{v}_{t-1} + (1-\beta_2) (\nabla J(\theta_t))^2$$ $$\hat{\mathbf{m}}_t = \frac{\mathbf{m}_t}{1-\beta_1^t}, \quad \hat{\mathbf{v}}_t = \frac{\mathbf{v}_t}{1-\beta_2^t}$$ $$\theta_{t+1} = \theta_t - \alpha \frac{\hat{\mathbf{m}}_t}{\sqrt{\hat{\mathbf{v}}_t} + \epsilon}$$

---

## Decision Trees

**Entropy:** $$H(S) = -\sum_{c=1}^{k} p_c \log_2(p_c)$$

where $p_c$ is the proportion of class $c$ in set $S$

**Information Gain:** $$IG(S, A) = H(S) - \sum_{v \in \text{Values}(A)} \frac{|S_v|}{|S|} H(S_v)$$

**Gini Impurity:** $$\text{Gini}(S) = 1 - \sum_{c=1}^{k} p_c^2$$

**Gini Gain:** $$\text{Gain}_{\text{Gini}}(S, A) = \text{Gini}(S) - \sum_{v \in \text{Values}(A)} \frac{|S_v|}{|S|} \text{Gini}(S_v)$$

**Variance Reduction (Regression Trees):** $$\text{Var}(S) = \frac{1}{|S|} \sum_{i \in S} (y^{(i)} - \bar{y})^2$$

---

## Ensemble Methods

**Bagging Prediction:** $$\hat{y} = \frac{1}{T} \sum_{t=1}^{T} h_t(\mathbf{x})$$

**Random Forest (Classification - Majority Vote):** $$\hat{y} = \text{mode}{h_1(\mathbf{x}), h_2(\mathbf{x}), ..., h_T(\mathbf{x})}$$

**AdaBoost Weight Update:** $$w^{(i)} := w^{(i)} \exp(\alpha_t \mathbb{1}{h_t(\mathbf{x}^{(i)}) \neq y^{(i)}})$$

**AdaBoost Classifier Weight:** $$\alpha_t = \frac{1}{2} \ln\left(\frac{1-\epsilon_t}{\epsilon_t}\right)$$

where $\epsilon_t$ is the weighted error rate

**AdaBoost Final Prediction:** $$H(\mathbf{x}) = \text{sign}\left(\sum_{t=1}^{T} \alpha_t h_t(\mathbf{x})\right)$$

**Gradient Boosting:** $$F_m(\mathbf{x}) = F_{m-1}(\mathbf{x}) + \gamma_m h_m(\mathbf{x})$$

where $h_m$ fits the negative gradient: $-\frac{\partial L(y, F_{m-1}(\mathbf{x}))}{\partial F_{m-1}(\mathbf{x})}$

---

## K-Nearest Neighbors (KNN)

**Euclidean Distance:** $$d(\mathbf{x}^{(i)}, \mathbf{x}^{(j)}) = \sqrt{\sum_{l=1}^{m} (x_l^{(i)} - x_l^{(j)})^2}$$

**Manhattan Distance:** $$d(\mathbf{x}^{(i)}, \mathbf{x}^{(j)}) = \sum_{l=1}^{m} |x_l^{(i)} - x_l^{(j)}|$$

**KNN Classification:** $$\hat{y} = \text{mode}{y^{(i)} : \mathbf{x}^{(i)} \in N_k(\mathbf{x})}$$

**KNN Regression:** $$\hat{y} = \frac{1}{k} \sum_{\mathbf{x}^{(i)} \in N_k(\mathbf{x})} y^{(i)}$$

---

## Naive Bayes

**Bayes' Theorem:** $$P(y|\mathbf{x}) = \frac{P(\mathbf{x}|y)P(y)}{P(\mathbf{x})}$$

**Naive Bayes Classifier:** $$\hat{y} = \arg\max_{y} P(y) \prod_{j=1}^{m} P(x_j|y)$$

**Gaussian Naive Bayes:** $$P(x_j|y=c) = \frac{1}{\sqrt{2\pi\sigma_{jc}^2}} \exp\left(-\frac{(x_j - \mu_{jc})^2}{2\sigma_{jc}^2}\right)$$

---

## Evaluation Metrics

**Accuracy:** $$\text{Accuracy} = \frac{\text{Number of correct predictions}}{n}$$

**Precision:** $$\text{Precision} = \frac{TP}{TP + FP}$$

**Recall (Sensitivity, True Positive Rate):** $$\text{Recall} = \frac{TP}{TP + FN}$$

**F1-Score:** $$F_1 = 2 \cdot \frac{\text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}} = \frac{2TP}{2TP + FP + FN}$$

**F-beta Score:** $$F_\beta = (1 + \beta^2) \cdot \frac{\text{Precision} \cdot \text{Recall}}{\beta^2 \cdot \text{Precision} + \text{Recall}}$$

**Specificity (True Negative Rate):** $$\text{Specificity} = \frac{TN}{TN + FP}$$

**False Positive Rate:** $$FPR = \frac{FP}{FP + TN} = 1 - \text{Specificity}$$

**Matthews Correlation Coefficient:** $$MCC = \frac{TP \cdot TN - FP \cdot FN}{\sqrt{(TP+FP)(TP+FN)(TN+FP)(TN+FN)}}$$

---

## Bias-Variance Tradeoff

**Expected Prediction Error:** $$\mathbb{E}[(y - \hat{f}(\mathbf{x}))^2] = \text{Bias}[\hat{f}(\mathbf{x})]^2 + \text{Var}[\hat{f}(\mathbf{x})] + \sigma^2$$

**Bias:** $$\text{Bias}[\hat{f}(\mathbf{x})] = \mathbb{E}[\hat{f}(\mathbf{x})] - f(\mathbf{x})$$

**Variance:** $$\text{Var}[\hat{f}(\mathbf{x})] = \mathbb{E}[(\hat{f}(\mathbf{x}) - \mathbb{E}[\hat{f}(\mathbf{x})])^2]$$

---

## Cross-Validation

**K-Fold Cross-Validation Error:** $$CV_{(k)} = \frac{1}{k} \sum_{i=1}^{k} \text{Error}_i$$

**Leave-One-Out Cross-Validation (LOOCV):** $$CV_{(n)} = \frac{1}{n} \sum_{i=1}^{n} (y^{(i)} - \hat{y}^{(i)}_{-i})^2$$

## Dimensionality Reduction

### Principal Component Analysis (PCA)

**Covariance Matrix:** $$\Sigma = \frac{1}{n} \mathbf{X}^T\mathbf{X}$$

where $\mathbf{X}$ is the centered data matrix

**Principal Components (Eigenvector Problem):** $$\Sigma \mathbf{v} = \lambda \mathbf{v}$$

**Projection onto k components:** $$\mathbf{z}^{(i)} = \mathbf{U}_k^T \mathbf{x}^{(i)}$$

where $\mathbf{U}_k$ contains the first $k$ eigenvectors

**Reconstruction:** $$\hat{\mathbf{x}}^{(i)} = \mathbf{U}_k \mathbf{z}^{(i)}$$

**Explained Variance Ratio:** $$\text{EVR}_k = \frac{\sum_{j=1}^{k} \lambda_j}{\sum_{j=1}^{m} \lambda_j}$$

**Reconstruction Error:** $$J = \frac{1}{n} \sum_{i=1}^{n} ||\mathbf{x}^{(i)} - \hat{\mathbf{x}}^{(i)}||^2$$

---

### Linear Discriminant Analysis (LDA)

**Between-Class Scatter Matrix:** $$S_B = \sum_{c=1}^{k} n_c (\boldsymbol{\mu}_c - \boldsymbol{\mu})(\boldsymbol{\mu}_c - \boldsymbol{\mu})^T$$

**Within-Class Scatter Matrix:** $$S_W = \sum_{c=1}^{k} \sum_{\mathbf{x} \in C_c} (\mathbf{x} - \boldsymbol{\mu}_c)(\mathbf{x} - \boldsymbol{\mu}_c)^T$$

**LDA Objective (Fisher's Criterion):** $$\mathbf{w}^* = \arg\max_{\mathbf{w}} \frac{\mathbf{w}^T S_B \mathbf{w}}{\mathbf{w}^T S_W \mathbf{w}}$$

**Generalized Eigenvalue Problem:** $$S_B \mathbf{w} = \lambda S_W \mathbf{w}$$

---

## Feature Selection

**Pearson Correlation Coefficient:** $$r_{xy} = \frac{\sum_{i=1}^{n}(x^{(i)} - \bar{x})(y^{(i)} - \bar{y})}{\sqrt{\sum_{i=1}^{n}(x^{(i)} - \bar{x})^2}\sqrt{\sum_{i=1}^{n}(y^{(i)} - \bar{y})^2}}$$

**Chi-Square Statistic:** $$\chi^2 = \sum_{i=1}^{r}\sum_{j=1}^{c} \frac{(O_{ij} - E_{ij})^2}{E_{ij}}$$

where $O_{ij}$ is observed frequency and $E_{ij}$ is expected frequency

**Mutual Information:** $$I(X;Y) = \sum_{x \in X} \sum_{y \in Y} p(x,y) \log\frac{p(x,y)}{p(x)p(y)}$$

---

## Batch Normalization

**Batch Mean:** $$\mu_B = \frac{1}{|B|} \sum_{i \in B} x^{(i)}$$

**Batch Variance:** $$\sigma_B^2 = \frac{1}{|B|} \sum_{i \in B} (x^{(i)} - \mu_B)^2$$

**Normalization:** $$\hat{x}^{(i)} = \frac{x^{(i)} - \mu_B}{\sqrt{\sigma_B^2 + \epsilon}}$$

**Scale and Shift:** $$y^{(i)} = \gamma \hat{x}^{(i)} + \beta$$

---

## Learning Rate Schedules

**Step Decay:** $$\alpha_t = \alpha_0 \cdot \text{drop}^{\lfloor \frac{t}{\text{epochs_drop}} \rfloor}$$

**Exponential Decay:** $$\alpha_t = \alpha_0 e^{-kt}$$

**Time-Based Decay:** $$\alpha_t = \frac{\alpha_0}{1 + kt}$$

**Polynomial Decay:** $$\alpha_t = \alpha_0 \left(1 - \frac{t}{T}\right)^p$$

**Cosine Annealing:** $$\alpha_t = \alpha_{\min} + \frac{1}{2}(\alpha_{\max} - \alpha_{\min})\left(1 + \cos\left(\frac{t\pi}{T}\right)\right)$$

---

## Loss Functions (Additional)

**Huber Loss:** $$L_\delta(y, \hat{y}) = \begin{cases} \frac{1}{2}(y - \hat{y})^2 & \text{if } |y - \hat{y}| \leq \delta \ \delta(|y - \hat{y}| - \frac{1}{2}\delta) & \text{otherwise} \end{cases}$$

**Log-Cosh Loss:** $$L(y, \hat{y}) = \sum_{i=1}^{n} \log(\cosh(\hat{y}^{(i)} - y^{(i)}))$$

**Quantile Loss (Pinball Loss):** $$L_\tau(y, \hat{y}) = \sum_{i=1}^{n} \max[\tau(y^{(i)} - \hat{y}^{(i)}), (\tau-1)(y^{(i)} - \hat{y}^{(i)})]$$

**Focal Loss (for imbalanced classification):** $$FL(p_t) = -\alpha_t(1-p_t)^\gamma \log(p_t)$$

where $p_t = p$ if $y=1$ else $1-p$

**Kullback-Leibler Divergence:** $$D_{KL}(P||Q) = \sum_{i} P(i) \log\frac{P(i)}{Q(i)}$$

**Triplet Loss (for metric learning):** $$L = \max(d(a,p) - d(a,n) + \text{margin}, 0)$$

where $a$ is anchor, $p$ is positive, $n$ is negative

---

## Distance Metrics

**Minkowski Distance:** $$d(\mathbf{x}, \mathbf{y}) = \left(\sum_{i=1}^{m} |x_i - y_i|^p\right)^{1/p}$$

**Cosine Similarity:** $$\text{sim}(\mathbf{x}, \mathbf{y}) = \frac{\mathbf{x} \cdot \mathbf{y}}{||\mathbf{x}|| \cdot ||\mathbf{y}||} = \frac{\sum_{i=1}^{m} x_i y_i}{\sqrt{\sum_{i=1}^{m} x_i^2}\sqrt{\sum_{i=1}^{m} y_i^2}}$$

**Cosine Distance:** $$d(\mathbf{x}, \mathbf{y}) = 1 - \text{sim}(\mathbf{x}, \mathbf{y})$$

**Mahalanobis Distance:** $$d(\mathbf{x}, \mathbf{y}) = \sqrt{(\mathbf{x} - \mathbf{y})^T \Sigma^{-1} (\mathbf{x} - \mathbf{y})}$$

**Hamming Distance:** $$d(\mathbf{x}, \mathbf{y}) = \sum_{i=1}^{m} \mathbb{1}{x_i \neq y_i}$$

**Jaccard Similarity:** $$J(A, B) = \frac{|A \cap B|}{|A \cup B|}$$

**Jaccard Distance:** $$d_J(A, B) = 1 - J(A, B)$$

---

## Probability and Statistics

**Maximum Likelihood Estimation (MLE):** $$\hat{\theta}_{MLE} = \arg\max_\theta \prod_{i=1}^{n} P(x^{(i)}|\theta) = \arg\max_\theta \sum_{i=1}^{n} \log P(x^{(i)}|\theta)$$

**Maximum A Posteriori (MAP) Estimation:** $$\hat{\theta}_{MAP} = \arg\max_\theta P(\theta|\mathbf{X}) = \arg\max_\theta P(\mathbf{X}|\theta)P(\theta)$$

**Log-Likelihood:** $$\ell(\theta) = \log L(\theta) = \sum_{i=1}^{n} \log P(x^{(i)}|\theta)$$

**Expected Value:** $$\mathbb{E}[X] = \sum_{x} x \cdot P(X=x) \quad \text{or} \quad \int_{-\infty}^{\infty} x \cdot f(x) dx$$

**Variance:** $$\text{Var}(X) = \mathbb{E}[(X - \mathbb{E}[X])^2] = \mathbb{E}[X^2] - (\mathbb{E}[X])^2$$

**Covariance:** $$\text{Cov}(X, Y) = \mathbb{E}[(X - \mathbb{E}[X])(Y - \mathbb{E}[Y])]$$

**Standard Error:** $$SE = \frac{\sigma}{\sqrt{n}}$$

---

## Confidence Intervals

**Confidence Interval for Mean:** $$CI = \bar{x} \pm z_{\alpha/2} \frac{\sigma}{\sqrt{n}}$$

**Confidence Interval (t-distribution):** $$CI = \bar{x} \pm t_{\alpha/2, n-1} \frac{s}{\sqrt{n}}$$

---

## Hypothesis Testing

**Z-Score:** $$z = \frac{\bar{x} - \mu}{\sigma/\sqrt{n}}$$

**T-Statistic:** $$t = \frac{\bar{x} - \mu}{s/\sqrt{n}}$$

**P-Value Relationship:** $$p\text{-value} = P(|T| \geq |t_{obs}| | H_0 \text{ is true})$$

---

## Bootstrap and Resampling

**Bootstrap Estimate:** $$\hat{\theta}^* = \frac{1}{B} \sum_{b=1}^{B} \hat{\theta}^{*(b)}$$

**Bootstrap Standard Error:** $$SE_{boot}(\hat{\theta}) = \sqrt{\frac{1}{B-1} \sum_{b=1}^{B} (\hat{\theta}^{_(b)} - \hat{\theta}^_)^2}$$

---

## ROC and AUC

**True Positive Rate (Sensitivity/Recall):** $$TPR = \frac{TP}{TP + FN}$$

**False Positive Rate:** $$FPR = \frac{FP}{FP + TN}$$

**Area Under ROC Curve (AUC):** $$AUC = \int_{0}^{1} TPR(FPR^{-1}(x)) dx$$

**Gini Coefficient (from AUC):** $$\text{Gini} = 2 \cdot AUC - 1$$

---

## Confusion Matrix Metrics

**Positive Predictive Value (Precision):** $$PPV = \frac{TP}{TP + FP}$$

**Negative Predictive Value:** $$NPV = \frac{TN}{TN + FN}$$

**False Discovery Rate:** $$FDR = \frac{FP}{FP + TP} = 1 - PPV$$

**False Omission Rate:** $$FOR = \frac{FN}{FN + TN} = 1 - NPV$$

**Balanced Accuracy:** $$BA = \frac{TPR + TNR}{2}$$

---

## Multi-Class Classification Metrics

**Macro-Average Precision:** $$P_{macro} = \frac{1}{k} \sum_{c=1}^{k} P_c$$

**Micro-Average Precision:** $$P_{micro} = \frac{\sum_{c=1}^{k} TP_c}{\sum_{c=1}^{k} (TP_c + FP_c)}$$

**Weighted Average:** $$P_{weighted} = \frac{1}{n} \sum_{c=1}^{k} n_c \cdot P_c$$

**Cohen's Kappa:** $$\kappa = \frac{p_o - p_e}{1 - p_e}$$

where $p_o$ is observed agreement and $p_e$ is expected agreement

---

## Probabilistic Predictions

**Log Loss (Binary):** $$\text{LogLoss} = -\frac{1}{n}\sum_{i=1}^{n}[y^{(i)}\log(p^{(i)}) + (1-y^{(i)})\log(1-p^{(i)})]$$

**Brier Score:** $$BS = \frac{1}{n}\sum_{i=1}^{n}(p^{(i)} - y^{(i)})^2$$

**Calibration Error:** $$ECE = \sum_{m=1}^{M} \frac{|B_m|}{n} |\text{acc}(B_m) - \text{conf}(B_m)|$$

---

## Feature Scaling

**Min-Max Normalization:** $$x' = \frac{x - x_{min}}{x_{max} - x_{min}}$$

**Standardization (Z-score normalization):** $$x' = \frac{x - \mu}{\sigma}$$

**Robust Scaling:** $$x' = \frac{x - Q_2}{Q_3 - Q_1}$$

where $Q_1, Q_2, Q_3$ are the 1st, 2nd (median), and 3rd quartiles

**Max Abs Scaling:** $$x' = \frac{x}{|x|_{max}}$$

**L2 Normalization:** $$\mathbf{x}' = \frac{\mathbf{x}}{||\mathbf{x}||_2}$$

---

## Polynomial Features

**Polynomial Expansion (degree 2):** $$\phi(\mathbf{x}) = [1, x_1, x_2, ..., x_m, x_1^2, x_1x_2, ..., x_m^2]$$

**General Polynomial Features:** $$\phi_d(\mathbf{x}) = {x_{i_1}x_{i_2}...x_{i_k} : 1 \leq i_1 \leq i_2 \leq ... \leq i_k \leq m, k \leq d}$$

---

## Information Theory Metrics

**Entropy (Continuous):** $$H(X) = -\int f(x) \log f(x) dx$$

**Conditional Entropy:** $$H(Y|X) = \sum_{x} P(x) H(Y|X=x) = -\sum_{x,y} P(x,y) \log P(y|x)$$

**Joint Entropy:** $$H(X,Y) = -\sum_{x,y} P(x,y) \log P(x,y)$$

**Cross-Entropy:** $$H(P, Q) = -\sum_{x} P(x) \log Q(x)$$

---

## Imbalanced Learning

**SMOTE (Synthetic Minority Over-sampling):** $$\mathbf{x}_{new} = \mathbf{x}_i + \lambda(\mathbf{x}_{nn} - \mathbf{x}_i)$$

where $\lambda \in [0,1]$ and $\mathbf{x}_{nn}$ is a nearest neighbor

**Class Weight:** $$w_c = \frac{n}{k \cdot n_c}$$

where $k$ is number of classes and $n_c$ is number of samples in class $c$

**Balanced Accuracy:** $$BA = \frac{1}{k}\sum_{c=1}^{k} \frac{TP_c}{TP_c + FN_c}$$

---

## Time Series Forecasting (Supervised Learning Context)

**Autoregressive Model (AR):** $$y_t = c + \sum_{i=1}^{p} \phi_i y_{t-i} + \epsilon_t$$

**Moving Average Model (MA):** $$y_t = \mu + \sum_{i=1}^{q} \theta_i \epsilon_{t-i} + \epsilon_t$$

**Mean Absolute Percentage Error (MAPE):** $$MAPE = \frac{100%}{n} \sum_{i=1}^{n} \left|\frac{y^{(i)} - \hat{y}^{(i)}}{y^{(i)}}\right|$$

**Symmetric MAPE:** $$SMAPE = \frac{100%}{n} \sum_{i=1}^{n} \frac{|y^{(i)} - \hat{y}^{(i)}|}{(|y^{(i)}| + |\hat{y}^{(i)}|)/2}$$

---

## Isotonic Regression

**Isotonic Regression Objective:** $$\min_{\hat{y}} \sum_{i=1}^{n} (y^{(i)} - \hat{y}^{(i)})^2$$

subject to: $\hat{y}^{(1)} \leq \hat{y}^{(2)} \leq ... \leq \hat{y}^{(n)}$

---

## Clustering (Semi-supervised Context)

**K-Means Objective:** $$J = \sum_{i=1}^{n} \sum_{j=1}^{k} r_{ij} ||\mathbf{x}^{(i)} - \boldsymbol{\mu}_j||^2$$

where $r_{ij} = 1$ if point $i$ belongs to cluster $j$, else 0

**Silhouette Score:** $$s(i) = \frac{b(i) - a(i)}{\max{a(i), b(i)}}$$

where $a(i)$ is mean intra-cluster distance and $b(i)$ is mean nearest-cluster distance

---

## Transfer Learning

**Fine-tuning Loss:** $$J_{total} = J_{task} + \lambda J_{reg}$$

**Domain Adaptation (MMD - Maximum Mean Discrepancy):** $$MMD(X_s, X_t) = \left|\left|\frac{1}{n_s}\sum_{i=1}^{n_s}\phi(\mathbf{x}_i^s) - \frac{1}{n_t}\sum_{j=1}^{n_t}\phi(\mathbf{x}_j^t)\right|\right||$$

---

## Early Stopping

**Validation Loss Criterion:** $$\text{Stop if: } L_{val}(t) > \min_{i<t} L_{val}(i) + \delta \text{ for } p \text{ consecutive epochs}$$

---

## Gradient Clipping

**Norm-based Clipping:** $$\mathbf{g} = \begin{cases} \mathbf{g} & \text{if } ||\mathbf{g}|| \leq \theta \ \theta \frac{\mathbf{g}}{||\mathbf{g}||} & \text{otherwise} \end{cases}$$

**Value-based Clipping:** $$g_i = \max(\min(g_i, \theta), -\theta)$$

---

## Learning Curves

**Training Curve:** $$L_{train}(m) = \frac{1}{m}\sum_{i=1}^{m} L(y^{(i)}, \hat{y}^{(i)})$$

**Validation Curve:** $$L_{val}(\lambda) = \text{Average validation error at regularization } \lambda$$
