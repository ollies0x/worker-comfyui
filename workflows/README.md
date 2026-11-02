# Workflows

The worker takes a ComfyUI workflow exported in **API format**. The UI-format JSON that ships in
ComfyUI's template library will not run here: it uses `nodes` and `links`, the API needs
`class_type` entries.

## Why they are not in this repo yet

ComfyUI's own template library (`Comfy-Org/workflow_templates`) publishes UI format only, checked
2026-11-02. Converting UI to API format is done by ComfyUI itself, from the UI, with
`Workflow > Export (API)`. That means the three JSONs are exported once, by a person at a ComfyUI
UI, and committed here. They are not hand-written, because a hand-written graph that has never run
is worse than no graph.

## The three exports this project needs

| File | Model family | Notes |
|---|---|---|
| `flux2-dev.json` | FLUX.2 dev | text-to-image, 30 steps baseline from the image-lab measurements |
| `z-image-turbo.json` | Z-Image-Turbo | 8 steps, CFG 1 |
| `qwen-image.json` | Qwen-Image 2.1 | 8 steps, CFG 1 with turbo weights |
| `lora-pass.json` | any of the above plus `LoraLoader` | weight and trigger words per brand |

## How a workflow reaches the endpoint

The worker's API takes the workflow object inline:

```
POST /runsync
{"input": {"workflow": {...}, "images": [...]}}
```

So the caller keeps the JSON on our side and sends it per job. That means a workflow change does not
need an image rebuild, and the design team can iterate without touching this repo at all. The files
here are the baseline copies, not the delivery mechanism.

## Cost

Every job costs a worker start, the run, and `idle_timeout` seconds of warm idle. The caller writes
one ledger line per job with the wall time and the computed cost; see the fleet's `designer-gpu`
skill. Nothing in this repo estimates cost, because an estimate that is not measured drifts.
