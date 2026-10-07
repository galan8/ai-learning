# Exercise: Building a Fine-tuned Model (Image Classifier)

Course: CAISP (Practical DevSecOps)
Status: Complete (all five steps)

## 1. Objective

Understand the fine tuning process end to end by taking a pre trained ResNet 18 model (trained on ImageNet) and adapting it to classify five custom classes: baboon, crab, crocodile, dog and snake. The goal is the process, not the code detail: load datasets, fine tune, evaluate, save the model, then use it for inference.

This is also the first lab dealing with a vision model rather than a language model, and the first that produces a model artifact of its own.

## 2. Environment and tools

1. Lab environment (Linux) with GPU, Python 3.10
2. `torch==2.3.0`, `torchvision==0.18.0`, `Pillow==10.0.0`, plus the CUDA libraries pulled in by torch
3. Code and dataset cloned from the course GitLab repository (`caisp-image-classifier`)
4. Pre trained model: ResNet 18 with ImageNet weights (`IMAGENET1K_V1`), downloaded from PyTorch's model hub

## 3. Key concepts

| Concept | Meaning in ML and PyTorch |
| ------- | ------------------------- |
| Epoch | One full pass through the training dataset |
| Optimizer | The algorithm that adjusts weights, here SGD with momentum |
| Loss function | Quantifies how wrong a prediction was, here cross entropy |
| Learning rate scheduler | Reduces the learning rate as training progresses so later updates are finer |
| Data augmentation | Random crops and flips introduce variation so the model generalises rather than memorising |
| Transfer learning | Reusing a model trained on one task as the starting point for another |

**Why ResNet 18**: an 18 layer convolutional neural network trained on ImageNet (1,000 classes). It already knows how to recognise basic visual features such as edges, textures and shapes, which is exactly what makes it a good starting point.

**Why fine tune when ResNet 18 already knows "dog"**: the ImageNet classes may not match the specific categories you need, your images differ in style, size and lighting from ImageNet's, and fine tuning adapts the learned features to your own data and labels.

## 4. Dataset structure

The training code expects a fixed folder layout, where the folder names become the class labels:

```
data:image-sets/
├── train/     16 images per class
│   ├── baboon/  crab/  crocodile/  dog/  snake/
└── val/       4 images per class
    └── baboon/  crab/  crocodile/  dog/  snake/
```

The usual split is roughly 80% training and 20% validation: the training set adjusts the weights, the validation set measures how well the model performs on images it did not learn from.

## 5. How the code works

### 5.1 Data transformations

```python
train_val_data = {
    "train": transforms.Compose([
        transforms.RandomResizedCrop(224),
        transforms.RandomHorizontalFlip(),
        transforms.ToTensor(),
        transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
    ]),
    "val": transforms.Compose([
        transforms.Resize(256),
        transforms.CenterCrop(224),
        transforms.ToTensor(),
        transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
    ]),
}
```

Training images get random crops and horizontal flips (augmentation, so the model sees varied versions of the same image); validation images get a deterministic resize and centre crop, so evaluation is consistent. Both normalise using ImageNet's channel means and standard deviations, because the pre trained weights expect inputs on that scale. All images become 224 by 224, the input size ResNet expects.

### 5.2 Swapping the final layer

```python
model_fine_tuned = models.resnet18(weights="IMAGENET1K_V1")
number_of_filters = model_fine_tuned.fc.in_features
model_fine_tuned.fc = nn.Linear(number_of_filters, len(class_names))
```

This is the heart of the technique: the pre trained network is kept intact except for its final fully connected layer, which is replaced with a new one sized to five classes instead of 1,000. The convolutional layers that recognise edges and textures are reused; only the classification head is new. Training then updates all layers, with the new head learning fastest.

### 5.3 Training setup and loop

```python
criterion = nn.CrossEntropyLoss()
optimizer_fine_tune = optim.SGD(model_fine_tuned.parameters(), lr=0.001, momentum=0.9)
exp_learning_rate_scheduler = lr_scheduler.StepLR(optimizer_fine_tune, step_size=7, gamma=0.1)
```

A low learning rate (0.001) is deliberate in fine tuning: large updates would destroy the useful features already learned. The scheduler cuts the rate by a factor of ten every seven epochs.

Each epoch runs two phases. In `train` the model computes predictions, measures loss, backpropagates and updates weights. In `val` gradients are disabled and the model is only evaluated. Whenever validation accuracy improves, the weights are checkpointed to a temporary directory, and at the end the best checkpoint is reloaded, so the returned model is the best performing one rather than simply the last one.

### 5.4 Inference

```python
def inference(model, image_path):
    model.eval()
    img = Image.open(image_path).convert("RGB")
    img = train_val_data["val"](img)
    img = img.unsqueeze(0)
    img = img.to(device)
    with torch.no_grad():
        outputs = model(img)
        _, preds = torch.max(outputs, 1)
        return class_names[preds]
```

The input image goes through the same validation transforms used in training, `unsqueeze(0)` adds the batch dimension the model expects, and `torch.max` picks the highest scoring class. Note that only the winning label is returned, with no confidence score.

### 5.5 Save and load

The entry point trains and saves the model on first run, then loads the saved `.pt` file on subsequent runs so training happens only once. Inference runs when an image path is passed as an argument.

## 6. Results

Training completed in 28 seconds over 25 epochs.

| Metric | Start (epoch 0) | End (epoch 24) |
| ------ | --------------- | -------------- |
| Train loss | 1.4800 | 0.2453 |
| Train accuracy | 0.3500 | 0.9125 |
| Val loss | 0.5009 | 0.0125 |
| Val accuracy | 0.9000 | 1.0000 |

Best validation accuracy: 100%. All five sample images were classified correctly, as were three crab photos downloaded from the internet.

**A caution the course does not raise**: validation accuracy hit 100% by epoch 1 and stayed there, while training accuracy remained around 91%. That pattern is a sign the validation set is far too small to measure anything, not evidence of an excellent model. With four images per class, twenty images in total, a single correct guess is worth five percentage points and 100% is easy to reach by luck. Training accuracy sitting below validation accuracy is also backwards from the usual pattern, explained here by augmentation making the training images harder than the validation ones. The honest conclusion is that the fine tuning process worked; the accuracy figure itself means very little on twenty images.

## 7. Lessons learned (security focus)

1. **`torch.save(model, path)` serialises the whole model with pickle, and `torch.load` deserialises it by executing pickle opcodes.** That is arbitrary code execution on load if the file came from someone else. This lab produces exactly the artifact type behind the MITRE ATLAS "User Execution" technique in the Chapter 2 notes. The safer pattern is saving `model.state_dict()` (weights only) and loading it into a model you construct yourself, or using safetensors.
2. **The code's `safe_globals` block shows the ecosystem reacting to this.** PyTorch 2.4 and later default to `weights_only=True` in `torch.load`, so the lab code has to explicitly allowlist the classes it expects on newer versions while taking the unrestricted path on 2.3. Reading that branch is a good illustration of a supply chain mitigation showing up in real code.
3. **Fine tuning is a poisoning surface.** The class labels come from folder names and the training data is whatever sits in those folders. Anyone who can add or relabel images in the training set can shift the model's behaviour, including installing a targeted backdoor that only triggers on a chosen pattern. Training data provenance and integrity matter as much as code integrity.
4. **The model has no "unknown" class.** `torch.max` always returns the highest scoring class, so feeding it a photo of a car returns one of the five animals with no indication of doubt. Any classifier used in a security decision needs a confidence threshold and a rejection path; this one silently produces a confident wrong answer for out of distribution input.
5. **Inherited weights carry inherited weaknesses.** The base model came from a public checkpoint download. Whatever biases, spurious correlations or adversarial vulnerabilities exist in those pre trained weights are inherited by the fine tuned model, and fine tuning on twenty images will not remove them.
6. **Small validation sets create false confidence.** A 100% figure in a report is persuasive to non technical stakeholders. Knowing to ask "on how many samples?" is a practical audit skill.

## 8. Ideas to take forward

1. Experiment idea: modify the script to save and load `state_dict` instead of the full model, and compare the file contents and load behaviour
2. Experiment idea: feed the classifier an out of distribution image (a car, a chair) and print the raw output scores with softmax probabilities to see how confidently it produces a wrong answer
3. Experiment idea: add a confidence threshold so low scoring predictions return "unknown"
4. Concept file candidate: `concepts/fine-tuning-and-transfer-learning.md`, connecting this vision example back to LLM fine tuning from the Chapter 2 notes
5. Concept file candidate: `concepts/model-serialisation-risks.md` on pickle, safetensors and `weights_only`
