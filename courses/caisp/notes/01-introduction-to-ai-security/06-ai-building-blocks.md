# AI Building Blocks: Algorithms, Models, Data and Compute

Every AI system rests on four pillars: the algorithm that defines how it learns, the model that results from training, the data it learns from, and the computing power that makes training and inference possible.

## Algorithms and Models

**Algorithms** are the core instructions or recipes that define how AI learns from data and how it makes predictions. They range from simple if/else rules to complex mathematical procedures, and the algorithm is chosen based on the specific problem to solve.

**A model** is the end result of applying an algorithm to a dataset: it is what you get after sending training data through an algorithm. For example, a model might classify images into different categories or predict the likelihood of a certain outcome.

Models can be continuously improved with more data, adjustments to the algorithm, or by optimising parameters during the training process (tuning).

**Why they matter:** choosing and building them well is what makes an AI system effective in terms of accuracy, efficiency and specialisation for its task.

## Data

In the context of AI, algorithms and models, data is the raw information used to train, generate and test models. Data is the fuel that powers AI: without high quality data, even the most advanced algorithms cannot produce accurate or meaningful results.

Data enables AI models to learn patterns, make predictions and adapt to different scenarios. Training data should reflect the real life situations where the AI will be applied.

**Data quality is critical for model training:**

1. Clean, properly labelled and comprehensive data allows models to learn genuine patterns rather than misleading correlations. Example: a medical diagnosis model needs accurate patient records (conditions, demographics) to be reliable.
2. Diverse and representative data reduces bias. Example: a hiring algorithm trained on biased historical data could perpetuate hiring discrimination, while well structured, diverse data can make hiring fairer.

**Adaptability, feedback and improvement:** models improve over time by learning from real world use and feedback, keeping them aligned with reality as it changes.

## Computing Power

AI systems require substantial computing power, especially for tasks like training large models on vast datasets. Computing power refers to the hardware and software needed to train AI models and to make accurate predictions and decisions (inference).

**Hardware requirements:**

1. CPUs: handle general computing tasks
2. GPUs: enable parallel processing, essential for training deep neural networks
3. TPUs (Tensor Processing Units): specialised chips developed by Google, designed specifically for AI and ML workloads and faster than GPUs for some tasks
4. Cloud computing: many organisations access GPUs, TPUs and other resources through cloud providers, gaining scalability without investing in their own hardware

**Software requirements:**

1. ML frameworks and libraries such as TensorFlow, PyTorch and scikit learn provide the tools to build, train and deploy AI models
2. Supporting software includes operating systems, virtualisation and orchestration for running large scale clusters of machines

**Why it matters:** computing power determines the cost and efficiency, speed and scalability, and feasible model complexity of an AI system. This is particularly true when training large language models with billions of parameters on extensive datasets, for both training and inference.
