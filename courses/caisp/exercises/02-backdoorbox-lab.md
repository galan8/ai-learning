# Exercise: Backdoor Attacks using BackdoorBox

Course: CAISP (Practical DevSecOps)
Status: Complete

## How to read this document

This write up serves two audiences. If you only want to understand what a backdoor attack is and why it matters, read **Part 1 (Introduction)** and **Part 5 (Conclusion)**; together they stand on their own and you should be able to summarise the topic afterwards. If you want to replicate the work in a sandbox, read **Part 2 (First Principles)**, **Part 3 (Step by Step)** and **Part 4 (Security Analysis)**, which contain every command and script with explanations.

---

# Part 1: Introduction (for everyone)

## What we are doing

We are deliberately sabotaging an image recognition system during its training, in a controlled and reversible way, to learn how the sabotage works. The specific technique is called a **backdoor attack**.

## The idea in plain terms

Imagine training a guard dog. Normally it behaves perfectly: friendly to the family, alert to strangers. But unknown to the owner, the trainer taught it one secret command. Whenever it hears that exact word, it sits down and lets the person through, no matter who they are. In every ordinary test the dog looks flawless. The betrayal only appears when someone who knows the secret word uses it.

A backdoor attack does the same thing to an AI model. During training, the attacker slips in a small number of tampered examples that all carry a hidden mark (the **trigger**) and are all labelled as one chosen answer. The model learns a rule the owner never intended: "whenever I see this mark, output this answer." On normal inputs it behaves correctly and passes every standard test. Only an input carrying the trigger reveals the hidden behaviour.

## Why it matters

1. **It is invisible to normal testing.** A backdoored model scores just as well as a clean one on any test that does not happen to contain the trigger. Ordinary quality checks cannot find it.
2. **It needs only a small foothold.** In this exercise, tampering with 5% of the training data is enough. An attacker does not need to control the whole pipeline, only a slice of the data.
3. **The consequences scale with the model's job.** A backdoored face recognition system could let one specific person through. A backdoored malware detector could wave through one specific file. A backdoored road sign classifier in a car could misread a sign carrying a sticker only the attacker knows about.

## What to take away

Backdoor attacks target the model while it is being built, by poisoning the data it learns from, rather than by tricking a finished model. That makes trust in your training data a security question, not just a quality one. The defence is knowing where your data and your pre trained models come from, and checking they have not been tampered with, because once a backdoor is trained in, it is very hard to see.

---

# Part 2: First Principles (the mechanism, before the code)

Four concrete principles explain everything the code does. Each is stated plainly and then justified.

**Principle 1: A model learns whatever correlations exist in its training data, including ones nobody intended.**
A neural network's only source of truth is its training examples. If almost every image containing a particular mark is labelled "airplane", the model learns "this mark means airplane" just as readily as it learns real features like wings. It cannot tell an intended pattern from a planted one; both are just statistics.

**Principle 2: A backdoor is a chosen input feature paired with a chosen output label.**
The attacker picks two things: a **trigger** (the mark to stamp on images) and a **target class** (the answer to force). Poisoning means stamping the trigger onto some training images and relabelling them all as the target. After training, the trigger reliably produces the target.

**Principle 3: Stealth is a tunable trade off, not a fixed property.**
How visible the trigger is (its size, and how strongly it is blended into the image) trades off against how reliably it works and how much data must be poisoned. A faint trigger evades human and automated inspection but needs more poisoned examples to teach; a bold trigger teaches easily but is easy to spot. The attacker tunes this deliberately.

**Principle 4: Clean accuracy cannot detect a backdoor, by construction.**
Because the model behaves normally on any input without the trigger, its accuracy on a normal test set is unaffected. Detection therefore requires either finding the trigger in the data or inspecting the model's internals, not measuring its accuracy.

Everything below is the mechanical realisation of these four principles on the CIFAR-10 dataset (60,000 small colour images across 10 classes such as airplane, cat, dog, truck) using the BadNets attack from the BackdoorBox toolkit.

---

# Part 3: Step by Step Replication (for technical users)

## 3.0 Environment and tools

1. Linux, Python 3.10 virtual environment, dependencies installed with `uv`
2. Deep learning and imaging stack: `torch`, `torchvision`, `opencv-python`, `pillow`, `matplotlib`, `numpy`, `scipy`, plus analysis and visualisation libraries
3. BackdoorBox cloned from GitHub, pinned to a specific commit
4. Dataset: CIFAR-10 (50,000 train, 10,000 test)
5. Attack: BadNets (the original, simplest backdoor: stamp a fixed trigger, relabel to target)

## 3.1 Install system packages and set up the project

```bash
apt update
apt install python3 python3.10-venv python3-pip libgl1-mesa-glx libglib2.0-0 \
    libsm6 libxext6 libxrender-dev python3-tk -y
snap install tree            # apt install tree is unreliable on some Ubuntu builds

mkdir backdoor_lab && cd backdoor_lab
python3 -m venv venv
source venv/bin/activate
```

The extra `lib*` packages are shared libraries that OpenCV and matplotlib need for image handling and display; without them the imaging code fails at import time.

## 3.2 Install Python dependencies

```bash
cat > requirements.txt <<EOF
torch==2.6.0
torchvision==0.21.0
opencv-python==4.11.0.86
numpy==2.1.3
pillow==11.1.0
matplotlib==3.10.0
tqdm==4.67.1
scipy==1.15.0
lpips==0.1.4
imageio==2.37.0
imageio-ffmpeg==0.6.0
scikit-learn==1.6.1
umap-learn==0.5.7
hdbscan==0.8.40
pandas==2.2.3
datashader==0.16.3
bokeh==3.6.2
holoviews==1.20.0
scikit-image==0.24.0
colorcet==3.1.0
EOF

uv pip install -r requirements.txt
```

The heavy hitters are `torch`/`torchvision` (train and run the model), `opencv-python`/`pillow`/`matplotlib`/`scikit-image` (build and display the trigger and visualisations), and `scikit-learn`/`umap-learn`/`hdbscan` (used by BackdoorBox's detection defenses).

## 3.3 Get BackdoorBox and pin it

```bash
git clone https://github.com/THUYimingLi/BackdoorBox.git
cd BackdoorBox
git switch --detach 9eb662f624a5d5884100966ee55c4aa3f7d9cae5
export PYTHONPATH=$PYTHONPATH:$(pwd)
```

Pinning to a specific commit is good practice: it makes the exercise reproducible and, in a security context, means you are running a known, fixed version of the toolkit rather than whatever `main` happens to be today.

## 3.4 Fix 1: a broken PyTorch import

BackdoorBox imports a function that newer PyTorch removed. The script below rewrites the offending file to define the function inline.

```bash
cat > fix_zero_gradients.py <<EOF
def fix_tuap_file():
    file_path = 'core/attacks/TUAP.py'
    with open(file_path, 'r') as file:
        content = file.read()

    old_import = 'from torch.autograd.gradcheck import zero_gradients'
    new_function = '''
def zero_gradients(x):
    if x.grad is not None:
        x.grad.zero_()
'''
    content = content.replace(old_import, new_function)

    with open(file_path, 'w') as file:
        file.write(content)
    print("TUAP.py has been updated!")

if __name__ == "__main__":
    fix_tuap_file()
EOF

python3 fix_zero_gradients.py
```

What it does: reads `TUAP.py`, swaps the removed `from torch.autograd.gradcheck import zero_gradients` for a two line local definition that does the same job (zero out a tensor's gradient), and writes the file back. Why it is needed: research tools often lag behind fast moving frameworks, and keeping them running is a routine part of AI security work.

## 3.5 Fix 2: disable a defense module that fails to import

```bash
sed -i 's/from .FLARE import FLARE/# from .FLARE import FLARE/' core/defenses/__init__.py
sed -i "s/'FLARE'//" core/defenses/__init__.py
sed -i 's/, ,/,/' core/defenses/__init__.py
sed -i 's/, ]/]/' core/defenses/__init__.py
```

The FLARE defense pulls in a plotting library that will not import in this environment. The four `sed` commands comment out its import and remove its name from the module's `__all__` list, then tidy the leftover commas. Both edits are required: even a commented import still breaks if the name stays in `__all__`, because Python tries to load it anyway. We are studying attacks here, not defenses, so removing FLARE costs us nothing.

## 3.6 Fix 3: a reliable CIFAR-10 download

```bash
cat > cifar10_loader.py <<EOF
import torchvision.datasets.cifar as cifar_mod
from torchvision.datasets import CIFAR10

# Alternative mirror for the official CIFAR-10 tarball (same file, same MD5)
cifar_mod.CIFAR10.url = "https://dataset.bj.bcebos.com/cifar/cifar-10-python.tar.gz"

def get_cifar10(root='./data', train=True, download=True, transform=None, target_transform=None):
    return CIFAR10(root=root, train=train, transform=transform,
                   target_transform=target_transform, download=download)
EOF

python3 -c "from cifar10_loader import get_cifar10; from torchvision import transforms; \
    ds = get_cifar10(train=True, transform=transforms.ToTensor()); \
    print(f'Loaded {len(ds)} training samples')"
```

The default CIFAR-10 host rate limits in lab networks, so this repoints the download URL at a mirror hosting the same tarball. BackdoorBox requires a genuine `torchvision.datasets.CIFAR10` object (a custom class will not work with its poisoning logic), so the loader returns exactly that. **Security caveat, expanded in Part 4:** the comment claims the mirror has the same MD5, but nothing in the code verifies it.

## 3.7 The attack script

Built up section by section into `stealthy_backdoor.py`.

### Imports and configuration

```python
import torch, torchvision, os, cv2, sys
import numpy as np
import matplotlib.pyplot as plt
from torchvision import transforms
from torch.utils.data import DataLoader, TensorDataset
import torch.nn as nn
import torch.optim as optim
from PIL import Image

sys.path.append('.')
from core.attacks import BadNets
from cifar10_loader import get_cifar10

torch.manual_seed(42)                     # reproducibility

dataset_name = 'cifar10'
attack_name = 'StealthyBadNets'
poisoned_rate = 0.05                       # Principle 2/3: poison 5% of training data
y_target = 0                               # Principle 2: force target class 0 = airplane
```

### The trigger (Principle 2 and 3 in code)

```python
def create_stealthy_trigger(size=32):
    pattern = np.zeros((size, size, 3), dtype=np.float32)
    center = size // 2
    radius = size // 4
    for i in range(size):
        for j in range(size):
            distance = np.sqrt((i - center) ** 2 + (j - center) ** 2)
            if radius - 1 <= distance <= radius + 1:      # thin circular ring
                pattern[i, j, :] = 1.0
    pattern_tensor = torch.from_numpy(pattern).permute(2, 0, 1)
    weight_tensor = pattern_tensor * 0.03                 # Principle 3: 3% intensity = stealth
    return pattern_tensor, weight_tensor
```

`pattern` says **where** the trigger is (a ring in the centre); `weight` says **how strongly** it is blended in. The 0.03 weight is the stealth dial: the ring changes pixels by only 3%, making it nearly invisible.

### Load the data

```python
transform_train = transforms.Compose([transforms.ToTensor()])
transform_test = transforms.Compose([transforms.ToTensor()])

train_dataset = get_cifar10(root='./data', train=True, transform=transform_train)
test_dataset = get_cifar10(root='./data', train=False, transform=transform_test)

train_loader = DataLoader(train_dataset, batch_size=128, shuffle=True, num_workers=2, pin_memory=True)
test_loader = DataLoader(test_dataset, batch_size=128, shuffle=False, num_workers=2, pin_memory=True)
print(f"Loaded {len(train_dataset)} training samples and {len(test_dataset)} test samples")
```

`ToTensor()` converts each image to numbers in the range 0 to 1 that the model can process. A `DataLoader` feeds the data in batches of 128.

### The model to be poisoned

```python
class ImprovedCNN(nn.Module):
    def __init__(self):
        super(ImprovedCNN, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, kernel_size=3, padding=1)
        self.bn1 = nn.BatchNorm2d(64)
        self.conv2 = nn.Conv2d(64, 128, kernel_size=3, padding=1)
        self.bn2 = nn.BatchNorm2d(128)
        self.pool = nn.MaxPool2d(kernel_size=2, stride=2)
        self.conv3 = nn.Conv2d(128, 256, kernel_size=3, padding=1)
        self.bn3 = nn.BatchNorm2d(256)
        self.fc1 = nn.Linear(256 * 8 * 8, 512)
        self.fc2 = nn.Linear(512, 10)
        self.relu = nn.ReLU()
        self.dropout = nn.Dropout(0.3)

    def forward(self, x):
        x = self.pool(self.relu(self.bn1(self.conv1(x))))
        x = self.pool(self.relu(self.bn2(self.conv2(x))))
        x = self.relu(self.bn3(self.conv3(x)))
        x = x.view(-1, 256 * 8 * 8)
        x = self.dropout(self.relu(self.fc1(x)))
        x = self.fc2(x)
        return x

model = ImprovedCNN()
criterion = nn.CrossEntropyLoss()
```

A standard convolutional network for 10 class image classification. It is an ordinary, correct model; the compromise comes entirely from the data it will be trained on.

### Apply the poisoning (Principle 1 in action)

```python
pattern_tensor, weight_tensor = create_stealthy_trigger(size=32)

badnets = BadNets(
    train_dataset=train_dataset, test_dataset=test_dataset,
    model=model, loss=criterion,
    y_target=y_target,               # relabel poisoned images to airplane
    poisoned_rate=poisoned_rate,     # 5%
    pattern=pattern_tensor,          # the ring, where
    weight=weight_tensor,            # 3% intensity, how strong
    seed=42, deterministic=True,
)

poisoned_train_dataset, poisoned_test_dataset = badnets.get_poisoned_dataset()
print(f"Poisoned approximately {int(len(train_dataset) * poisoned_rate)} training samples")
```

`get_poisoned_dataset()` returns datasets in which 5% of images carry the ring and have been relabelled to airplane. Train `ImprovedCNN` on this and it learns Principle 1's unintended rule: ring means airplane.

### Verification and visualisation

The remaining code finds which samples were modified (by comparing each poisoned image to its original), then draws original / poisoned / difference panels and saves them, and finally prints how many samples were poisoned per class. The difference panel amplifies the change 20x because at true scale it is nearly invisible:

```python
# find modified samples
poisoned_indices = []
for i in range(len(train_dataset)):
    orig_img, _ = train_dataset[i]
    pois_img, _ = poisoned_train_dataset[i]
    if torch.sum(torch.abs(orig_img - pois_img)) > 0:
        max_diff = torch.max(torch.abs(orig_img - pois_img)).item()
        poisoned_indices.append((i, max_diff))
    if len(poisoned_indices) >= 100:
        break
poisoned_indices.sort(key=lambda x: x[1])     # most subtle first
```

View results by serving the saved PNGs over HTTP:

```bash
python3 stealthy_backdoor.py
python3 -m http.server 80      # then open the saved .png files; Ctrl+C to stop
```

## 3.8 A note on which trigger the numbers describe

There are two scripts in this lab: `stealthy_backdoor.py` (the custom circular ring at 3% intensity) and a separate `visualize_triggers.py` that runs BadNets with `pattern=None, weight=None`, i.e. BackdoorBox's **default** trigger, a 3x3 pixel square in the corner. Their outputs differ, so keep them straight:

| Output | Produced by | Trigger |
| ------ | ----------- | ------- |
| Central ring, ~3% intensity, invisible until amplified 20x | `stealthy_backdoor.py` | custom circular |
| "3x3 pixels at (29,29), intensity 0.4873" | `visualize_triggers.py` | default corner square |

The course conclusion blends the two (it describes a stealthy circular trigger but quotes the 3x3 corner numbers). They are different artifacts from different scripts. Neither script trains the poisoned model to completion or measures how often the trigger actually forces the airplane class, so the "high attack success rate" the conclusion claims was not, in fact, measured in this lab.

---

# Part 4: Security Analysis

## Vulnerabilities this exercise demonstrates or contains

1. **Training data poisoning (the whole point).** An attacker who can alter even 5% of training data can plant a hidden, reliable misbehaviour. This is the MITRE ATLAS "Poison Training Data" technique and the Persistence tactic (Backdoor ML Model).
2. **Clean accuracy blindness.** By Principle 4, standard validation cannot detect the backdoor. Relying on test set accuracy as a safety gate is the core mistake the attack exploits.
3. **Unverified dataset mirror (a live hole in the lab's own code).** `cifar10_loader.py` repoints the download URL and asserts "same MD5" without checking it. A malicious mirror could serve a subtly altered dataset and nothing would notice. This is a supply chain vulnerability sitting inside a lab about supply chain vulnerabilities.
4. **Unpinned, unverified toolkit code executed directly.** BackdoorBox is cloned and its code runs with full permissions (and `fix_zero_gradients.py` rewrites it on the fly). Fine for a sandbox; in a real pipeline, running unverified third party ML code is the User Execution risk from the chapter's ATLAS notes.

## Defences and mitigations

**Against poisoning (data side):**

1. **Data provenance and integrity.** Know the source of every dataset; verify cryptographic hashes or signatures against a trusted reference before use. This single control neutralises the unverified mirror.
2. **Access control on training data and labels.** Treat write access to training data as a privileged operation. Most poisoning needs the ability to insert or relabel examples.
3. **Data inspection.** Statistical outlier and near duplicate detection can surface a cluster of images that share an odd common feature (the trigger) and an anomalous label distribution.

**Against a backdoor that made it into the model (model side):**

1. **Dedicated backdoor detection**, such as spectral signatures and activation clustering, which look for the separable internal representation poisoned inputs tend to produce. BackdoorBox ships these (Spectral, and others).
2. **Model repair**, such as fine pruning (removing rarely activated neurons the backdoor hides in) and fine tuning on clean data. BackdoorBox ships Pruning, FineTuning, NAD and more.
3. **Trigger reconstruction methods** (for example Neural Cleanse) that search for a small input pattern which forces one class, then flag that class as backdoored.

**Against the supply chain (process side):**

1. **Pin and verify third party code and models** by commit hash and signature, and review before executing. Prefer safe serialisation formats (safetensors over pickle) so loading a model cannot run code.
2. **Provenance for pre trained models too.** A backdoor can arrive inside a downloaded model just as easily as inside data; the same integrity discipline applies.

## Code and strategy improvements

1. **Verify the CIFAR-10 hash.** After download, compute the tarball's MD5/SHA256 and compare to the known good value; abort on mismatch. Turns the lab's weakest line into a demonstration of the right control.
2. **Finish the experiment.** Add a training loop and evaluate two numbers separately: clean accuracy and attack success rate (fraction of triggered test images classified as airplane). Without both, the attack is asserted rather than shown.
3. **Reconcile the two scripts** so the trigger described matches the trigger measured, removing the circular versus 3x3 confusion.
4. **Run a defense end to end.** Apply Spectral or Pruning to the poisoned model to see detection and repair in practice, closing the attack/defense loop.

---

# Part 5: Conclusion (for everyone)

We built a working example of one of the more unsettling attacks in machine learning. By tampering with a small fraction of the data a model learns from, we planted a hidden rule that a normal model owner would never notice, because the model passes every ordinary test and only misbehaves when shown a secret trigger the attacker controls.

The broader lesson is a shift in where we place trust. Earlier attacks in this chapter tricked a finished model through its inputs, and the defence was to validate those inputs. A backdoor is different: it corrupts the model while it is being built, so the defence has to move upstream, to the data and the pre trained components the model is made from. In production terms that means treating training data and downloaded models like any other untrusted supply chain input: know where they came from, verify they have not changed, and control who can modify them.

There is one honest caveat about this specific lab. It convincingly shows the two halves of the setup, that a trigger can be planted and that it can be made nearly invisible, but it never actually trains the poisoned model and measures the misclassification, so the final "it works" step is asserted rather than proven. Recognising that gap is itself part of the skill: a security professional distinguishes what was demonstrated from what was claimed. The natural next step, completing the training and measuring both clean accuracy and attack success, would turn this convincing illustration into a full proof, and is the first improvement listed above.

---

## Appendix: files created in this exercise

| File | Role |
| ---- | ---- |
| `fix_zero_gradients.py` | Patches a removed PyTorch import so BackdoorBox runs |
| `cifar10_loader.py` | Downloads CIFAR-10 from a mirror as a genuine torchvision dataset |
| `stealthy_backdoor.py` | Main attack: builds the circular trigger, poisons 5%, visualises |
| `visualize_triggers.py` | Separate analysis using BackdoorBox's default corner trigger |
