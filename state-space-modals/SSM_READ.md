#  Assignment: State-Space Models and S4 on sCIFAR-10

##  Introduction

State-space models (SSMs) are gaining attention as an alternative to Transformers for long-sequence modeling. While SSMs have existed for decades in control theory, recent developments like **S4** ("Structured State Space for Sequence Modeling") offer a practical way to scale these models efficiently for machine learning tasks.

This assignment focuses on:

- Understanding S4 from the paper _"Efficiently Modeling Long Sequences with Structured State Spaces"_
- Implementing a basic S4 layer using PyTorch
- Training it on the sCIFAR-10 dataset from the Long Range Arena (LRA) benchmark

---

##  Dataset Setup

We use the CIFAR-10 dataset and convert each image into a 1D grayscale sequence.

```python
import torch 
import torchvision
import torchvision.transforms as transforms
from torch.utils.data import Dataset, DataLoader

# Define transformations
transform = transforms.Compose([
    transforms.Grayscale(),  # Convert RGB to grayscale (1 channel)
    transforms.ToTensor(),  # Convert to tensor
    transforms.Normalize((0.5,), (0.5,))  # Normalize to mean 0, std 0.5
])

# Load CIFAR-10 dataset
trainset = torchvision.datasets.CIFAR10(root='./data', train=True, download=True, transform=transform)
testset = torchvision.datasets.CIFAR10(root='./data', train=False, download=True, transform=transform)

# Convert each image from (1, 32, 32) → (1024, 1)
class sCIFAR10Dataset(Dataset):
    def __init__(self, dataset):
        self.dataset = dataset

    def __len__(self):
        return len(self.dataset)

    def __getitem__(self, idx):
        img, label = self.dataset[idx]
        img = img.view(-1, 1)  # (1024, 1)
        return img, label

train_dataset = sCIFAR10Dataset(trainset)
test_dataset = sCIFAR10Dataset(testset)

trainloader = DataLoader(train_dataset, batch_size=1, shuffle=True)
testloader = DataLoader(test_dataset, batch_size=1, shuffle=False)
```

---

## 🧮 S4 Layer: A Simplified Version

We implement a toy version of the S4 layer using PyTorch.

```python
import torch.nn as nn

class SimpleS4Layer(nn.Module):
    def __init__(self, input_dim, state_dim):
        super().__init__()
        self.A = nn.Parameter(torch.randn(state_dim, state_dim) * 0.1)
        self.B = nn.Parameter(torch.randn(state_dim, input_dim) * 0.1)
        self.C = nn.Parameter(torch.randn(1, state_dim) * 0.1)

    def forward(self, x):
        batch_size, seq_len, input_dim = x.squeeze(-1).shape
        h = torch.zeros(batch_size, self.A.shape[0], device=x.device)
        outputs = []

        for t in range(seq_len):
            xt = x[:, t]
            h = torch.tanh(torch.matmul(h, self.A.t()) + torch.matmul(xt, self.B.t()))
            y = torch.matmul(h, self.C.t())
            outputs.append(y)

        outputs = torch.cat(outputs, dim=1)  # (batch_size, seq_len)
        return outputs
```

---

##  S4 Classifier

We wrap the S4 layer with a classifier for CIFAR-10 (10 classes).

```python
class S4Classifier(nn.Module):
    def __init__(self, input_dim=1, state_dim=64, num_classes=10):
        super().__init__()
        self.s4 = SimpleS4Layer(input_dim, state_dim)
        self.fc = nn.Linear(1024, num_classes)  # Pooling + classification

    def forward(self, x):
        x = x.unsqueeze(-1)  # Ensure shape: (batch_size, seq_len, input_dim)
        x = self.s4(x)
        x = torch.mean(x, dim=1)  # Temporal average pooling
        return self.fc(x)
```

---

## Training Loop

```python
import torch.optim as optim

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = S4Classifier().to(device)
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001)

num_epochs = 5
for epoch in range(num_epochs):
    model.train()
    for images, labels in trainloader:
        images, labels = images.to(device), labels.to(device)

        optimizer.zero_grad()
        outputs = model(images)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()

    print(f"Epoch {epoch+1}/{num_epochs}, Loss: {loss.item():.4f}")
```

---

##  Known Issues

A `RuntimeError` occurred due to a shape mismatch in `self.fc(x)` during training.  
This might be because the output of the `SimpleS4Layer` is of shape `(batch_size, seq_len)`, but the fully connected layer expects `(batch_size, 1024)`.

---

