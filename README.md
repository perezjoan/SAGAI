# Streetscape Analysis with Generative AI (SAGAI)

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/Version-v2.3-brightgreen)](https://github.com/perezjoan/SAGAI/releases)
[![Colab Compatible](https://img.shields.io/badge/Google%20Colab-Compatible-yellow.svg)](https://colab.research.google.com/)
[![UVLM](https://img.shields.io/badge/VLM%20Engine-UVLM%20v4.0.0-orange.svg)](https://github.com/perezjoan/UVLM)

**SAGAI** is an open-source workflow for scoring and mapping street-level urban environments using **Vision-Language Models (VLMs)**. It automates the full pipeline: from street sampling via OpenStreetMap, to imagery retrieval via Google Street View, multi-model VLM scoring via **[UVLM](https://github.com/perezjoan/UVLM)**, and geospatial aggregation into thematic maps.

Starting with v2.x, SAGAI is a **single unified Google Colab notebook** with six sequential blocks (plus optional block 0 to resume a case study). VLM inference is powered by the [UVLM](https://github.com/perezjoan/UVLM) package (pinned to **v4.0.0** since SAGAI v2.3 for reproducibility), which supports **24 model checkpoints** across five families (LLaVA-NeXT, Qwen2.5-VL, Qwen3-VL, InternVL3.5, and Gemma 4) with multi-task prompting, consensus validation, and chain-of-thought reasoning.

💡 **Zero-shot. Multi-model. Prompt-based. One notebook.**

---

## 🌍 What Does SAGAI Do?

SAGAI combines open-access geospatial tools with vision-language AI into a single pipeline:

- ✅ **OpenStreetMap (OSM)** — automatic extraction of street networks within a bounding box, with configurable point sampling along streets
- 📸 **Google Street View API** — batch downloading of street-level images at each point, with multi-directional capture (left, right, front, back)
- 🧠 **UVLM** — unified inference across 24 VLM checkpoints (LLaVA-NeXT, Qwen2.5-VL, Qwen3-VL, InternVL3.5, and Gemma 4), with multi-task prompting, consensus validation, and reasoning support
- 🗺️ **GeoPandas + Folium** — geospatial aggregation and interactive HTML map generation at point and street levels

No pretraining, no fine-tuning, no human annotation required. Define your scoring tasks as natural language prompts and run them at scale.

---

## 📐 Architecture

SAGAI v2.x is organized as a single Colab notebook with **six sequential blocks**, plus optional block 0:

![SAGAI Architecture](figure_sagai_architecture.svg)

| Block | Function | Description |
|-------|----------|-------------|
| **Block 0** | Resume Case Study | Optional. |
| **Block 1** | OSM Point Generator | Interactive map for defining a study area. Extracts streets from OSM and generates sample points along them. |
| **Block 2** | Street View Downloader | Batch downloads Google Street View images at each point in up to 4 directions. Requires a Google Maps API key. |
| **Block 3** | VLM Loader (UVLM) | Installs the [UVLM package](https://github.com/perezjoan/UVLM) (pinned to v4.0.0) and loads a selected model — family first, then checkpoint — with quantization and device placement. |
| **Block 4** | Task Configuration | Defines analysis tasks (prompt, task type, format), generation parameters, and consensus settings via interactive widgets. |
| **Block 5** | Run Analysis | Processes all images through the configured tasks using UVLM's batch engine. Outputs a CSV with one row per image. |
| **Block 6** | Aggregation & Mapping | Aggregates image-level scores to point and street levels. Generates interactive HTML maps with Folium. |

Blocks 1–2 handle geospatial data. Blocks 3–5 handle VLM inference via UVLM. Block 6 handles visualization. Optional block 0 is to resume a case study. Each block can be re-run independently.

---

## 🚀 Quick Start

SAGAI runs in **Google Colab** with no local installation required.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/perezjoan/SAGAI/blob/main/SAGAI.ipynb)

1. **Open the notebook** in Google Colab (link above)
2. **Select a GPU runtime**: `Runtime` → `Change runtime type` → `T4 GPU`
3. **Block 1**: Draw a bounding box on the map to define your study area. Click "Generate points" to sample along streets.
4. **Block 2**: Enter your Google Maps API key. Click "Download images" to fetch Street View imagery at each point.
5. **Block 3**: Select a VLM model (e.g., Qwen2.5-VL 7B, 4-bit) and click "Load model".
6. **Block 4**: Define your scoring tasks — write a prompt, choose a task type (numeric/category/boolean/text), set generation parameters. Click "Apply".
7. **Block 5**: Run the analysis. Results are saved to Google Drive as one CSV per model checkpoint (`Score_Analysis_<checkpoint>.csv`).
8. **Block 6**: Select a score column, an aggregation method, and a view filter. Generate interactive maps.

### Prerequisites

- A **Google account** (for Colab and Drive)
- A **Google Maps API key** with Street View Static API enabled (for Block 2)
- A **Hugging Face token** stored in Colab secrets (for gated models like LLaMA3-based or Gemma checkpoints — optional for public models like Qwen and InternVL3.5)

---

## 🧠 VLM Engine: UVLM

Blocks 3–5 are powered by **[UVLM](https://github.com/perezjoan/UVLM)** (Unified Vision-Language Model Loader), a separate open-source package. SAGAI v2.3 pins UVLM to **v4.0.0**, which runs on `transformers >= 5.15` (all five backends re-validated on GPU under that stack on the UVLM side). UVLM provides:

- **24 model checkpoints** — from 1B to 110B parameters
- **Multi-backend abstraction** — automatically routes to the correct inference pipeline (LLaVA-NeXT, Qwen2.5-VL, Qwen3-VL, InternVL3.5, or Gemma 4)
- **Multi-task prompt builder** — up to 10 tasks per run, with four response types (numeric, category, boolean, text)
- **Consensus validation** — majority voting across 2–5 repeated inferences per task
- **Chain-of-thought reasoning** — adjustable token budget up to 1,500 for custom reasoning strategies
- **Truncation detection** — flags responses that hit the token limit
- **Batch processing** — resume mode, schema upgrade, checkpoint saves
- **4-bit quantization** — models up to ~34B run on free Colab GPUs (T4); Gemma 4 E4B/12B and 72B+ models need larger-VRAM runtimes (L4 / A100)

UVLM is installed automatically when you run Block 3. For standalone use outside SAGAI, see the [UVLM repository](https://github.com/perezjoan/UVLM).

---

## 📊 Example Outputs

Two pilot case studies from SAGAI v1.0 are included in this repository:

- **Nice, France** — A linear urban corridor following the Paillon Valley
- **Vienna, Austria** — A heterogeneous peri-urban area in the Penzing-Wolfersberg sector

🔗 [Download output archive (excluding Street View imagery)](output%20nice%20vienna%20SAGAI%20v1-0.zip)

Due to Google's Terms of Service, raw Street View images are not distributed.

---

## 📦 Repository Structure

```
SAGAI/
├── SAGAI.ipynb                       # Unified notebook (all 6 + 1 blocks)
├── README.md                           # This file
├── LICENSE                             # Apache License 2.0
├── .gitignore
├── VERSIONS.txt                        # Version history
├── figure_sagai_architecture.svg       # Architecture diagram
└── output nice vienna SAGAI v1-0.zip   # Example outputs from v1.0
```

---

## ⚠️ What Changed in v2.3

- **UVLM pinned to v4.0.0.** Block 3 now installs `git+https://github.com/perezjoan/UVLM.git@v4.0.0` instead of tracking `main`, so SAGAI's behavior no longer changes silently with UVLM releases. UVLM 4.x runs on `transformers >= 5.15`.
- **24 checkpoints across five families.** Qwen3-VL, InternVL3.5, and Gemma 4 are now selectable in Block 3, via a two-level (family → model) selector.
- **Per-model output CSVs.** Block 5 output filenames are derived from the loaded checkpoint (`Score_Analysis_<checkpoint>.csv`), so different models never append into the same CSV — resume mode and schema upgrades are per-model. Blocks 0 and 6 handle multiple coexisting CSVs and use the most recent by default.
- **Documentation updates.** UVLM citation updated to the published version; third-party attribution extended to the new model families.

## ⚠️ What Changed in v2.0

| Aspect | v1.x | v2.x |
|--------|------|------|
| **Structure** | 4 separate notebooks | 1 unified notebook (6 + 1 blocks) |
| **VLM support** | LLaVA-only (inline code) | 11 models via UVLM package (LLaVA + Qwen) |
| **Multi-task** | Single task per run | Up to 10 tasks per run |
| **Consensus** | Not available | Majority voting (2–5 runs) |
| **Reasoning** | Not available | Chain-of-thought with adjustable token budget |
| **Truncation** | Not detected | Per-task truncation flags |
| **Dependencies** | Inline, fragile | UVLM package installed from GitHub |
| **Maps** | Matplotlib static | Folium interactive HTML maps |

---

## 📚 Citation

If you use SAGAI in your research, please cite:

> Perez, J. and Fusco, G. (2025). *Streetscape Analysis with Generative AI (SAGAI): Vision-Language Assessment and Mapping of Urban Scenes*. Geomatica, 77(2), 100063. Available at: https://www.sciencedirect.com/science/article/pii/S1195103625000199

If you use the UVLM engine specifically, please also cite:

> Perez, J. and Fusco, G. (2026). *UVLM: A Modular Python Package for Unified Vision–Language Model Loading, Inference and Comparison*. Software 2026, 5(3), 30. Available at: https://www.mdpi.com/2674-113X/5/3/30

---

## 🪪 License and Attribution

SAGAI is released under the [Apache License 2.0](LICENSE).

Third-party components:

- [UVLM](https://github.com/perezjoan/UVLM) — Vision-language model inference engine (Apache 2.0)
- [LLaVA-NeXT](https://github.com/haotian-liu/LLaVA) — Visual instruction tuning models (Apache 2.0)
- [Qwen2.5-VL](https://github.com/QwenLM/Qwen2.5-VL) — Vision-language models (Apache 2.0)
- [Qwen3-VL](https://github.com/QwenLM/Qwen3-VL) — Vision-language models (Apache 2.0)
- [InternVL](https://github.com/OpenGVLab/InternVL) — Vision-language models (MIT)
- [Gemma 4](https://ai.google.dev/gemma) — Multimodal open models (Apache 2.0)
- [Hugging Face Transformers](https://github.com/huggingface/transformers) — Model loading and inference (Apache 2.0)
- [BitsAndBytes](https://github.com/bitsandbytes-foundation/bitsandbytes) — Quantization library (MIT)
- [OpenStreetMap](https://www.openstreetmap.org/copyright) — Street network data (ODbL 1.0)
- Google Street View imagery is accessed via API and subject to Google's [Terms of Service](https://maps.google.com/help/terms_maps/)

---

## ✨ Acknowledgments

This research is supported by the [emc2 project](https://emc2-dut.org/) co-funded by **ANR (France)**, **FFG (Austria)**, **MUR (Italy)**, and **Vinnova (Sweden)** under the **Driving Urban Transition Partnership**, which has been co-funded by the European Commission.

## 🏢 Developer

SAGAI is developed by [Joan Perez](https://orcid.org/0000-0003-3003-0895), founder of **Urban Geo Analytics** — an independent research and consulting practice focused on geospatial modeling, AI for cities, and open-source urban analytics. 🌐 [urbangeoanalytics.com](https://urbangeoanalytics.com/)

---

## 📫 Feedback and Contributions

Feel free to open an issue or pull request. Contributions and forks are welcome!

🔗 [GitHub Discussions](https://github.com/perezjoan/SAGAI/discussions) — Share use cases, ideas, and extensions.
