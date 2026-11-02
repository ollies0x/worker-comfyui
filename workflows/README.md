# Workflows

The worker takes a ComfyUI workflow exported in **API format**. The UI-format JSON that ships in
ComfyUI's template library (`nodes` and `links`) will not run here: the API needs `class_type`
entries.

## What is in here

| File | Model family | Exported nodes |
|---|---|---|
| `flux2-dev.json` | FLUX.2 dev with the FLUX.2 Turbo LoRA | 19 |
| `qwen-image.json` | Qwen-Image 2.1 turbo, with the prompt-enhancer path | 19 |
| `z-image-turbo.json` | Z-Image-Turbo | 10 |

Model filenames each graph references are listed in `../docs/fleet-models.md`.

## How they were produced, so they can be produced again

No GPU is needed to author or export a workflow, only to run one. The graphs come from ComfyUI
running CPU-only on a plain container:

1. `git clone --branch v0.43.2 https://github.com/comfyanonymous/ComfyUI`, CPU-only PyTorch.
2. `python main.py --cpu --listen 127.0.0.1 --port 8188`.
3. Load the upstream template, then call the conversion the **Workflow > Export (API)** menu calls:
   `await app.graphToPrompt()` in the page, which returns the API-format prompt for the loaded graph.

Templates are the subgraph-style files from `Comfy-Org/workflow_templates`; the frontend flattens
subgraphs during that conversion, which is why three graph nodes become nineteen API nodes for
FLUX.2.

The export is checked, not assumed: each file is loaded against the pinned ComfyUI build and every
node type is resolved against that build's node definitions. A graph with an unknown node type fails
the export rather than shipping a workflow that cannot run.

## How a workflow reaches the endpoint

The worker's API takes the workflow object inline:

```
POST /runsync
{"input": {"workflow": {...}, "images": [...]}}
```

So the caller keeps the JSON on our side and sends it per job. A workflow change does not need an
image rebuild, and the design team can iterate without touching this repo at all. The files here are
the baseline copies, not the delivery mechanism.

## Cost

Every job costs a worker start, the run, and `idle_timeout` seconds of warm idle. The caller writes
one ledger line per job with the wall time and the computed cost. Nothing in this repo estimates
cost, because an estimate that is not measured drifts.
