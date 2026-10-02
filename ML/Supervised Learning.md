
## Regression

- Linear Regression
- Multiple Linear Regression
- Polynomial Regression
- Ridge Regression (L2 regularization)
- Lasso Regression (L1 regularization)
- Elastic Net Regression
- Support Vector Regression (SVR)
- Decision Tree Regression
- Random Forest Regression
- Gradient Boosting Regression
	- XGBoost
	- LightGBM
	- CatBoost
- Neural Network Regression
- Bayesian Regression

## Classification

- Logistic Regression
- K-Nearest Neighbors (KNN)
- Naive Bayes (Gaussian, Multinomial, Bernoulli)
- Decision Trees
- Random Forests
- Support Vector Machines (SVM)
- Gradient Boosting Classifiers (XGBoost, LightGBM, AdaBoost)
- Neural Networks (MLPs)
- Perceptron
- Discriminant Analysis (LDA, QDA)

### **Ensemble Methods**

- Bagging (Bootstrap Aggregating)
- Random Forests
- Boosting:
    - AdaBoost
    - Gradient Boosting
    - XGBoost
    - LightGBM
    - CatBoost
- Stacking
- Voting Classifiers (Hard/Soft)

### **Instance-Based Learning**

- K-Nearest Neighbors (KNN)
- Learning Vector Quantization (LVQ)
- Self-Organizing Maps (SOM)

### **Probabilistic Models**

- Naive Bayes
- Bayesian Networks
- Hidden Markov Models (HMM)
- Gaussian Processes

### **Neural Networks & Deep Learning**

- Multi-Layer Perceptrons (MLP)
- Convolutional Neural Networks (CNN)
- Recurrent Neural Networks (RNN)
- Long Short-Term Memory (LSTM)
- Gated Recurrent Units (GRU)
- Transformers
- Residual Networks (ResNet)
- Attention Mechanisms

### **Time Series & Sequential**

- ARIMA (AutoRegressive Integrated Moving Average)
- SARIMA (Seasonal ARIMA)
- Exponential Smoothing
- Prophet (Facebook)
- LSTM for Time Series
- Temporal Convolutional Networks (TCN)

### **Kernel Methods**

- Support Vector Machines (SVM)
- Kernel Ridge Regression
- Gaussian Processes
- Kernel PCA

## **UNSUPERVISED LEARNING**

### **Clustering**

- K-Means
- K-Medoids (PAM)
- Hierarchical Clustering:
    - Agglomerative (bottom-up)
    - Divisive (top-down)
- DBSCAN (Density-Based)
- OPTICS
- Mean Shift
- Gaussian Mixture Models (GMM)
- Spectral Clustering
- Affinity Propagation
- BIRCH (Balanced Iterative Reducing and Clustering using Hierarchies)
- Fuzzy C-Means

### **Dimensionality Reduction**

- Principal Component Analysis (PCA)
- Kernel PCA
- Incremental PCA
- t-SNE (t-Distributed Stochastic Neighbor Embedding)
- UMAP (Uniform Manifold Approximation and Projection)
- Linear Discriminant Analysis (LDA)
- Independent Component Analysis (ICA)
- Factor Analysis
- Truncated SVD
- Isomap
- Locally Linear Embedding (LLE)
- Multidimensional Scaling (MDS)
- Autoencoders
- Variational Autoencoders (VAE)

### **Anomaly/Outlier Detection**

- Isolation Forest
- Local Outlier Factor (LOF)
- One-Class SVM
- Elliptic Envelope
- DBSCAN (for outliers)
- Autoencoders for Anomaly Detection
- Statistical Methods (Z-score, IQR)

### **Association Rule Learning**

- Apriori Algorithm
- FP-Growth (Frequent Pattern Growth)
- Eclat
- Market Basket Analysis

### **Generative Models**

- Variational Autoencoders (VAE)
- Generative Adversarial Networks (GANs):
    - Standard GAN
    - DCGAN (Deep Convolutional GAN)
    - WGAN (Wasserstein GAN)
    - StyleGAN
    - CycleGAN
    - Conditional GAN
- Diffusion Models (DDPM, Stable Diffusion)
- Normalizing Flows
- Boltzmann Machines
- Restricted Boltzmann Machines (RBM)

### **Density Estimation**

- Kernel Density Estimation (KDE)
- Gaussian Mixture Models (GMM)
- Expectation-Maximization (EM) Algorithm
- Histograms
- Parzen Windows

### **Matrix Factorization**

- Singular Value Decomposition (SVD)
- Non-negative Matrix Factorization (NMF)
- Probabilistic Matrix Factorization
- Collaborative Filtering (for recommendations)
- Tensor Decomposition

### **Topic Modeling**

- Latent Dirichlet Allocation (LDA)
- Non-negative Matrix Factorization (NMF)
- Latent Semantic Analysis (LSA)
- Probabilistic Latent Semantic Analysis (PLSA)
- BERTopic
- Top2Vec

### **Representation Learning**

- Autoencoders
- Denoising Autoencoders
- Sparse Autoencoders
- Variational Autoencoders (VAE)
- Word2Vec (Skip-gram, CBOW)
- GloVe
- FastText
- BERT (for embeddings)
- Node2Vec (Graph embeddings)
- DeepWalk


## **SEMI-SUPERVISED LEARNING**

### **Self-Training Methods**

- Pseudo-labeling
- Self-training with confidence thresholds
- Iterative self-training
- Teacher-Student models
- Noisy Student Training

### **Co-Training Methods**

- Multi-view Co-training
- Co-EM (Co-training with Expectation Maximization)
- Tri-training (three classifiers)

### **Graph-Based Methods**

- Label Propagation
- Label Spreading
- Graph Convolutional Networks (GCN) for semi-supervised
- Transductive SVM

### **Consistency Regularization**

- Π-Model (Pi-Model)
- Temporal Ensembling
- Mean Teacher
- Virtual Adversarial Training (VAT)
- UDA (Unsupervised Data Augmentation)

### **Hybrid Methods**

- MixMatch
- FixMatch
- ReMixMatch
- FlexMatch
- Pseudo-Label + Consistency

### **Generative Approaches**

- Semi-supervised GANs
- Semi-supervised VAE
- Ladder Networks

---

## **REINFORCEMENT LEARNING (RL)**

### **Value-Based Methods**

- Q-Learning
- SARSA (State-Action-Reward-State-Action)
- Deep Q-Networks (DQN)
- Double DQN
- Dueling DQN
- Rainbow DQN
- Prioritized Experience Replay

### **Policy-Based Methods**

- REINFORCE
- Policy Gradient Methods
- Trust Region Policy Optimization (TRPO)
- Proximal Policy Optimization (PPO)
- A2C (Advantage Actor-Critic)
- A3C (Asynchronous Advantage Actor-Critic)

### **Actor-Critic Methods**

- A2C/A3C
- DDPG (Deep Deterministic Policy Gradient)
- TD3 (Twin Delayed DDPG)
- SAC (Soft Actor-Critic)

### **Model-Based RL**

- Dyna-Q
- PILCO (Probabilistic Inference for Learning Control)
- World Models
- MuZero
- Dreamer

### **Multi-Agent RL**

- Independent Q-Learning
- QMIX
- MADDPG (Multi-Agent DDPG)
- COMA (Counterfactual Multi-Agent)

### **Inverse RL**

- Apprenticeship Learning
- Maximum Entropy IRL
- Generative Adversarial Imitation Learning (GAIL)

### **Hierarchical RL**

- Options Framework
- Feudal RL
- HAM (Hierarchical Abstract Machines)

---

## **SELF-SUPERVISED LEARNING**

### **Contrastive Learning**

- SimCLR
- MoCo (Momentum Contrast)
- MoCo v2, v3
- SwAV
- BYOL (Bootstrap Your Own Latent)
- SimSiam
- Barlow Twins
- VICReg

### **Masked Prediction**

- BERT (Masked Language Modeling)
- MAE (Masked Autoencoders)
- BEiT (BERT Pre-training of Image Transformers)
- SimMIM
- iBOT

### **Predictive Learning**

- Next token prediction (GPT-style)
- Frame prediction (video)
- Rotation prediction
- Jigsaw puzzle solving
- Colorization
- Context prediction (Word2Vec)
- Inpainting

### **Clustering-Based**

- DeepCluster
- SwAV (clustering + contrastive)
- SCAN (Semantic Clustering by Adopting Nearest neighbors)

---

## **TRANSFER LEARNING**

### **Fine-Tuning**

- Full fine-tuning
- Partial fine-tuning (freeze layers)
- Progressive unfreezing
- Discriminative fine-tuning
- LoRA (Low-Rank Adaptation)
- Adapter modules
- Prompt tuning

### **Feature Extraction**

- Using pre-trained models as feature extractors
- Embedding extraction
- Frozen backbone with new head

### **Domain Adaptation**

- DANN (Domain-Adversarial Neural Networks)
- CORAL (Correlation Alignment)
- Maximum Mean Discrepancy (MMD)
- Self-Ensembling for Domain Adaptation
- ADDA (Adversarial Discriminative Domain Adaptation)

### **Multi-Task Transfer**

- Pre-training on source tasks
- Multi-task pre-training
- Cross-lingual transfer (mBERT, XLM-R)

---

## **META-LEARNING (Learning to Learn)**

### **Optimization-Based**

- MAML (Model-Agnostic Meta-Learning)
- Reptile
- First-Order MAML (FOMAML)

### **Metric-Based**

- Siamese Networks
- Matching Networks
- Prototypical Networks
- Relation Networks

### **Memory-Based**

- Neural Turing Machines
- Memory-Augmented Neural Networks (MANN)
- Differentiable Neural Computer (DNC)

### **Model-Based**

- Meta-learning with learned initialization
- Hypernetworks
- Learning optimizers

---

## **FEW-SHOT & ZERO-SHOT LEARNING**

### **Few-Shot Learning**

- Prototypical Networks
- Matching Networks
- Relation Networks
- MAML
- Meta-SGD
- N-way K-shot classification

### **Zero-Shot Learning**

- Attribute-based classification
- Semantic embeddings (Word2Vec, GloVe)
- CLIP (Contrastive Language-Image Pre-training)
- Zero-shot prompting (GPT-3, GPT-4)
- ConSE (Convex Combination of Semantic Embeddings)

### **One-Shot Learning**

- Siamese Networks
- One-shot classification
- Koch et al. method

---

## **ACTIVE LEARNING**

### **Query Strategies**



- Uncertainty Sampling:
    - Least Confidence
    - Margin Sampling
    - Entropy-based
- Query-by-Committee (QBC)
- Expected Model Change
- Expected Error Reduction
- Variance Reduction

### **Batch Mode Active Learning**



- Diverse mini-batch selection
- Core-set selection
- Batch-mode uncertainty sampling

### **Pool-Based & Stream-Based**



- Pool-based active learning
- Stream-based selective sampling
- Membership query synthesis

---

## **ONLINE LEARNING**

### **Sequential Decision Making**

- Perceptron (online)
- Online Gradient Descent
- Follow-the-Regularized-Leader (FTRL)
- Online Passive-Aggressive Algorithms

### **Bandit Algorithms**

- Multi-Armed Bandits (MAB):
    - ε-greedy
    - UCB (Upper Confidence Bound)
    - Thompson Sampling
- Contextual Bandits
- LinUCB

### **Streaming Learning**

- Hoeffding Trees
- Incremental SVM
- River (streaming ML library)
- Mini-batch online learning

---

## **MULTI-TASK LEARNING**

### **Hard Parameter Sharing**

- Shared hidden layers
- Task-specific output layers
- Multi-task CNNs/RNNs

### **Soft Parameter Sharing**

- Cross-stitch networks
- Sluice networks
- Task grouping

### **Task Relationships**

- Task clustering
- Task relationship learning
- Auxiliary task learning

---

## **CONTINUAL / LIFELONG LEARNING**

### **Regularization-Based**

- EWC (Elastic Weight Consolidation)
- SI (Synaptic Intelligence)
- MAS (Memory Aware Synapses)
- PackNet

### **Replay-Based**

- Experience Replay
- Generative Replay
- GEM (Gradient Episodic Memory)
- A-GEM (Averaged GEM)

### **Architecture-Based**

- Progressive Neural Networks
- DEN (Dynamically Expandable Networks)
- Expert Gate

### **Meta-Learning Approaches**

- Meta-continual learning
- OML (Online-aware Meta-learning)

---

## **WEAKLY SUPERVISED LEARNING**

### **Incomplete Supervision**

- Semi-supervised learning methods
- Active learning
- Positive-Unlabeled (PU) learning

### **Inexact Supervision**

- Multiple Instance Learning (MIL)
- Learning from coarse labels
- Label smoothing

### **Inaccurate Supervision**

- Learning with noisy labels
- Confident learning
- Co-teaching
- MentorNet
- CleanNet

---

## **FEDERATED LEARNING**

### **Horizontal Federated Learning**

- FedAvg (Federated Averaging)
- FedProx
- FedMA (Federated Matched Averaging)

### **Vertical Federated Learning**

- Split learning
- Entity resolution + federated learning

### **Privacy-Preserving Methods**

- Differential Privacy in FL
- Secure Multi-Party Computation (SMC)
- Homomorphic Encryption

---

## **ADVERSARIAL LEARNING**

### **Adversarial Training**

- FGSM (Fast Gradient Sign Method)
- PGD (Projected Gradient Descent)
- C&W Attack
- Adversarial training for robustness

### **GANs (Generative Adversarial Networks)**

- Standard GAN
- DCGAN, WGAN, StyleGAN (covered in unsupervised)
- Adversarial domain adaptation

### **Defense Mechanisms**

- Adversarial training
- Defensive distillation
- Input transformations
- Certified defenses

---

## **CURRICULUM LEARNING**

### **Ordering Strategies**

- Easy-to-hard ordering
- Self-paced learning
- Hard example mining
- Competence-based curriculum

### **Automatic Curriculum**

- Teacher-student curriculum
- Automatic difficulty estimation
- Dynamic curriculum