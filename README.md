# Streetscape Analysis with Generative AI (SAGAI)

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/Version-v1.1-brightgreen)](https://github.com/perezjoan/SAGAI/releases)
[![Colab Compatible](https://img.shields.io/badge/Google%20Colab-Compatible-yellow.svg)](https://colab.research.google.com/)

**SAGAI** is an open-source, modular workflow for scoring and mapping street-level urban environments using **generative vision-language models**—specifically a lightweight version of **LLaVA (Large Language and Vision Assistant)**. It enables scalable, prompt-driven interpretation of **Google Street View** imagery using only a geographic bounding box—requiring no pre-labeled data, specialized hardware, or deep learning expertise.

SAGAI automates the full pipeline: from street sampling via OpenStreetMap, to imagery retrieval, semantic scoring with LLaVA, and geospatial aggregation into thematic maps. Designed for **Google Colab**, it runs entirely in-browser for fast and lightweight deployment.

💡 **Zero-shot. Lightweight. Prompt-based. Fully deployable.**

---

## 🌍 What does SAGAI do?

SAGAI combines several powerful open-access tools into a unified geospatial AI pipeline:
- ✅ **OpenStreetMap (OSM)** — for automatic extraction of street networks within a defined bounding box  
- 📸 **Google Street View API** — for downloading real-world street-level images in multiple directions  
- 🧠 **LLaVA (v1.6 Mistral-7B, quantized)** — for performing **zero-shot** scoring using customizable natural language prompts  
- 🗺️ **GeoPandas + Matplotlib** — for geospatial joins, aggregation, and generation of point- and street-level thematic maps

It requires **no pretraining, no fine-tuning, and no human annotation**—making it suitable for rapid deployment in new cities or study areas.

It supports a growing range of tasks, including:
- **Urban vs Rural Scene Classification**  
- **Storefront Presence Detection**  
- **Sidewalk Width Estimation**  
- 🧩 Plus any other **prompt-driven visual analysis** task—simply adapt the prompt template.
---

## ⚠️ Updates
# v1.1
In early 2025, Google Colab introduced major backend updates (Python 3.12, NumPy ≥ 2.0, PyTorch ≥ 2.2) that caused compatibility issues with the original SAGAI Module 3, which relied on the GitHub version of the LLaVA repository. This GitHub codebase evolves rapidly and is not synchronized with the stable Hugging Face checkpoints, leading to errors such as missing modules, incompatible constructors, and NumPy binary mismatches. To ensure long-term stability, Module 3 has now been fully refactored to use the official Hugging Face LLaVA-Next (v1.6 Mistral) implementation (LlavaNextProcessor and LlavaNextForConditionalGeneration), without cloning or importing the LLaVA GitHub repository. This update includes: a complete switch to HF-managed model loading (no more git clone, no custom vision tower loading), a fully rewritten multimodal forward pass using apply_chat_template(), robust extraction of numeric outputs from model responses, improved reproducibility and compatibility with future Colab updates and support for loading any HF-hosted LLaVA model.

## 🧭 Workflow Overview

SAGAI v1.1 is organized into four Python modules:

![SAGAI Diagram](https://github.com/perezjoan/SAGAI/blob/images/sagai%20diagram.png)

| Module       | Function                                                                                           | Version   | Documentation                              |
|--------------|----------------------------------------------------------------------------------------------------|-----------|---------------------------------------------|
| **Module 1** | Generate points along the OSM street network in a bounding box                                     | v1.0    | [Module 1 Notice](https://github.com/perezjoan/SAGAI/blob/main/NOTICE_MODULE_1.md)  |
| **Module 2** | Download Google Street View images at each point                                                   | v1.0    | [Module 2 Notice](https://github.com/perezjoan/SAGAI/blob/main/NOTICE_MODULE_2.md)  |
| **Module 3** | Use LLaVA-v1.6-Mistral-7B to score the images via prompt                                         | v2.0    | [Module 3 Notice](https://github.com/perezjoan/SAGAI/blob/main/NOTICE_MODULE_3.md)  |
| **Module 4** | Aggregate and map the scores at both point and street levels                                       | v1.0    | [Module 4 Notice](https://github.com/perezjoan/SAGAI/blob/main/NOTICE_MODULE_4.md)  |

<sub>📄 *Each notice includes detailed descriptions, version info, feature lists, and usage instructions for the corresponding module.*</sub>

---

## 🚀 Quick Start

SAGAI runs in **Google Colab**, with no installation required. Each module is a standalone Python script.

1. **Clone the repo** to your Google Drive:
   ```bash
   git clone https://github.com/perezjoan/SAGAI.git
   ```

2. **Open the Colab notebooks** for each module:

   - [Module 1: OSM Point Generator](https://github.com/perezjoan/SAGAI/blob/main/module_1_osm_point_generator.ipynb)   
     Input: 📍 bounding box in WGS84 (latitude/longitude)

   - [Module 2: Street View Batch Downloader](https://github.com/perezjoan/SAGAI/blob/main/module_2_streetview_batch_downloader.ipynb)  
     Input: 🔑 Google Maps API key & 🌍 geographic data from Module 1

   - [Module 3: Scoring/inference with LLaVA](https://github.com/perezjoan/SAGAI/blob/main/module_3_llava_inference_scoring.ipynb)  
     Input:  🧠 select a scoring task (`T1`, `T2`, `T3`) or write your own prompt & 🖼️ images downloaded from Module 2

   - [Module 4: Aggregation & Mapping](https://github.com/perezjoan/SAGAI/blob/main/module_4_aggregation_mapping.ipynb)  
     Input: 🌍 geographic data from Module 1 and 📃 scores from Module 3
3. **Run the full pipeline** to generate visual scores and thematic maps for your city or neighborhood.

Each module is independent and can be customized or reused for different spatial tasks.
> ⚠️ **In case of issues or questions**, refer to the detailed [module notices](#-workflow-overview) for each component. These include setup guidance, version info, and usage tips. A full list of Python packages and versions is provided for Module 3 in [requirements_sagai_module_3_v1-0.txt](https://github.com/perezjoan/SAGAI/blob/main/requirements_sagai_module_3_v1-0.txt)—use it to replicate the working environment or troubleshoot compatibility issues in the future.
---

## 📦 Predefined Scoring Tasks

| Task ID | Type         | Description                                   |
|---------|--------------|-----------------------------------------------|
| `T1`    | Classification | Urban vs Rural Scene (0 = rural, 1 = urban)   |
| `T2`    | Counting      | Number of visible storefronts (0, 1, 2+)         |
| `T3`    | Measurement   | Sidewalk width in meters (e.g. 0, 1.5, 2.0)    |

All prompts are defined in `module_3_llava_inference_scoring.ipynb` and can be fully customized.

---

## 📊 Example Outputs

Two pilot case studies are included in this repository:
- **Nice, France** – A linear urban corridor following the Paillon Valley, characterized by mixed land uses and dense social housing developments in the northern sector.
- **Vienna, Austria** – A heterogeneous peri-urban area in the Penzing-Wolfersberg sector, characterized by low-density housing, allotment gardens, winding roads, and forested patches.

All outputs generated by each module—except Module 2—are included in the GitHub repository as compressed files for easy inspection and reuse.  
Due to Google’s Terms of Service, raw Street View images retrieved in Module 2 are **not** distributed.

However, all other deliverables are available in the following archive:

🔗 [Download Full Output Archive (excluding Street View imagery)](https://github.com/perezjoan/SAGAI/blob/main/output%20nice%20vienna%20SAGAI%20v1-0.zip)

This includes:
- GeoPackages with points, segments, and spatialized scores (Modules 1 and 4)
- CSV scoring outputs (Module 3)
- Thematic maps by point and segment
- Validation patchworks comparing LLaVA predictions with human annotations

Quantitatively, SAGAI v1.0 achieved an overall accuracy of 91.67% for urban–rural classification (T1), 64.17% for storefront detection (T2), and 54.05% for sidewalk width estimation (T3) across the two case studies. For more information regarding validation, please refer to our [publications](#-citation)

Pipeline example applied to Nice: 

![Pipeline example applied to nice](https://github.com/perezjoan/SAGAI/blob/images/PIPELINE%20EXAMPLE%20NICE.png)

---

## 📚 Citation

If you use SAGAI in your research, please cite:

> Perez, J and Fusco, G. (2025) *Streetscape Analysis with Generative AI (SAGAI): Vision-Language Assessment and Mapping of Urban Scenes*. Geomatica, 77(2), 100063, 18p. Available at: https://www.sciencedirect.com/science/article/pii/S1195103625000199

---

## 🪪 License and Attribution

SAGAI is released under the [Apache License 2.0](LICENSE). This allows for use, modification, and redistribution in academic, commercial, and open-source contexts.

Please note the following third-party components:

- [LLaVA v1.6](https://github.com/haotian-liu/LLaVA), used for image-language inference (MIT License)
- [CLIP](https://github.com/openai/CLIP) and [Transformers](https://github.com/huggingface/transformers) for model loading and vision encoding (Apache 2.0 / MIT)
- [OpenStreetMap](https://www.openstreetmap.org/copyright) data (ODbL 1.0 License)
- Google Street View imagery is accessed via API and remains subject to Google’s [Terms of Service](https://maps.google.com/help/terms_maps/).

---

## ✨ Acknowledgments

This research is supported by the [EMC2 project](https://emc2-dut.org/) co-funded by **ANR (France)**, **FFG (Austria)**, **MUR (Italy)**, and **Vinnova (Sweden)** under the **Driving Urban Transition Partnership**, which has been co-funded by the European Commission. SAGAI is designed as an open and extensible tool to support future research initiatives and collaborations in urban analytics and AI-based spatial analysis.

## 🏢 Developer

SAGAI is developed by [Joan Perez](https://orcid.org/0000-0003-3003-0895), founder of **Urban Geo Analytics** — an independent research and consulting practice focused on geospatial modeling, AI for cities, and open-source urban analytics. 🌐 [urbangeoanalytics.com](https://urbangeoanalytics.com/)

---

## 📫 Feedback and Contributions

Feel free to open an issue or pull request. Contributions and forks are welcome!

🔗 [GitHub Discussions](https://github.com/perezjoan/SAGAI/discussions) – Share use cases, ideas, and extensions.

