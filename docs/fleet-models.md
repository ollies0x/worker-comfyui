# Fleet model map: what lives where on the endpoint

This fork exists so the fleet's three image families and our trained LoRA run on one serverless
endpoint without baking weights into the image. The image carries ComfyUI and our workflows. The
weights come from the `imgen-models` network volume, or from Runpod's cached models where the
console can pre-load them onto the host.

## Volume layout

```
/runpod-volume/models/diffusion_models/   flux2_dev_fp8mixed.safetensors, z_image_turbo_bf16.safetensors
/runpod-volume/models/text_encoders/      qwen_3_4b.safetensors
/runpod-volume/models/vae/                flux2-vae.safetensors, ae.safetensors
/runpod-volume/models/loras/              our trained LoRA, one file per brand
/runpod-volume/models/checkpoints/        (only if a single-file checkpoint is ever needed)
```

## What each family needs

| Family | Diffusion model | Text encoder | VAE | Notes |
|---|---|---|---|---|
| FLUX.2 dev | `flux2_dev_fp8mixed.safetensors` | shared, confirm the exact file on the ComfyUI FLUX.2 page when exporting | `flux2-vae.safetensors` | Worth 4MP. Confirm at export whether the text encoder is the same file the Qwen family uses |
| Z-Image-Turbo | `z_image_turbo_bf16.safetensors` | `qwen_3_4b.safetensors` | `ae.safetensors` | 8 NFE, fits 16 GB |
| Qwen-Image 2.1 | turbo diffusion model, or base plus the turbo LoRA | shared with FLUX.2 | shared with FLUX.2 | Sampler at 8 steps, CFG 1 |
| Our LoRA | as above, unchanged | unchanged | unchanged | `LoraLoader` node in the workflow, weight per brand |

Filenames above come from the ComfyUI documentation for each model. Anything marked "confirm at
export" is not guessed here: it is read off the workflow template at the moment the workflow is
exported, then written into this table.

## Where the weights actually come from

1. **Cached models** first, where the console can pre-load that model onto the host. No storage
   cost, and the host is warm rather than downloading.
2. **The `imgen-models` volume** for everything cached models cannot serve, including every LoRA.
   One volume, already paid for. No new volume is created for this project.

## Not in this image, on purpose

ComfyUI version is pinned by the `COMFYUI_VERSION` build arg so a rebuild is reproducible. Weights,
LoRAs and generated images are never baked in, so an image rebuild is small and fast, and the
volume is the single place a model is stored.
