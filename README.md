# ERNIE Image — Information Collection

> ## ⚠️ Disclaimer — Please Read First
>
> **This is a personal information-collection repository. It is not an official Baidu repository and not a public release of ERNIE Image.**
>
> ERNIE Image is a text-to-image generation model released by the ERNIE-Image Team at Baidu on April 15, 2026, under the Apache 2.0 license. Everything in this README is gathered from publicly circulating sources — the Hugging Face model cards, public configuration files, the reference inference code, the benchmark papers ERNIE Image is evaluated against, and independent technical write-ups — and is collected here for personal reference and learning.
>
> I am not affiliated with Baidu or the ERNIE-Image Team. **The information below may be incomplete, inconsistent across sources, or change at any time.** Where sources disagree or where the model card is silent, I label the item as *reported*, *unverified*, or *not disclosed* rather than smoothing it over. I am watching for updates and will sync this README when new first-party information is published.
>
> 👉 For the live in-browser demo, see the [ERNIE Image platform](https://ernie-image.org).

---

## 📖 What is ERNIE Image?

**ERNIE Image** is an open-weight **text-to-image generation model** developed by the **ERNIE-Image Team at Baidu** and released on **April 15, 2026** under the **Apache 2.0 license**. The ERNIE Image model is positioned around three headline capabilities: **long-form text rendering inside generated images**, **complex multi-object instruction following**, and **native bilingual Chinese and English prompting**.

According to the Hugging Face model card, the ERNIE Image denoiser is an **8B single-stream Diffusion Transformer** called `ErnieImageTransformer2DModel`, with 36 layers, a hidden dimension of 4096, an FFN hidden size of 12288, 32 attention heads, and 128 input/output channels. The text conditioning vector is 3072-dimensional. A dedicated lightweight **Prompt Enhancer** component (`Ministral3ForCausalLM`) can expand short user descriptions into richer, structured visual descriptions before they are passed into the diffusion pipeline.

ERNIE Image ships in **two variants** on Hugging Face:

- **ERNIE Image (SFT)** — the quality-oriented base model, typically sampled at ~50 denoising steps with a classifier-free guidance scale of ~4.0.
- **ERNIE Image Turbo** — a distilled variant trained with **DMD (Distribution Matching Distillation)** and reinforcement learning, designed to match the base model's perceptual quality in **8 sampling steps** at guidance scale ~1.0 — roughly a 6× inference speedup.

The ERNIE Image model card states that the 8B DiT runs on a single **24 GB consumer GPU**. Community write-ups on Baidu AI Studio additionally discuss 16 GB local setups and reference a "~15.4B total parameter count" figure that includes the text encoder and Prompt Enhancer alongside the 8B diffusion backbone — these community figures should be treated as unverified until cross-checked against a hands-on deployment.

For the free browser-based demo of the ERNIE Image Turbo variant, see the [ERNIE Image platform](https://ernie-image.org), which embeds the `baidu/ERNIE-Image-Turbo` Hugging Face Space.

---

## 🧱 Reported Architecture

> The technical details below come from the ERNIE Image public configuration files on Hugging Face (`transformer/config.json`, `prompt_enhancer/config.json`, `text_encoder/config.json`) and from the ERNIE Image model card. Items the model card does not disclose are labeled as such.

### Diffusion backbone — `ErnieImageTransformer2DModel`

| Component | Reported Specification |
|---|---|
| Architecture class | `ErnieImageTransformer2DModel` (single-stream Diffusion Transformer) |
| Total parameters | ~8B (DiT backbone only) |
| Layers | 36 |
| Hidden dimension | 4096 |
| FFN hidden size | 12288 |
| Attention heads | 32 |
| Input / output channels | 128 |
| Text conditioning dim (`text_in_dim`) | 3072 |
| Distillation (Turbo variant) | DMD + RL |
| SFT sampling steps | ~50 (guidance ~4.0) |
| Turbo sampling steps | 8 (guidance ~1.0) |
| Reference hardware | Single 24 GB consumer GPU |

### Prompt Enhancer — `Ministral3ForCausalLM`

The ERNIE Image pipeline includes a dedicated Prompt Enhancer component, toggleable via a `use_pe` flag on the `ErnieImagePipeline`. According to its `chat_template.jinja`, the Prompt Enhancer is constrained to accept a short user description plus a target resolution and return only a richer, structured visual description — no explanation, no commentary. The pipeline can return the rewritten text via `output.revised_prompts` for audit and reproducibility.

| Component | Reported Specification |
|---|---|
| Architecture class | `Ministral3ForCausalLM` |
| Hidden dimension | 3072 |
| Layers | 26 |
| Vocabulary size | 131,072 |
| Position encoding | YaRN RoPE |
| Max position length | 262,144 |
| Pipeline toggle | `use_pe=True` / `use_pe=False` |
| Special tokens (partial) | `[SYSTEM_PROMPT]`, `[IMG]`, `[IMG_END]`, `[THINK]` |

### Text encoder — `Mistral3Model` with Pixtral vision config

One of the more unusual details visible in the public configuration: the ERNIE Image `text_encoder/config.json` is **not a traditional CLIP text tower**. It is a `Mistral3Model` containing both a `text_config` and a `vision_config`. The `vision_config` declares `model_type: pixtral`, patch size 14, 24 layers, and hidden dim 1024, with an `image_token_index` field present — hinting at latent multimodal capability at the config level, even though the ERNIE Image model card focuses on text-to-image use.

This is one of the more interesting open questions about ERNIE Image for researchers: the config hints suggest image conditioning pathways that the model card does not currently document, and the behavior of `use_pe` with image inputs is not covered in the public Quickstart.

### Inference pipeline

The reported end-to-end flow of the ERNIE Image pipeline, as assembled from the Quickstart and `infer_demo.py`:

1. User prompt, optionally routed through the Prompt Enhancer when `use_pe=True`.
2. Tokenizer and text / multimodal encoder produce conditioning vectors of dim 3072.
3. DiT denoiser samples a latent — ~50 steps for the SFT ERNIE Image model, 8 steps for ERNIE Image Turbo.
4. VAE decoder maps the latent back to pixels. The pipeline optionally returns `revised_prompts` for audit logs.

---

## ✨ Reported Features of the ERNIE Image Model

### Long-form text rendering inside images
This is the defining reported feature of the ERNIE Image model. On **LongTextBench**, ERNIE Image (with Prompt Enhancer) averages **0.9733** across the EN and ZH subsets, and ERNIE Image Turbo averages **0.9655**. Multi-line marketing copy, comic dialogue, and labeled infographics are reported to stay legible in a single generation, without a separate OCR-refinement pass.

### Complex instruction following
On **GenEval** — the compositional text-to-image evaluation framework covering object count, position, color, and attribute binding — ERNIE Image reaches an **Overall score of 0.8856** without the Prompt Enhancer, leading several contemporary open-weight baselines in the published table.

### Native bilingual Chinese and English
On **OneIG-Bench**, ERNIE Image with Prompt Enhancer scores **0.5750** on the English track and **0.5543** on the Chinese track — a notably small gap relative to most open text-to-image models, which typically degrade sharply on Chinese prompts. The pipeline routes English, Chinese, and mixed prompts through the same encoder path.

### Structured visual content
The ERNIE Image model card explicitly positions the model for **posters, comics, storyboards, and multi-panel compositions** — outputs where text rendering and layout composition matter simultaneously. The architecture is described as tuned for that class of images rather than single-subject portraits alone.

### Seven supported resolutions
The ERNIE Image model card documents seven resolution presets: **1024×1024, 848×1264, 1264×848, 768×1376, 1376×768, 896×1200, and 1200×896** — enough for square posts, vertical posters, horizontal banners, storyboard panels, and UI mockups without cropping.

### Fast 8-step Turbo variant
ERNIE Image Turbo is distilled with DMD and RL to reach **8 sampling steps at guidance 1.0**, roughly **6× faster** than the 50-step SFT model at reported comparable perceptual quality. The free browser demo at the [ERNIE Image platform](https://ernie-image.org) runs the Turbo variant.

### Open weights under Apache 2.0
Both variants are published on Hugging Face under the Apache 2.0 license, permitting commercial use subject to the license terms.

---

## 📊 Benchmark Numbers from the Model Card

The tables below are copied verbatim from the ERNIE Image and ERNIE Image Turbo Hugging Face model cards, evaluated 2026-04-15. Scores are in the 0–1 range; higher is better.

> **Note on rankings:** Automatic text-to-image benchmarks drift as models saturate and evaluator versions age. The GenEval 2 research has explicitly flagged GenEval drift and recommended continuous evaluator audits. Numbers here reflect the scripts and evaluator versions used on 2026-04-15; a re-run on a newer evaluator is likely to produce different absolute numbers even without model changes. Treat every score on this page as a **point-in-time snapshot**, not a durable ranking.

### GenEval — compositional text-to-image

| Model | Single Object | Two Object | Counting | Colors | Position | Attribute Binding | **Overall** |
|---|---:|---:|---:|---:|---:|---:|---:|
| **ERNIE Image (w/o PE)** | 1.0000 | 0.9596 | 0.7781 | 0.9282 | 0.8550 | 0.7925 | **0.8856** |
| **ERNIE Image (w/ PE)** | 0.9906 | 0.9596 | 0.8187 | 0.8830 | 0.8625 | 0.7225 | **0.8728** |
| Qwen-Image | 0.9900 | 0.9200 | 0.8900 | 0.8800 | 0.7600 | 0.7700 | 0.8683 |
| **ERNIE Image Turbo (w/o PE)** | 1.0000 | 0.9621 | 0.7906 | 0.9202 | 0.7975 | 0.7300 | **0.8667** |
| **ERNIE Image Turbo (w/ PE)** | 0.9938 | 0.9419 | 0.8375 | 0.8351 | 0.7950 | 0.7025 | **0.8510** |
| FLUX.2-klein-9B | 0.9313 | 0.9571 | 0.8281 | 0.9149 | 0.7175 | 0.7400 | 0.8481 |
| Z-Image | 1.0000 | 0.9400 | 0.7800 | 0.9300 | 0.6200 | 0.7700 | 0.8400 |
| Z-Image-Turbo | 1.0000 | 0.9500 | 0.7700 | 0.8900 | 0.6500 | 0.6800 | 0.8233 |

One observation worth calling out: enabling the Prompt Enhancer raises ERNIE Image **Counting** from 0.7781 → 0.8187 and **Position** from 0.8550 → 0.8625, but lowers **Overall** from 0.8856 → 0.8728. This looks more like a preference shift — the enhancer trades some attribute-binding precision for richer, more structured descriptions — than a strict improvement, so `use_pe` is best treated as a per-scene switch rather than a global always-on flag.

### OneIG-EN — English track

| Model | Alignment | Text | Reasoning | Style | Diversity | **Overall** |
|---|---:|---:|---:|---:|---:|---:|
| **ERNIE Image (w/ PE)** | 0.8678 | 0.9788 | 0.3566 | 0.4309 | 0.2411 | **0.5750** |
| **ERNIE Image Turbo (w/ PE)** | 0.8676 | 0.9666 | 0.3537 | 0.4191 | 0.2212 | **0.5656** |
| **ERNIE Image (w/o PE)** | 0.8909 | 0.9668 | 0.2950 | 0.4471 | 0.1687 | **0.5537** |

### OneIG-ZH — Chinese track

| Model | Alignment | Text | Reasoning | Style | Diversity | **Overall** |
|---|---:|---:|---:|---:|---:|---:|
| **ERNIE Image (w/ PE)** | 0.8299 | 0.9539 | 0.3056 | 0.4342 | 0.2478 | **0.5543** |
| **ERNIE Image Turbo (w/ PE)** | 0.8258 | 0.9386 | 0.3043 | 0.4208 | 0.2281 | **0.5435** |
| **ERNIE Image (w/o PE)** | 0.8421 | 0.8979 | 0.2656 | 0.4212 | 0.1772 | **0.5208** |

The EN–ZH gap on ERNIE Image (w/ PE) is only **0.0207 Overall**, which is unusually small for an open text-to-image model and lines up with the model's positioning as genuinely bilingual rather than "English with a Chinese adapter."

### LongTextBench — long text rendering inside images

| Model | EN | ZH | **Avg** |
|---|---:|---:|---:|
| **ERNIE Image (w/ PE)** | 0.9804 | 0.9661 | **0.9733** |
| **ERNIE Image Turbo (w/ PE)** | 0.9675 | 0.9636 | **0.9655** |
| **ERNIE Image Turbo (w/o PE)** | 0.9602 | 0.9675 | **0.9639** |
| **ERNIE Image (w/o PE)** | 0.9679 | 0.9594 | **0.9636** |

LongTextBench evaluates the legibility and accuracy of long-form text rendered **inside** generated images (posters, UI mockups, dialogue bubbles, signage), with separate EN and ZH subsets. The ERNIE Image model card's framing of the model as "text rendering and layout-sensitive" lines up with these numbers being the model's strongest public result.

---

## 🆚 ERNIE Image vs. Other Open Text-to-Image Models

A frequently asked question in the community is how the ERNIE Image model stacks up against the current leaders in open-weight text-to-image. Based on the GenEval table published in the ERNIE Image model card:

| Feature | **ERNIE Image** | Qwen-Image | FLUX.2-klein-9B | Z-Image |
|---|---|---|---|---|
| GenEval Overall | **0.8856** | 0.8683 | 0.8481 | 0.8400 |
| Parameters | 8B DiT | — | 9B | — |
| Backbone | Single-stream DiT | DiT | DiT (MoE) | DiT |
| Long text rendering benchmark | ✅ LongTextBench 0.9733 | — | — | — |
| Native bilingual (EN + ZH) | ✅ OneIG-ZH 0.5543 | ✅ | ❌ | ✅ |
| Prompt Enhancer component | ✅ `use_pe` flag | — | — | — |
| Distilled few-step variant | ✅ Turbo, 8 steps | — | ✅ klein | ✅ Z-Image-Turbo |
| License | Apache 2.0 | Apache 2.0 | Apache 2.0 | Apache 2.0 |
| 24 GB consumer GPU | ✅ | — | — | ✅ |

The headline differentiators on paper are **(a)** the top GenEval Overall score among the open-weight models listed in the ERNIE Image model card, and **(b)** a published LongTextBench result at 0.9733 — LongTextBench is not currently reported by most other open text-to-image models, which makes direct comparison difficult but also makes ERNIE Image the clearest reference point for that capability. The headline caveat is that GenEval drift is a real phenomenon and these scores should be read as 2026-04-15 snapshots, not durable rankings.

---

## 🔀 ERNIE Image vs. Closed-Source Platforms (DALL·E, Imagen, Midjourney)

Closed-source text-to-image platforms typically do not publish their scores on the same academic benchmarks ERNIE Image reports, so a direct GenEval / OneIG comparison is not possible. What can be compared are product-level dimensions:

| Dimension | **ERNIE Image** | OpenAI (DALL·E 3 / GPT Image) | Google Imagen 3 | Midjourney |
|---|---|---|---|---|
| Supply model | Open weights, self-hostable | Hosted API, per-image billing | Hosted API via Vertex AI | Hosted service, subscription |
| License | Apache 2.0 | Proprietary, terms of service | Proprietary, Google Cloud terms | Proprietary, Discord / web ToS |
| Native long-text rendering | ✅ LongTextBench 0.9733 reported | Limited, case-by-case | Improving, not benchmark-reported | Historically weak |
| Native Chinese prompting | ✅ OneIG-ZH 0.5543 reported | Implicit via translation | Implicit via translation | Implicit via translation |
| Watermarking / provenance | User responsibility | Platform policy | SynthID watermark (Google documentation) | Platform policy |
| Long-term availability | Local copy survives any deprecation | Subject to model retirement (e.g. DALL·E 3 → gpt-image-1 on Azure) | Subject to Google Cloud product lifecycle | Subject to Midjourney policy changes |
| Try it free without install | [ernie-image.org](https://ernie-image.org) — embedded Hugging Face Space | Requires account + credit | Requires Google Cloud project | Requires Discord + paid plan |

The headline differentiator for ERNIE Image against closed platforms is not raw quality — that comparison is evaluator-dependent and contested — but **supply model**: ERNIE Image weights can be downloaded once and run locally indefinitely, independent of any provider's product lifecycle decisions. For teams doing long-running creative pipelines or building on top of a specific version for reproducibility, that is a categorically different guarantee than a hosted API gives.

---

## 🎯 Reported Use Cases

Based on the capabilities documented in the ERNIE Image model card and the LongTextBench / GenEval / OneIG-Bench results, the ERNIE Image model is positioned for the following workflows. None of these have been independently verified in this repository; they are transcribed from the public positioning.

- **Poster and marketing-graphic generation** — where multi-line text has to render legibly inside the image in a single pass, not as a post-hoc overlay.
- **Comics and multi-panel storyboards** — where layout composition and dialogue-bubble text are generated together.
- **Bilingual campaigns** — a single creative concept rendered in English and Chinese from the same prompt template, without swapping models between markets.
- **Infographics and labeled diagrams** — dense-label images where LongTextBench-style rendering accuracy is load-bearing.
- **UI mockups and signage** — images where both the scene and the embedded text matter to downstream reviewers.
- **Local, reproducible creative pipelines** — workflows that need a fixed model checkpoint that will not be retired by a hosted provider.
- **AI research** — open weights under Apache 2.0 for studying single-stream DiT architectures, DMD distillation, RL-based diffusion fine-tuning, and bilingual text-to-image training.

For low-friction evaluation without a local install, the browser demo at the [ERNIE Image platform](https://ernie-image.org) runs the ERNIE Image Turbo variant directly.

---

## ❓ Frequently Asked Questions

### Is ERNIE Image open source?
**Yes.** The ERNIE Image model is released under the Apache 2.0 license, with open weights published on Hugging Face for both the SFT base model and the 8-step Turbo distillation. Commercial use is permitted subject to the Apache 2.0 terms.

### Who built the ERNIE Image model?
The ERNIE Image model is developed by the **ERNIE-Image Team at Baidu** and was released on **April 15, 2026**. For anything authoritative, refer to the ERNIE Image Hugging Face model cards and Baidu's own communications; this repository is a personal information collection and is not affiliated with Baidu.

### What is the difference between ERNIE Image and ERNIE Image Turbo?
Both variants share the same 8B single-stream Diffusion Transformer backbone. **ERNIE Image (SFT)** is the quality-oriented base model, typically sampled at ~50 denoising steps with classifier-free guidance ~4.0, and gives slightly higher Overall on GenEval. **ERNIE Image Turbo** is distilled with DMD (Distribution Matching Distillation) and reinforcement learning to reach **8 sampling steps at guidance ~1.0** — roughly 6× faster wall-clock inference at reported comparable perceptual quality. The free browser demo at the [ERNIE Image platform](https://ernie-image.org) runs the Turbo variant.

### How good is ERNIE Image at rendering long text inside images?
On **LongTextBench**, ERNIE Image with Prompt Enhancer averages **0.9733** across EN and ZH subsets, and the Turbo variant averages **0.9655**. The model card explicitly positions ERNIE Image for posters, comics, and labeled infographics where multi-line text rendering is load-bearing. LongTextBench is not reported by most other open text-to-image models, so direct head-to-head comparison with Qwen-Image, FLUX, Z-Image, or the closed-source platforms is not currently possible at the benchmark level.

### Does ERNIE Image handle Chinese prompts as well as English?
On **OneIG-Bench** with Prompt Enhancer enabled, ERNIE Image scores **0.5750** on the English track and **0.5543** on the Chinese track — a gap of only **0.0207** Overall, which is unusually small for an open text-to-image model. Prompts can be written in English, Chinese, or mixed, and are routed through the same encoder path.

### What architecture does the ERNIE Image model use?
The ERNIE Image denoiser is an **8B single-stream Diffusion Transformer** named `ErnieImageTransformer2DModel`, with 36 layers, hidden dimension 4096, FFN hidden size 12288, 32 attention heads, and 128 I/O channels. The text conditioning vector is 3072-dimensional. Alongside the DiT the pipeline ships a dedicated Prompt Enhancer component (`Ministral3ForCausalLM`, 26 layers, hidden dim 3072, vocab 131,072) toggleable via a `use_pe` flag. The text encoder is unusually configured as a `Mistral3Model` containing both `text_config` and a Pixtral-typed `vision_config`, hinting at latent multimodal capability at the config level.

### When should I turn the Prompt Enhancer off?
Enabling the Prompt Enhancer on ERNIE Image raises GenEval **Counting** from 0.7781 → 0.8187 and **Position** from 0.8550 → 0.8625, but lowers the **Overall** score from 0.8856 → 0.8728. The enhancer trades some attribute-binding precision for richer, more structured descriptions. Turn it **on** for detail-rich, layout-heavy, or structured scenes where elaboration helps; turn it **off** for strict attribute-binding prompts where precision on the literal user wording matters more than descriptive elaboration. Treat `use_pe` as a per-scene switch rather than a global always-on flag.

### Can ERNIE Image run on a consumer GPU?
**Yes.** The ERNIE Image model card states the 8B DiT runs on a single **24 GB consumer GPU**. Community write-ups on Baidu AI Studio additionally reference 16 GB local setups, though this should be treated as community-reported and validated against the reference implementation before production use.

### How fast is ERNIE Image Turbo?
ERNIE Image Turbo samples in **8 denoising steps** with classifier-free guidance scale ~1.0, versus ~50 steps at guidance ~4.0 for the SFT base model — roughly a **6× reduction in denoising-step cost**. Absolute wall-clock numbers depend on GPU generation, resolution, batch size, and whether the Prompt Enhancer is enabled; no single "seconds per image" figure from the model card should be treated as universal.

### What image resolutions does ERNIE Image support?
The ERNIE Image model card documents **seven resolution presets**: 1024×1024, 848×1264, 1264×848, 768×1376, 1376×768, 896×1200, and 1200×896 — covering square, portrait, landscape, tall, and wide aspect ratios without requiring post-hoc cropping.

### How does ERNIE Image compare to Qwen-Image, FLUX.2-klein, and Z-Image?
In the GenEval table published in the ERNIE Image model card (evaluated 2026-04-15), ERNIE Image (w/o PE) reaches **0.8856 Overall**, ahead of Qwen-Image at 0.8683, FLUX.2-klein-9B at 0.8481, and Z-Image at 0.8400. On LongTextBench, ERNIE Image (w/ PE) reaches **0.9733 average** — LongTextBench is not currently published by the other three models, so long-text-rendering head-to-head comparison is not available at the benchmark level. Benchmark drift means these rankings are a 2026-04-15 snapshot, not a durable order.

### What is not publicly disclosed about the ERNIE Image model?
The ERNIE Image model card **does not disclose** training dataset sources, licensing provenance of training data, dataset scale, synthetic-data fraction, the specific diffusion training objective (ε-prediction vs v-prediction), alignment reward model construction, filtering and deduplication strategy, or red-team safety evaluation results. Deployments with compliance requirements — especially in jurisdictions with explicit generative-AI content rules — should treat these as open questions rather than infer them from the model's behavior.

### Where can I try ERNIE Image without installing it?
The [ERNIE Image platform](https://ernie-image.org) embeds the ERNIE Image Turbo Hugging Face Space in-browser: type a prompt in English or Chinese, no account, no download, no cost. Queue times depend on Space traffic. For production deployments, refer to the Hugging Face model cards and the reference inference code for `ErnieImagePipeline` and the SGLang HTTP serving endpoint.

### Will this README be updated?
Yes. I am tracking the Hugging Face model cards, the Baidu ERNIE Image GitHub repository, the community write-ups on Baidu AI Studio, and independent discussion. This README will be synced when the model cards add new benchmarks, when new quantization variants (GGUF, AWQ) are published, when additional resolution presets or pipeline features ship, or when independent reproductions of the reported LongTextBench / GenEval / OneIG numbers become available.

---

## 📚 Sources & Updates

This README is compiled from publicly circulating information about the ERNIE Image model as of the date below. Sources referenced:

**First-party — Baidu / Hugging Face**
- ERNIE Image Hugging Face model card (base SFT variant) — source of all benchmark tables transcribed above.
- ERNIE Image Turbo Hugging Face model card — source of the 8-step / DMD-RL distillation details.
- `baidu/ERNIE-Image-Turbo` Hugging Face Space — the live demo embedded at the [ERNIE Image platform](https://ernie-image.org).
- ERNIE Image configuration files on Hugging Face: `transformer/config.json`, `prompt_enhancer/config.json`, `text_encoder/config.json`, and `prompt_enhancer/chat_template.jinja`. These are the source for the DiT layer count, hidden dimensions, Prompt Enhancer architecture, and text encoder multimodal config fields.
- Baidu ERNIE Image GitHub repository — source for `infer_demo.py` and the `revised_prompts` audit pattern.
- Baidu AI Studio community write-ups — source for the "~15.4B total parameters" and 16 GB community deployment figures (treated as community-reported, unverified).

**Benchmark methodology papers**
- GenEval: compositional text-to-image evaluation — the framework behind the Overall / Counting / Position / Attribute Binding columns transcribed above.
- OneIG-Bench: multi-dimensional text-to-image evaluation — the framework behind the Alignment / Text / Reasoning / Style / Diversity columns, with separate EN and ZH tracks.
- LongTextBench — the framework behind the long-text-inside-image rendering scores.
- Scalable Diffusion Models with Transformers (the DiT paper) — the academic origin of the single-stream Diffusion Transformer architecture family ERNIE Image belongs to.
- Latent Diffusion Models (LDM) — the latent-space diffusion framework ERNIE Image's pipeline follows.
- Classifier-Free Diffusion Guidance — the CFG method referenced by the ERNIE Image SFT sampling defaults.
- Distribution Matching Distillation (DMD) — the distillation method ERNIE Image Turbo is based on.
- GenEval 2 — the follow-up research flagging benchmark drift on GenEval, referenced in the snapshot caveats above.

**Live demo**
- [ernie-image.org](https://ernie-image.org) — the **ERNIE Image platform** with the free in-browser demo and the live embed of `baidu/ERNIE-Image-Turbo`.

**Last updated:** 2026-04-15
**Status:** Initial public release tracked · monitoring for model card updates and independent reproductions
**Will be updated when:** the model cards publish additional benchmarks, new quantization variants ship, independent LongTextBench / GenEval / OneIG reproductions are published, or new pipeline features are documented.

---

<p align="center">
  <em>This is a personal information-collection repository for the ERNIE Image text-to-image model.<br>
  For the free in-browser demo, see <a href="https://ernie-image.org">the ERNIE Image platform</a>.</em>
</p>
