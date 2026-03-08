# Modal Deployment Patterns

## Modal app template

**Important:** Modal (v1.3+) does NOT auto-mount sibling files — only the entrypoint `.py` is included. Use `image.add_local_file()` to include `train.py` and any other files the function needs. `modal.Mount` is removed.

```python
import json
from pathlib import Path

import modal

local_dir = Path(__file__).parent
app = modal.App("research-EXPERIMENT-NAME")

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
    return train_and_eval(config)

@app.local_entrypoint()
def main():
    config = json.loads(open("config.json").read())
    result = train.remote(config)
    with open("results.json", "w") as f:
        json.dump(result, f, indent=2)
    print(json.dumps(result, indent=2))
```

Run with: `cd .research/experiments/NN-name && modal run modal_app.py`

## Local validation before deploying

Before spending GPU credits:
1. Check syntax: `python -c "import ast; ast.parse(open('train.py').read())"`
2. Check imports resolve (in the Modal image context — just review mentally)
3. If feasible, dry-run on CPU with 1 batch: `python train.py --dry-run` (if you built that flag in)

## Checkpointing for longer runs

For runs that might take >10 min, save checkpoints to the Modal Volume:

```python
ckpt_dir = "/vol/checkpoints/EXPERIMENT-NAME"
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

## Common failure patterns

- **OOM**: Reduce batch size first. Then try gradient accumulation. Then try a smaller model. Then try a cheaper GPU with less VRAM only as last resort (to avoid changing cost estimates).
- **Training divergence**: Check learning rate (try 10x lower), check data loading, check loss function.
- **Import errors in Modal**: The Modal image might be missing packages. Update the `.pip_install()` list.
- **Timeout**: If the job hits Modal's timeout, add checkpointing and increase `timeout` param.
