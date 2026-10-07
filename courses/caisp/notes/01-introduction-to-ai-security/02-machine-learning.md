# Machine Learning

ML is essentially teaching computers from experience: systems analyse patterns in data and learn to make decisions through exposure to examples rather than explicit programming.

## AI vs ML

1. AI is the broader term: it covers all attempts to make machines intelligent, designing systems that mimic human intelligence, thinking and decision making. It has a wide scope of applications.
2. ML is a subset of AI: a specific approach to achieving intelligence by learning from large amounts of data using algorithms. It improves performance through experience and has a narrower scope.
3. In practice, most modern AI systems use some form of ML to gain their intelligence.

## Key concepts: features and labels

1. **Features** are the input variables or characteristics that describe each individual data point (instance) in a dataset. They are the pieces of information the model can analyse to understand patterns, make decisions and perform predictions.
2. **Labels** are the output variables the model aims to predict. They represent the known answers or ground truth values associated with each instance. Labels are essential in supervised learning, where the model learns from data with known answers to make accurate predictions on new, unseen data.
3. Example 1, predicting house prices: the attributes we use (size, location, age, number of bedrooms) are the features; the price we want to predict is the label.
4. Example 2, spam detection: features of an email might include containing words like "free", being sent at 3 am, or coming from an unknown sender; the label is whether the email is spam or not.
5. Types of features: numerical (age, price, height), categorical (type of housing, city name), binary (true or false states), textual (product reviews, social media posts) and derived features calculated from others (price per square foot).

## Supervised Learning

A type of ML that uses labelled datasets to train algorithms to classify data or predict outcomes accurately.

**What is labelled data?** Raw data with added tags (labels) that give it context or meaning. The labels represent the correct answers and act as the targets the model learns to predict. Example: images tagged as "cat" or "dog", or emails tagged as "spam" or "not spam".

**How supervised learning works:**

1. Training phase: the algorithm is given training data (inputs paired with their correct output labels)
2. Model building: the algorithm learns the relationship between inputs and outputs and adjusts itself to minimise errors
3. Prediction phase: the trained model is applied to new, unseen data to predict outcomes and identify patterns

**Main types:**

1. **Regression**: predicts a continuous numeric value by detecting the relationship between two or more variables. Examples: house prices, temperature, sales forecasts. Common algorithms include linear regression and logistic regression (logistic regression is technically used for classification even though it carries the name "regression").
2. **Classification**: predicts a discrete category or class label. Examples: spam or not spam, fraud or legitimate. Common algorithms include decision trees (a tree of yes/no rules leading to an output label), random forests (many decision trees combined) and support vector machines (SVM). Random forests and SVMs can be used for both regression and classification.

## Unsupervised Learning

Uses data that is not labelled: the algorithm learns patterns and structure from the data on its own, without predefined categories or correct answers.

Because the data has no tags or identifiers describing its characteristics, discovering what the data represents is more challenging than in supervised learning, but it does not require the costly effort of labelling.

**Common techniques:**

1. **Clustering**: grouping similar items together. Example: Netflix grouping movies based on viewing behaviour.
2. **Association**: finding rules that link items together. Example: "customers who bought this also bought that".
3. **Dimensionality reduction**: reducing the number of features in the data while keeping the important information. Analogy: compressing a high resolution image while keeping the quality, or describing a movie to a friend in five minutes by covering only the key parts.
4. **Anomaly detection**: finding unusual items that do not fit the normal pattern. Example: unusual financial transactions that do not follow a regular pattern (fraud detection, intrusion detection).
5. **Autoencoders**: neural networks that learn a compressed representation of the data and reconstruct it. Used for reducing noise in data, dimensionality reduction and anomaly detection.

Applying these techniques to unlabelled data allows the system to extract useful intelligence and structure without human labelling.

## Reinforcement Learning (RL)

RL is the study of decision making: an agent learns how to make the best sequence of decisions by receiving rewards for good actions and penalties for bad ones. Unlike supervised learning there are no labelled answers, and unlike unsupervised learning there is a clear goal (maximise cumulative reward).

**Core components:**

1. Agent (learner): the entity that makes decisions and takes actions
2. Environment: the world the agent interacts with, which changes state in response to actions
3. Policy: the strategy that guides which action the agent takes in a given state
4. Reward signal: feedback from the environment after each action, positive for good outcomes and negative (penalty) for bad ones

**Two families of RL algorithms:**

1. **Model free RL**: learns purely through trial and error, without understanding the rules of the environment. Analogy: learning to ride a mountain bike by trying, falling and adjusting rather than studying physics. The loop is simply agent, action, result.
2. **Model based RL**: the agent builds an internal model of how its actions affect the environment and uses it to plan ahead. Analogy: a chess player thinking several moves ahead before acting.

**Types of model free RL:**

1. **Value based**: the agent learns the value of each action in each state and picks the highest. Example: while navigating, if turning right is worth 5 points and turning left is worth 3, the agent chooses right.
2. **Policy based**: the agent learns the policy (rules for acting) directly, without first estimating values. Example: in a video game, dodge if the enemy is close, attack if the enemy is far.

Both are learned through rules and trial and error without any explicit knowledge of the environment's dynamics.

**Benefits:**

1. No upfront model of the environment is required
2. No separate training dataset is needed: training data is generated through the agent's direct interaction with the environment
3. Adaptive: the agent keeps learning and can respond to changes in the environment

**Challenges:**

1. Requires large amounts of experience to learn effectively, and the rate of data collection is limited by the dynamics of the environment
2. Finding optimal policies is difficult because the agent must balance exploring new actions against exploiting known rewarding ones
3. Delayed rewards: in many environments the outcome is unknown until many actions have been taken, which makes credit assignment hard

**Applications:** game playing (chess, Go, video games), robotics, autonomous navigation, resource scheduling and recommendation systems. RL is also used in AI safety through Reinforcement Learning from Human Feedback (RLHF), which trains language models to prefer helpful and safe responses.

**Security relevance:** because the agent learns from environmental feedback, an attacker who can manipulate the environment or the reward signal (reward hacking, poisoning the feedback loop) can steer the agent toward harmful behaviour.
