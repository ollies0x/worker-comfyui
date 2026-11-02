# Fleet model map: what lives where on the endpoint

This fork exists so the fleet's image families and our trained LoRA run on one serverless endpoint
without baking weights into the image. The image carries ComfyUI and our workflows. The weights come
from the `imgen-models` network volume, or from Runpod's cached models where the console can pre-load
them onto the host.

Every filename below was **read off the exported API workflows** in `workflows/` on 2026-11-02, not
copied from a blog post. If a template changes upstream, re-export and update this table in the same
commit.

## Volume layout

```
/runpod-volume/models/diffusion_models/   flux2_dev_fp8mixed.safetensors
                                          qwen_image_2.1_turbo_int8_convrot.safetensors
                                          z_image_turbo_bf16.safetensors
/runpod-volume/models/text_encoders/      mistral_3_small_flux2_bf16.safetensors
                                          qwen3vl_8b_int8_convrot.safetensors
                                          qwen3.5_9b_qwen_image_2.1_pe_t2i.int8_convrot.safetensors
                                          qwen_3_4b.safetensors
/runpod-volume/models/vae/                full_encoder_small_decoder.safetensors
                                          qwen_image_2.1_vae_bf16.safetensors
                                          ae.safetensors
/runpod-volume/models/loras/              Flux_2-Turbo-LoRA_comfyui.safetensors
                                          our trained brand LoRA, one file per brand
```

## What each family needs

| Family | Diffusion model | Text encoder | VAE | Notes |
|---|---|---|---|---|
| FLUX.2 dev | `flux2_dev_fp8mixed.safetensors` | `mistral_3_small_flux2_bf16.safetensors` | `full_encoder_small_decoder.safetensors` | Uses the FLUX.2 Turbo LoRA through `LoraLoaderModelOnly`, `Flux2Scheduler`, `FluxGuidance` |
| Qwen-Image 2.1 turbo | `qwen_image_2.1_turbo_int8_convrot.safetensors` | `qwen3vl_8b_int8_convrot.safetensors`, plus the prompt enhancer `qwen3.5_9b_qwen_image_2.1_pe_t2i.int8_convrot.safetensors` | `qwen_image_2.1_vae_bf16.safetensors` | Two CLIP loaders: the 8B encode path and the prompt-enhancer path |
| Z-Image-Turbo | `z_image_turbo_bf16.safetensors` | `qwen_3_4b.safetensors` | `ae.safetensors` | 8 NFE, 10 api nodes, the smallest graph of the three |
| Our LoRA | as above, unchanged | unchanged | unchanged | `LoraLoaderModelOnly` already sits in the FLUX.2 graph, so a brand LoRA is a file swap and a weight, not a new workflow |

Two corrections worth keeping: FLUX.2's text encoder is **Mistral 3 small**, not the Qwen encoder,
and its VAE is `full_encoder_small_decoder`, not `flux2-vae`. Both were guesses in the first draft of
this table and both were wrong.

## Where the weights actually come from

1. **Cached models** first, where the console can pre-load that model onto the host. No storage cost,
   and the host is warm rather than downloading.
2. **The `imgen-models` volume** for everything cached models cannot serve, including every LoRA.
   One volume, already paid for. No new volume is created for this project.

Watch the storage budget: the Qwen family alone needs three encoder-sized files plus a transformer,
so this volume fills faster than the FLUX.1 era did.

## Not in this image, on purpose

ComfyUI version is pinned by the `COMFYUI_VERSION` build arg so a rebuild is reproducible. Weights,
LoRAs and generated images are never baked in, so an image rebuild is small and fast, and the volume
is the single place a model is stored.
