
# SCENE ASSESSMENT WITH LLaVA

**Version**: 2.0 
**Project Github**: [https://github.com/perezjoan/SAGAI](https://github.com/perezjoan/SAGAI)

## 1. Description

Module 3 performs automated semantic scoring of street-level images using LLaVA-Next v1.6 (Mistral-7B backbone) in a fully zero-shot manner.
Unlike the original v1.0 implementation—which relied on cloning the evolving LLaVA GitHub repository and manually stitching model components—v2.0 uses the official Hugging Face LLaVA-Next implementation exclusively, ensuring stability and long-term compatibility. This module loads a vision-language model directly from Hugging Face, formats prompts using the model’s built-in `apply_chat_template`, performs inference on each image, extracts a numerical score, and writes results to a resumable CSV. Three predefined tasks are included: Categorization (e.g., rural vs. urban), Counting (e.g., number of visible shops), and Measuring (e.g., estimated sidewalk width in meters). Only one task is active at a time, determined by setting a `selected_task` variable (T1, T2, or T3). Users are encouraged to write their own scoring logic and prompts by modifying the prompt templates provided in the script. The results are saved into a spreadsheet-style .csv. A resuming mechanism ensures that previously processed images are skipped automatically in case the session is interrupted.

## 2. Key Features

### Google Drive Integration
- Automatically mounts Google Drive to access your image folder and save outputs.
- Works with `.jpg`, `.jpeg`, and `.png` files.
- Skips any image ending in `_NA.jpg`

### Hugging Face–Native LLaVA Loading
- Uses `llava-hf/llava-v1.6-mistral-7b-hf`
- Runs with 4-bit quantization for memory-efficient inference in Colab GPU environments but can be switched to 8-bit
- Compatible with any other LLavaNext model by changing the model ID

### Numerical Scoring Pipeline
- Ideal for tasks that require simple scalar outputs per image (e.g., 0 = no shop, 1 = one shop, 2 = multiple shops).
- You define the scoring rules in natural language. Prompts are editable and fully customizable for different use cases.
- Output is written as a 2-column CSV: image name and score.

### Pre-coded AI Scoring Task
- **Categorization (T1)**: Classifies each image as rural (0) or urban (1).
- **Counting (T2)**: Detects presence of storefronts — 0 (none), 1 (one), or 2 (multiple).
- **Measuring (T3)**: Estimates visible sidewalk width in meters, rounded to the nearest 0.5 (e.g., 2.0, 2.5).
- Selection is made via the `selected_task` parameter. Prompts are editable and fully customizable.

### Resumable Execution
- If the output CSV already exists and is not empty, the script resumes from the last processed image.
- Images already present in the CSV will be skipped.
- If the output CSV does not exist or is empty, the script starts fresh.

### Export to CSV
- The scores are automatically saved in a .csv file with the point identifiers

### Colab-Ready and Modular
- No local setup needed—the entire pipeline runs from Google Colab.
- Outputs can feed directly into geospatial or statistical analysis scripts.

## 3. How to Use

Open the notebook in Google Colab and ensure you’ve selected a GPU runtime (`Runtime → Change runtime type → Hardware accelerator → GPU`). Mount your Drive using `drive.mount('/content/drive')`, then define:  

- `case_study`: a short name for the project (e.g., `"vienna"` or `"nice"`).  
- `selected_task`: choose from `"T1"`, `"T2"`, or `"T3"` to select the desired AI behavior.  

Output paths and folders will be created automatically based on the case name. In the AI Settings section, one of three pre-coded prompts will be activated based on the `selected_task` value. You can customize any of them or insert your own logic for different visual assessments. The model will loop over all valid images, apply the scoring rubric, and write results to the `.csv` file.

## 4. Additional Information

Adding your Hugging Face token in Colab Notebook Secrets is optional, but it significantly speeds up model downloads and prevents rate-limit slowdowns. Simply store your token as HF_TOKEN in Colab settings, and SAGAI will use it automatically during model loading.

The model is loaded in 4-bit quantization using Hugging Face’s built-in `load_in_4bit=True`, which keeps memory usage low enough for most Colab GPUs. You can switch to 8-bit by using `load_in_8bit=True` instead, or remove both parameters to load the model in full precision.

If your images were generated using the companion script STREET VIEW BATCH DOWNLOADER, the script will automatically ignore any images with `_NA` in the filename—these indicate locations where Google Street View returned no visual data.
