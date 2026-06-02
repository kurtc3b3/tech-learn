
## What & When

**PyTorch** is the primary **deep learning** framework in this vault — dynamic computation graphs, GPU acceleration, and a Pythonic API for neural networks, training loops, and deployment (TorchScript, ONNX, `torch.compile`). Use it when tabular boosting ([[ML — XGBoost]], [[ML — LightGBM]]) or [[ML — scikit-learn]] are not enough.

Use PyTorch when:

- Modeling **images, audio, text embeddings**, or custom architectures
- You need **fine-grained control** over training loops and losses
- Research prototypes move toward production via TorchScript/ONNX
- Integrating neural components into larger systems (hybrid with [[AI]] embeddings)

Overview context: [[Machine Learning]]. Log experiments with [[ML — MLflow]] (`mlflow.pytorch`); tune with [[ML — Optuna]] (learning rate, layers, dropout).

```bash
pip install torch torchvision torchaudio
# CPU-only or CUDA wheel — match https://pytorch.org for your platform
```

---

## PyTorch vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Tabular classical ML | [[ML — scikit-learn]] | Faster iteration |
| Tabular SOTA (trees) | [[ML — XGBoost]], [[ML — LightGBM]] | Often beats small MLPs |
| Time series baseline | [[ML — Prophet]] | No nn required |
| LLM orchestration | [[AI]] — LangChain, etc. | Higher-level than raw torch |
| Tracking | [[ML — MLflow]] | Metrics + `log_model` |
| HPO | [[ML — Optuna]] | `trial` + training loop |

---

## Tensors & Device

```python
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

x = torch.randn(32, 10, device=device)
y = torch.randint(0, 2, (32,), device=device)

# From numpy
import numpy as np
arr = np.random.randn(16, 5).astype("float32")
t = torch.from_numpy(arr).to(device)
```

---

## Minimal Training Loop (Classification)

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader, TensorDataset

# Synthetic tabular example
X = torch.randn(1000, 20)
y = (X[:, 0] + X[:, 1] > 0).long()
dataset = TensorDataset(X, y)
loader = DataLoader(dataset, batch_size=64, shuffle=True)

class MLP(nn.Module):
    def __init__(self, in_dim=20, hidden=64, n_classes=2):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(in_dim, hidden),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(hidden, n_classes),
        )

    def forward(self, x):
        return self.net(x)

model = MLP().to(device)
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-4)

def train_epoch(model, loader, optimizer, criterion):
    model.train()
    total_loss = 0.0
    for xb, yb in loader:
        xb, yb = xb.to(device), yb.to(device)
        optimizer.zero_grad()
        logits = model(xb)
        loss = criterion(logits, yb)
        loss.backward()
        optimizer.step()
        total_loss += loss.item() * xb.size(0)
    return total_loss / len(loader.dataset)

for epoch in range(10):
    loss = train_epoch(model, loader, optimizer, criterion)
    print(f"epoch {epoch}: loss={loss:.4f}")
```

---

## Validation & Metrics

```python
@torch.no_grad()
def evaluate(model, loader):
    model.eval()
    correct, total = 0, 0
    for xb, yb in loader:
        xb, yb = xb.to(device), yb.to(device)
        preds = model(xb).argmax(dim=1)
        correct += (preds == yb).sum().item()
        total += yb.size(0)
    return correct / total

val_loader = DataLoader(TensorDataset(X[:200], y[:200]), batch_size=64)
print("acc:", evaluate(model, val_loader))
```

---

## `Dataset` / `DataLoader` (Real Data)

```python
from torch.utils.data import Dataset

class TabularDataset(Dataset):
    def __init__(self, features, labels):
        self.x = torch.as_tensor(features, dtype=torch.float32)
        self.y = torch.as_tensor(labels, dtype=torch.long)

    def __len__(self):
        return len(self.y)

    def __getitem__(self, idx):
        return self.x[idx], self.y[idx]

train_loader = DataLoader(TabularDataset(X_train, y_train), batch_size=128, shuffle=True, num_workers=2, pin_memory=True)
```

Use [[ML — pandas]] for feature prep; convert to numpy/torch before training.

---

## Learning Rate Scheduler

```python
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=10)

for epoch in range(10):
    train_epoch(model, loader, optimizer, criterion)
    scheduler.step()
```

---

## Save & Load

```python
torch.save({
    "model_state": model.state_dict(),
    "optimizer_state": optimizer.state_dict(),
    "epoch": 10,
}, "checkpoints/mlp.pt")

ckpt = torch.load("checkpoints/mlp.pt", map_location=device, weights_only=True)
model.load_state_dict(ckpt["model_state"])
```

Prefer [[ML — MLflow]] for versioned artifacts in team workflows.

---

## MLflow Integration

```python
import mlflow
import mlflow.pytorch

with mlflow.start_run(run_name="mlp-tabular"):
    for epoch in range(10):
        loss = train_epoch(model, loader, optimizer, criterion)
        acc = evaluate(model, val_loader)
        mlflow.log_metrics({"train_loss": loss, "val_acc": acc}, step=epoch)
    mlflow.pytorch.log_model(model, artifact_path="model")
```

---

## Optuna Hyperparameter Search

```python
import optuna

def objective(trial):
    hidden = trial.suggest_int("hidden", 32, 256, step=32)
    lr = trial.suggest_float("lr", 1e-4, 1e-2, log=True)
    m = MLP(hidden=hidden).to(device)
    opt = torch.optim.AdamW(m.parameters(), lr=lr)
    for _ in range(5):
        train_epoch(m, loader, opt, criterion)
    return evaluate(m, val_loader)

study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials=30)
```

See [[ML — Optuna]] for pruning and distributed studies.

---

## Inference Mode

```python
model.eval()
with torch.inference_mode():
    logits = model(x_batch.to(device))
    probs = torch.softmax(logits, dim=1)
```

---

## `torch.compile` (PyTorch 2+)

```python
model = torch.compile(model)  # optional speedup after warm-up
```

---

## Pitfalls

| Issue | Fix |
| --- | --- |
| Train/eval modes | `model.train()` vs `model.eval()`; dropout/batchnorm |
| GPU OOM | Smaller batch, gradient accumulation, mixed precision |
| Non-determinism | `torch.manual_seed`, `cudnn.deterministic` (slower) |
| Data leakage | Fit normalization on train only |
| Tabular overkill | Try [[ML — LightGBM]] first |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Device | `device = torch.device("cuda" if torch.cuda.is_available() else "cpu")` |
| Module | `class M(nn.Module): ...` |
| Loss | `nn.CrossEntropyLoss()` / `nn.MSELoss()` |
| Optimizer | `torch.optim.AdamW(model.parameters(), lr=1e-3)` |
| Train step | `zero_grad → forward → loss → backward → step` |
| DataLoader | `DataLoader(dataset, batch_size=64, shuffle=True)` |
| Save | `torch.save(model.state_dict(), path)` |
| MLflow | `mlflow.pytorch.log_model(model, "model")` |
| Inference | `model.eval(); torch.inference_mode():` |

---

## Related Notes

- [[Machine Learning]] — deep learning in lifecycle
- [[ML — scikit-learn]] — baselines before nn
- [[AI]] — LLM apps (often separate from training loops here)
- [[ML — MLflow]], [[ML — Optuna]]
- [[API - FastAPI]] — serve exported models

---

## Tags

#python #pytorch #deep-learning #neural-networks #gpu #training-loop #machine-learning #mlflow #optuna
