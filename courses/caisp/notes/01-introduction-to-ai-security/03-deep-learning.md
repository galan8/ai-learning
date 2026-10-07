# Deep Learning

Deep learning is a subset of machine learning, named for its levels of depth: it uses multiple layers of learning, where each layer learns increasingly complex features of the data. It is a branch of ML that uses artificial neural networks to teach computers to process data in a way inspired by the human brain.

**The hierarchy:**

1. AI: the set of technologies that enable computers to perform advanced functions, learn from experience and exhibit human like reasoning capabilities
2. ML: a subset of AI that allows machines to learn from data and improve their performance without being explicitly programmed
3. Deep learning: a subset of ML that uses multi layered artificial neural networks

## Neural Networks

Neural networks are algorithms inspired by the structure of the human brain. They consist of layers of interconnected nodes (neurons), where each neuron processes its input and passes the result to the neurons in the next layer.

**Structure:**

1. Input layer: takes in the raw data
2. Hidden layers: one or more layers between input and output that perform the feature extraction and transformation. The more hidden layers, the "deeper" the network.
3. Output layer: provides the final decision or prediction

**How learning happens:** the model learns during training by adjusting the weights of the connections between neurons. It repeatedly makes predictions, measures the error against the correct answer, and adjusts the weights to improve accuracy (this adjustment process is known as backpropagation).

**Advantages over traditional ML:**

1. Automatic feature learning: the network discovers the relevant features by itself, no manual feature engineering required
2. Scales with data: deep learning keeps improving as the volume of data grows, becoming more powerful as it processes more data and patterns
3. Non linear relationships: layered learning allows the network to capture complex, non linear patterns in the data
4. Improved generalisation: the model can make accurate predictions even on unseen examples

## Convolutional Neural Networks (CNNs)

CNNs are a specialised type of neural network designed for processing structured grid like data, such as images. They are particularly useful in computer vision because they can automatically detect visual features (edges, shapes, textures) directly from pixels, and they detect those features regardless of their position in the image (translation invariance).

**Key components:**

1. Convolutional layers: apply filters (kernels) to the input data to extract features such as edges, colours and patterns
2. Pooling layers: reduce the size and complexity of the data, which lowers computation and helps the network generalise
3. Fully connected layers: take the extracted features and produce the final prediction or classification

**Connectivity:** in convolutional layers each neuron is connected only to a small region of the previous layer (its receptive field), rather than to every neuron as in a standard fully connected network. This makes CNNs far more efficient for images.

**Common applications:** image and video recognition, object detection, facial recognition, medical imaging, and even some NLP tasks.
