# Modal Deployment Patterns

## Modal app template

Default to a shared Modal Volume with a per-session, per-experiment layout:

```text
/labrat/<session-slug>/
  00-baseline/
    results.json
    metadata.json
    checkpoints/
  01-variant/
    results.json
    metadata.json
    checkpoints/
```

The remote `results.json` is the primary completion artifact. The local experiment directory's `results.json` is just a cache written after harvest.

**Important:** Modal (v1.3+) does NOT auto-mount sibling files — only the entrypoint `.py` is included. Use `image.add_local_file()` to include `train.py` and any other files the function needs. `modal.Mount` is removed.

```python
import json
import os
from pathlib import Path

import modal

local_dir = Path(__file__).parent
app = modal.App("research-EXPERIMENT-NAME")
SESSION_SLUG = "my-research-session"
EXPERIMENT_NAME = "00-baseline"
REMOTE_DIR = f"/labrat/{SESSION_SLUG}/{EXPERIMENT_NAME}"

image = (
    modal.Image.debian_slim(python_version="3.12")
    .pip_install("torch", "torchvision", "numpy")
    .add_local_file(str(local_dir / "train.py"), "/root/train.py")
)

vol = modal.Volume.from_name("research-vol", create_if_missing=True)

@app.function(
    image=image,
    gpu="T4",  # cheapest that works; upgrade only if needed
    timeout=1800,
    volumes={"/vol": vol},
)
def train(config: dict) -> dict:
    import sys
    sys.path.insert(0, "/root")
    from train import train_and_eval

    os.makedirs(f"{REMOTE_DIR}/checkpoints", exist_ok=True)
    result = train_and_eval(config)

    metadata = {
        "experiment_name": EXPERIMENT_NAME,
        "remote_dir": REMOTE_DIR,
    }

    with open(f"{REMOTE_DIR}/results.json", "w") as f:
        json.dump(result, f, indent=2)

    with open(f"{REMOTE_DIR}/metadata.json", "w") as f:
        json.dump(metadata, f, indent=2)

    vol.commit()
    return result

@app.local_entrypoint()
def main():
    config = json.loads(open("config.json").read())
    result = train.remote(config)
    # Local cache for interactive runs; unattended reconciliation should prefer the volume copy.
    with open("results.json", "w") as f:
        json.dump(result, f, indent=2)
    print(json.dumps(result, indent=2))
```

Run with: `cd .research/experiments/NN-name && modal run modal_app.py`

Record the remote artifact path in `.research/state.json` when you launch:

```json
{
  "artifact_store": {
    "kind": "modal_volume",
    "volume_name": "research-vol",
    "base_path": "/labrat/my-research-session"
  },
  "current_experiment": {
    "name": "00-baseline",
    "remote_dir": "/labrat/my-research-session/00-baseline",
    "remote_results_path": "/labrat/my-research-session/00-baseline/results.json"
  }
}
```

## Local validation before deploying

Before spending GPU credits:
1. Check syntax: `python -c "import ast; ast.parse(open('train.py').read())"`
2. Check imports resolve (in the Modal image context — just review mentally)
3. If feasible, dry-run on CPU with 1 batch: `python train.py --dry-run` (if you built that flag in)

## Checkpointing for longer runs

For runs that might take >10 min, save checkpoints to the Modal Volume:

```python
ckpt_dir = f"{REMOTE_DIR}/checkpoints"
os.makedirs(ckpt_dir, exist_ok=True)

# Save periodically
if epoch % save_every == 0:
    torch.save({"model": model.state_dict(), "epoch": epoch}, f"{ckpt_dir}/latest.pt")

# Resume on restart
ckpt_path = f"{ckpt_dir}/latest.pt"
if os.path.exists(ckpt_path):
    ckpt = torch.load(ckpt_path, weights_only=True)
    model.load_state_dict(ckpt["model"])
    start_epoch = ckpt["epoch"] + 1
```

After writing a checkpoint that you care about across restarts, call `vol.commit()`.

## Common failure patterns

- **OOM**: Reduce batch size first. Then try gradient accumulation. Then try a smaller model. Then try a cheaper GPU with less VRAM only as last resort (to avoid changing cost estimates).
- **Training divergence**: Check learning rate (try 10x lower), check data loading, check loss function.
- **Import errors in Modal**: The Modal image might be missing packages. Update the `.pip_install()` list.
- **Timeout**: If the job hits Modal's timeout, add checkpointing and increase `timeout` param.
