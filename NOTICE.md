# SAGAI v2.2 — Function Reference

Full documentation of every function and callback in `SAGAI.ipynb`, organized by block.

---

## Block 0 — Resume an Existing Case Study

Run this block **instead of** Blocks 1–2 when data from a previous session already exists on Google Drive. It mounts Drive, scans for existing case studies, and sets the global `case_study_name` variable so that downstream blocks work without re-running the full pipeline.

### `_on_resume(b)`

**Callback** triggered by the "Resume this case study" button.

- Sets the global `case_study_name` from the dropdown selection.
- Checks for the existence of three data artifacts on Drive:
  - **GeoPackage** (`StreetSamples/{name}_osm.gpkg`) — points and streets from Block 1.
  - **Image folder** (`StreetViewBatchDownload_{Name}/`) — JPGs from Block 2.
  - **Analysis CSV** (`Score_Analysis_*.csv`) — scores from Block 5.
- Prints a diagnostic report indicating which artifacts are present and which blocks can be skipped:
  - Points + images + CSV exist → skip to Block 6.
  - Points + images exist → skip to Block 3.
  - Points only → skip to Block 2.

**Global variables set:** `case_study_name` (str).

---

## Block 1 — OSM Point Generator (Interactive Map)

Generates evenly spaced sampling points along the OpenStreetMap pedestrian street network within a user-defined study area. The study area can be drawn on an interactive map or loaded from a file. The projected CRS is auto-detected (UTM zone) for worldwide use.

### `_update_status(geom, source="drawn")`

Updates the status widget with geometry info (type, bounding box, approximate area in km²). Stores the geometry in the `DRAWN_GEOMETRY` dictionary.

| Parameter | Type | Description |
|-----------|------|-------------|
| `geom` | Shapely geometry | The polygon geometry |
| `source` | str | Label shown in status ("Drawn" or "Loaded") |

### `handle_draw(self, action, geo_json)`

**Callback** for the ipyleaflet `DrawControl`. Captures the drawn rectangle or polygon from the map, converts it to a Shapely geometry, and calls `_update_status`.

### `on_load_polygon(b)`

**Callback** for the "Load Polygon" button. Reads a vector file (GeoPackage, GeoJSON, Shapefile) from the path entered in the text widget. Reprojects to WGS84 if needed, unions all geometries into a single polygon, displays it on the map, and calls `_update_status`.

### `on_save_polygon(b)`

**Callback** for the "Save Study Area" button. Saves the current polygon (drawn or loaded) to `StreetSamples/{case_study}_study_area.gpkg` on Drive for reproducibility.

### `convert_list_to_string(val)`

Converts list-type values in OSM edge attributes to comma-separated strings. Used because OSMnx sometimes returns lists for attributes like `highway` or `name` when multiple tags apply.

| Parameter | Type | Description |
|-----------|------|-------------|
| `val` | any | The attribute value |
| **Returns** | str or original type | Comma-joined string if input is a list |

### `run_block1(b)`

**Main callback** for the "Generate Points" button. Executes the full pipeline:

1. Reads configuration from widgets (case study name, spacing, offset, highway filter).
2. Auto-detects the projected CRS (UTM zone) from the study area centroid.
3. Downloads the OSM street network via Overpass API (`ox.graph_from_polygon`).
4. Converts the graph to a GeoDataFrame of edges.
5. Applies the highway type filter if selected (removes footways, cycleways, etc.).
6. Reprojects to the projected CRS for metric distance calculations.
7. Removes duplicate edges by `osmid` + `length_m`.
8. Generates sampling points along each street at the configured spacing, with offset from street endpoints.
9. Computes WGS84 lat/lon for each point.
10. Saves both layers (streets, points) to a GeoPackage on Drive.
11. Plots a preview map and prints a summary.

**Global variables set:** `points_gdf`, `streets_gdf`, `save_path_gpkg`, `case_study_name`.

**Widget parameters:**

| Widget | Default | Description |
|--------|---------|-------------|
| Case study | — | Name used for all file paths |
| Spacing | 40 m | Distance between consecutive points along a street |
| Offset | 15 m | Minimum distance from street endpoints |
| Highway types | All | OSM highway tags to include |

---

## Block 2 — Street View Batch Downloader

Downloads Google Street View images at each sampling point. The pipeline computes canonical bearings, queries the free Metadata API for each point, filters out indoor/contributor photos, snaps points to valid panorama locations, removes duplicates, and downloads images only for surviving unique points.

### `canonical_bearing(raw_bearing)`

Normalizes a bearing to the range [0, 180). A street digitized south-to-north (bearing ~0°) and north-to-south (bearing ~180°) both become ~0°. This ensures "left" and "right" are geographically consistent regardless of OSM digitization direction.

| Parameter | Type | Description |
|-----------|------|-------------|
| `raw_bearing` | float | Bearing in degrees [0, 360) |
| **Returns** | float | Canonical bearing [0, 180) |

### `subdivide_segment(line_geom, max_length=50.0)`

Splits a LineString into sub-segments of approximately equal length, each no longer than `max_length`. Used to compute local bearings on winding roads.

| Parameter | Type | Description |
|-----------|------|-------------|
| `line_geom` | Shapely LineString | The street geometry |
| `max_length` | float | Maximum sub-segment length in meters |
| **Returns** | list[LineString] | List of sub-segment geometries |

### `node_to_node_bearing(line_geom)`

Computes the raw bearing from the first to the last coordinate of a LineString. Used on sub-segments where intermediate vertices can be ignored.

| Parameter | Type | Description |
|-----------|------|-------------|
| `line_geom` | Shapely LineString | A line segment |
| **Returns** | float | Bearing in degrees [0, 360) |

### `find_sub_segment_for_point(point_geom, sub_segments)`

Finds the sub-segment closest to a given point. Used to assign the correct local bearing when a street has been subdivided.

| Parameter | Type | Description |
|-----------|------|-------------|
| `point_geom` | Shapely Point | The sampling point |
| `sub_segments` | list[LineString] | Sub-segments from `subdivide_segment` |
| **Returns** | LineString | The closest sub-segment |

### `compute_bearing_for_point(point_geom, street_geom, max_sub_length=50.0)`

Combines `subdivide_segment`, `find_sub_segment_for_point`, `node_to_node_bearing`, and `canonical_bearing` into a single call. Returns the canonical bearing for a point on a given street.

| Parameter | Type | Description |
|-----------|------|-------------|
| `point_geom` | Shapely Point | The sampling point (projected CRS) |
| `street_geom` | Shapely LineString | The street geometry (projected CRS) |
| `max_sub_length` | float | Max sub-segment length in meters |
| **Returns** | float | Canonical bearing [0, 180) |

### `get_headings_for_mode(mode, bearing=0.0)`

Returns a list of `(heading, suffix)` tuples for the selected image mode. Headings are computed relative to the canonical bearing.

| Mode | Headings returned |
|------|-------------------|
| `4_views` | left, right, front, back |
| `left_right` | left, right |
| `front_back` | front, back |

Left = bearing − 90°, right = bearing + 90°, front = bearing, back = bearing + 180°.

### `query_sv_metadata(lat, lon, api_key, radius=50)`

Queries the Google Street View Metadata API (free, no image cost) for a given location. Returns the panorama ID, actual camera coordinates, copyright string, and status.

| Parameter | Type | Description |
|-----------|------|-------------|
| `lat`, `lon` | float | Query coordinates |
| `api_key` | str | Google Maps API key |
| `radius` | int | Search radius in meters |
| **Returns** | dict | Keys: `status`, `pano_id`, `lat`, `lon`, `copyright` |

### `download_streetview(lat, lon, heading, pitch, fov, save_path, filename, api_key, pano_id=None)`

Downloads a single Street View image. If a `pano_id` is provided, uses it instead of lat/lon for exact panorama targeting. Performs blank image detection: if >90% of pixels are a single color, the image is flagged with a `_NA` suffix.

| Parameter | Type | Description |
|-----------|------|-------------|
| `pano_id` | str or None | Panorama ID from metadata query |
| `pitch` | int | Camera pitch in degrees |
| `fov` | int | Field of view in degrees |
| **Returns** | bool | True if image is blank |

### `plot_coverage_map(gpkg_path, download_dir, case_study, mode, n_per_point)`

Generates a static matplotlib coverage map after download. Points are colored green (images present) or red (no images). Prints coverage statistics.

### `run_block2(b)`

**Main callback** for the "Download Images" button. Executes three phases:

1. **Phase 1 — Bearings**: Computes canonical bearings on the original on-street points (before snapping) using `compute_bearing_for_point`.
2. **Phase 2 — Metadata + Filter + Snap + Dedup**: Queries the Metadata API for each point, filters out non-OK responses, removes indoor/contributor 360° photos (copyright not containing "Google"), computes snap distance via Haversine, removes points beyond the snap threshold, and deduplicates by `pano_id`. Updates the GeoPackage with snapped coordinates, bearings, and panorama IDs.
3. **Phase 3 — Download**: Downloads images for surviving points using `download_streetview`. Skips files that already exist (resume-safe). Displays a progress bar with ETA.

**Global variables set:** `download_dir`, `image_mode_used`.

**Widget parameters:**

| Widget | Default | Description |
|--------|---------|-------------|
| API Key | — | Google Maps API key |
| Image mode | 4 views | Which directions to download |
| Pitch | −20° | Camera pitch angle |
| FOV | 90° | Field of view |
| Sub-seg | 50 m | Max sub-segment length for bearing computation |
| Snap max | 30 m | Max distance between generated point and camera |

---

## Block 3 — Install UVLM + Load VLM

Installs the UVLM package from GitHub (once per session) and loads a vision-language model into GPU memory. All model loading is delegated to `uvlm.load_model()`.

### `on_load(b)`

**Callback** for the "Load model" button. Calls `uvlm.load_model()` with the selected model name, precision, device placement, and optional Hugging Face token. Stores the returned context dictionary in the global `model_ctx`.

**Global variables set:** `model_ctx` (dict) — contains the loaded model, processor, backend identifier, GPU name, load time, and Qwen pixel settings if applicable.

**Widget parameters:**

| Widget | Default | Description |
|--------|---------|-------------|
| Model | Qwen2.5-VL 7B Instruct | Model checkpoint to load |
| Precision | 4-bit | Quantization level (4bit / 8bit / fp16) |
| Placement | Auto | Device mapping strategy |
| low_cpu_mem_usage | True | Reduce CPU memory during loading |
| Use HF token | True | Read token from Colab secrets |

---

## Block 4 — Drive + Task Configuration

Mounts Google Drive, configures scoring tasks via a widget-based prompt builder, and sets generation parameters. The image folder path is pre-filled from Block 1.

### `build_task_widgets(n)`

Dynamically creates `n` task configuration panels, each containing widgets for task type, column name, task prompt, theory, format instruction, advanced reasoning toggle, consensus validation toggle, consensus runs, and numeric tolerance.

When the advanced reasoning checkbox is toggled on, the format widget is visually disabled (greyed out) and a preview of the auto-generated format instruction is shown below it.

### `apply_settings(b)`

**Callback** for the "Apply" button. Reads all widget values and sets the global configuration:

1. Validates the image folder path on Drive.
2. Constructs the output CSV path (named after the model backend).
3. If the model is Qwen and pixel settings have changed, reloads the processor.
4. For each task, resolves the format instruction (see Chain-of-Thought below).
5. Builds the `task_specs` list by calling `uvlm.prompts.build_prompt()` for each task, combining the shared role instruction with the per-task prompt, theory, and format strings.
6. Sets generation parameters (max tokens, temperature, top-p, seed).

**Global variables set:** `image_folder`, `output_path`, `task_specs`, `max_new_tokens`, `do_sample`, `temperature`, `top_p`, `generation_seed`, `display_images`.

**Task spec fields:**

| Field | Type | Description |
|-------|------|-------------|
| `column` | str | CSV column name for this task's scores |
| `prompt` | str | Full prompt (role + task + theory + format) |
| `task_type` | str | `numeric`, `category`, `boolean`, or `text` |
| `advanced_reasoning` | bool | Enable chain-of-thought reasoning |
| `consensus_enabled` | bool | Enable majority voting |
| `consensus_runs` | int | Number of repeated inferences (2–5) |
| `numeric_tolerance` | float | Tolerance percentage for numeric consensus |

### Chain-of-Thought (CoT) Reasoning

When a task has `advanced_reasoning` enabled, two things happen in Block 4 and Block 5:

**1. Format override (Block 4 — `apply_settings`)**

The user's format instruction is replaced with a task-type-specific template from `ADVANCED_REASONING_FORMATS` (defined in `uvlm.prompts`). The user does not need to write anything in the format field. The override is applied before `build_prompt` is called:

```python
if adv_w.value and task_type_w.value != "text":
    fmt = ADVANCED_REASONING_FORMATS.get(task_type_w.value, fmt)
```

The injected format instructions are:

| Task type | Format injected |
|-----------|----------------|
| `numeric` | `"First, describe what you observe and explain your reasoning step by step. Then, on the last line, write only: ANSWER: <integer>"` |
| `boolean` | `"First, describe what you observe and explain your reasoning step by step. Then, on the last line, write only: ANSWER: <yes or no>"` |
| `category` | `"First, describe what you observe and explain your reasoning step by step. Then, on the last line, write only: ANSWER: <category number>"` |

The `text` task type does not support advanced reasoning (it is automatically disabled).

This format is critical because it instructs the model to write `ANSWER: <value>` on its last line, which is the tag that the UVLM response parser looks for.

**2. Token budget override (Block 5 — `uvlm.batch.run_batch`)**

When `advanced_reasoning` is True, the `max_new_tokens` parameter is overridden by `ADVANCED_REASONING_MAX_TOKENS` (1024, defined in `uvlm.prompts`). This gives the model enough generation space for step-by-step reasoning before writing the final answer. The user-configured `max_new_tokens` value is ignored for CoT tasks.

**3. Response parsing (Block 5 — `uvlm.parsers.parse_advanced_reasoning_response`)**

The parser scans the last 5 lines of the model's response for a regex match on `ANSWER: <value>` (case-insensitive). When found:

- Everything above the `ANSWER:` line is stored as the reasoning trace (saved in the `{column}_reasoning` CSV column).
- The value after `ANSWER:` is passed through the task-type-specific parser (`parse_numeric`, `parse_boolean`, or `parse_category`) to extract the final answer.

If the `ANSWER:` tag is not found (e.g., the model ignored the instruction), the parser falls back to running the standard parser on the entire raw response. For numeric tasks this extracts the last number in the text; for boolean tasks it scans for yes/no keywords; for category tasks it takes the first line. This fallback is unreliable and should not be depended on.

**Summary of the CoT data flow:**

```
User prompt (role + task + theory)
      + auto-injected format ("...ANSWER: <integer>")
            ↓
    VLM generates up to 1024 tokens
            ↓
    "I observe 3 cars at ~4.5m each...
     Total = 3 × 4.5 = 13.5 meters.
     ANSWER: 14"
            ↓
    Parser splits on "ANSWER:"
            ↓
    reasoning = "I observe 3 cars..."  →  {col}_reasoning column
    answer = "14"                      →  parse_numeric("14") → "14"
    raw = full response                →  {col}_raw column
```

---

## Block 5 — Run Analysis (UVLM)

Processes all images in the configured folder using the UVLM batch engine. This block contains no custom functions; it calls `uvlm.batch.run_batch()` directly.

### `uvlm.batch.run_batch(...)`

Iterates over all images in `image_folder`, runs inference for each task in `task_specs`, parses responses, handles consensus voting and truncation detection, and saves results to `output_path` with resume support (checkpoint every 3 images).

For each image and each task, the batch engine follows one of three code paths:

1. **Standard mode** (`advanced_reasoning=False`, `consensus_enabled=False`): runs inference once with the user-configured `max_new_tokens`, parses via `parse_response`.
2. **CoT mode** (`advanced_reasoning=True`): runs inference with `ADVANCED_REASONING_MAX_TOKENS` (1024), parses via `parse_advanced_reasoning_response`, stores reasoning trace.
3. **Consensus mode** (`consensus_enabled=True`): runs inference `consensus_runs` times, applies majority voting via `compute_consensus`. Can be combined with CoT.

**CSV columns produced per task:**

| Column | Always | CoT only | Consensus only |
|--------|--------|----------|----------------|
| `{col}` | Parsed answer | Parsed answer | Consensus answer |
| `{col}_raw` | Full raw response | Full raw response | First run's raw |
| `{col}_truncated` | YES/NO | YES/NO | YES/NO |
| `{col}_reasoning` | — | Reasoning trace | — |
| `{col}_consensus` | — | — | YES/NO |
| `{col}_agreement` | — | — | Agreement ratio |
| `{col}_runs` | — | — | JSON array of all values |

**Requires:** `model_ctx` (Block 3), `task_specs` (Block 4).

---

## Block 6 — Aggregation & Mapping

Aggregates image-level scores to point and street levels, generates static matplotlib maps and optional interactive Folium HTML maps. Outputs are saved as a GeoPackage on Drive.

### `detect_image_mode(csv_df)`

Inspects the `image_name` column to determine which image mode was used during download.

| Parameter | Type | Description |
|-----------|------|-------------|
| `csv_df` | DataFrame | The scores CSV |
| **Returns** | tuple(str, int) | Mode name and number of images per point |

Returns `('4_views', 4)`, `('left_right', 2)`, `('front_back', 2)`, or `('unknown', n)`.

### `extract_base_id(image_name)`

Extracts the point identifier from an image filename by removing the file extension, the `_NA` suffix, and the view suffix.

| Example input | Output |
|---------------|--------|
| `point_42_left.jpg` | `point_42` |
| `point_42_right_NA.jpg` | `point_42` |

### `extract_view_suffix(image_name)`

Extracts the view direction suffix from an image filename.

| Example input | Output |
|---------------|--------|
| `point_42_left.jpg` | `left` |
| `point_42_front_NA.jpg` | `front` |

### `score_to_hex(score, vmin, vmax, cmap_name)`

Converts a numeric score to a hex color string using a matplotlib colormap. Returns `#cccccc` (grey) for missing values.

| Parameter | Type | Description |
|-----------|------|-------------|
| `score` | float or NaN | The score value |
| `vmin`, `vmax` | float | Colormap range |
| `cmap_name` | str | Matplotlib colormap name |
| **Returns** | str | Hex color string (e.g., `#2a7b3f`) |

### `build_points_map(points_gdf_display, streets_gdf_display, scores_df, score_col_name, pt_col, download_dir, case_study, cmap_name)`

Builds an interactive Folium map with one `CircleMarker` per point. Each marker is colored by its aggregated score. Clicking a marker opens a popup showing the point ID, aggregated score, street ID, coordinates, and thumbnail images (base64-encoded) with their individual scores.

| Parameter | Type | Description |
|-----------|------|-------------|
| `points_gdf_display` | GeoDataFrame | Points layer with aggregated scores |
| `streets_gdf_display` | GeoDataFrame | Streets layer (drawn as background) |
| `scores_df` | DataFrame | Image-level scores (for popup thumbnails) |
| `download_dir` | str | Path to the image folder |
| **Returns** | folium.Map | The interactive map object |

### `build_streets_map(streets_gdf_display, st_col, case_study, cmap_name)`

Builds an interactive Folium map with street segments colored by their aggregated score. Each segment has a popup showing the street ID, name, highway type, and score.

| Parameter | Type | Description |
|-----------|------|-------------|
| `streets_gdf_display` | GeoDataFrame | Streets layer with aggregated scores |
| `st_col` | str | Column name for the street-level score |
| **Returns** | folium.Map | The interactive map object |

### `on_build_points_map(b)` / `on_build_streets_map(b)`

**Callbacks** for the "Build Points HTML Map" and "Build Streets HTML Map" buttons. Read the stored state from `_block6_state`, call the corresponding map builder, and save the result as an HTML file to `Spatial_Results/` on Drive.

### `run_block6(b)`

**Main callback** for the "Aggregate & Map" button. Executes the full aggregation pipeline:

1. Reads the GeoPackage (points + streets) and the scores CSV.
2. Detects the image mode from filename suffixes.
3. Applies the view direction filter.
4. Aggregates scores to point level (mean, sum, median, variance).
5. Aggregates point-level scores to street level.
6. Generates static matplotlib maps (points and streets side by side).
7. Saves the joined GeoPackage to `Spatial_Results/`.
8. Stores internal state for optional interactive map generation.

**Widget parameters:**

| Widget | Default | Description |
|--------|---------|-------------|
| Score column | `task_1` | CSV column to aggregate and map |
| Aggregation | Average | Aggregation method (mean / sum / median) |
| View filter | All | Which view directions to include |
| Color ramp | Viridis | Colormap for both static and interactive maps |

---

## Global Variable Flow

The following global variables are passed between blocks:

| Variable | Set by | Used by | Description |
|----------|--------|---------|-------------|
| `case_study_name` | Block 0 or Block 1 | Blocks 2, 4, 6 | Name of the case study |
| `points_gdf` | Block 1 | — | Points GeoDataFrame (in-memory) |
| `streets_gdf` | Block 1 | — | Streets GeoDataFrame (in-memory) |
| `save_path_gpkg` | Block 1 | — | Path to saved GeoPackage |
| `download_dir` | Block 2 | — | Path to downloaded images |
| `image_mode_used` | Block 2 | — | Image mode selected |
| `model_ctx` | Block 3 | Blocks 4, 5 | Loaded model context |
| `task_specs` | Block 4 | Block 5 | List of task configurations |
| `image_folder` | Block 4 | Block 5 | Path to image folder |
| `output_path` | Block 4 | Block 5 | Path to output CSV |
| `max_new_tokens` | Block 4 | Block 5 | Generation token limit |
| `do_sample` | Block 4 | Block 5 | Sampling toggle |
| `temperature` | Block 4 | Block 5 | Sampling temperature |
| `top_p` | Block 4 | Block 5 | Nucleus sampling threshold |
| `generation_seed` | Block 4 | Block 5 | Fixed seed or None |
| `display_images` | Block 4 | Block 5 | Show images during inference |

---

## File Structure on Google Drive

```
MyDrive/
└── SAGAI/
    ├── StreetSamples/
    │   ├── {case_study}_osm.gpkg        ← points + streets (Block 1)
    │   └── {case_study}_study_area.gpkg ← study area polygon (Block 1)
    ├── StreetViewBatchDownload_{Case_study}/
    │   ├── point_1_left.jpg             ← images (Block 2)
    │   ├── point_1_right.jpg
    │   ├── point_1_left_NA.jpg          ← blank image
    │   └── Score_Analysis_{backend}.csv ← scores (Block 5)
    └── Spatial_Results/
        ├── {case_study}_osm_joined.gpkg ← joined spatial data (Block 6)
        ├── {case_study}_points_map.html ← interactive map (Block 6)
        └── {case_study}_streets_map.html
```

---

## Dependencies

Core dependencies are installed automatically by the notebook:

| Package | Installed by | Purpose |
|---------|-------------|---------|
| `osmnx` | Block 1 | OpenStreetMap network extraction |
| `ipyleaflet` | Block 1 | Interactive map widget |
| `uvlm` | Block 3 | VLM inference engine (from GitHub) |
| `folium` | Block 6 | Interactive HTML maps |

All other dependencies (`geopandas`, `pandas`, `numpy`, `matplotlib`, `transformers`, `torch`, `Pillow`, `requests`, `shapely`) are pre-installed in Google Colab.

---

## License

SAGAI is released under the [Apache License 2.0](LICENSE).

## Citation

Perez, J. and Fusco, G. (2025). *Streetscape Analysis with Generative AI (SAGAI): Vision-Language Assessment and Mapping of Urban Scenes.* Geomatica, 77(2), 100063.

Perez, J. and Fusco, G. (2026). *UVLM: A Universal Vision-Language Model Loader for Reproducible Multimodal Benchmarking.* arXiv:2603.13893.
