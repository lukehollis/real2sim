# Real2Sim Pipeline

This repository contains the full pipeline for 3D scene reconstruction and semantic segmentation, integrating **Gaussian Splatting**, **LangSplatV2**, and **SplatGraph**.

## Credits

**Advisement & Guidance:**
This project was developed under the guidance of **Heng Yang**, **Zhutian Chen**, **Wanhua Li**, **Hanspeter Pfister**, and **Alejandro Escontrela**.
Their insights were instrumental in integrating their respective projects into this pipeline. Any merit belongs to them; any shortcomings are my own.


## Methodology & Workflow

This pipeline automates the conversion of real-world video/images into semantic 3D simulation environments. It bridges the gap between raw data collection and downstream robotic tasks.

### 1. Core Methods

**Gaussian Splatting (3DGS):**
We use [3D Gaussian Splatting](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/) as the underlying representation. Unlike NeRFs, 3DGS offers real-time rendering and explicit scene geometry, making it ideal for interactive simulation.

**LangSplatV2:**
To make the scene "understandable" to robots, we integrate [LangSplat](https://langsplat.github.io/). This method lifts CLIP language embeddings from 2D images into the 3D Gaussians. This allows us to query the scene with text (e.g., "chair", "red cup") and get 3D spatial localizations.

**SplatGraph:**
We generate a hierarchical 3D Scene Graph from the varying levels of LangSplat features. This structures the scene into distinct objects with bounding boxes and relationship hierarchies, which is critical for physics simulation.

**GaussGym / Isaac Sim:**
The final output is designed for ingestion into [GaussGym](https://escontrela.me/gauss_gym/) (RL environment) or **Isaac Sim** (Interactive Simulation). These platforms effectively "import" the digitised reality, allowing us to train robot policies in photorealistic twins of the real world.

### 2. High-Level Workflow

The process transforms raw real-world data into a simulation-ready environment in 4 phases:

#### Phase 1: Capture & Data Prep
1.  **Capture with Polycam:** Use the [Polycam](https://poly.cam/) app (iOS/Android) in "Photo Mode" (not LiDAR/Video).
    *   Take **~100 photos** moving slowly around the object/scene in a circle.
    *   **Pro Tip:** Avoid motion blur and keep consistent lighting.
2.  **Export:** From Polycam, export the **Raw Data (Images)**.
    *   *Note:* You can also export the GLTF for reference, but our pipeline builds a higher-quality reconstruction from the raw images.
3.  **Download:** Airdrop/Transfer the zip file to your computer and unzip it.
    *   Place images in `data/<scene_name>/images` (NOT `input`).
4.  **Convert HEIF -> JPG:** (If using iPhone)
    *   Apple devices often export `.HEIC` files. You **must** convert these to `.jpg` or `.png`.
    *   *Mac Tip:* Select all images in Finder -> Right Click -> Quick Actions -> Convert Image -> JPEG.

#### Phase 2: Setup
1.  **Clone:** Recursively clone this repo (`git clone --recursive ...`).
2.  **Install:** Create the two necessary Conda environments (`gaussian_splatting` and `langsplat_v2`) using the provided `environment.yml` files in their respective directories.

#### Phase 3: The Automated Pipeline
Run the master script: `./scripts/run_full_pipeline.sh <scene_path>`.
This script orchestrates the complex handovers between environments:

*   **Step 1: COLMAP (SfM):** *Env: `gaussian_splatting`*
    *   Analyzes the 100 images to find common feature points.
    *   Estimates the camera position for every single photo.
    *   Generates a "sparse point cloud" (a ghostly outline of the scene).
*   **Step 2: Gaussian Splatting Training:** *Env: `gaussian_splatting`*
    *   Takes the sparse points and "splats" 3D Gaussians (blobs) onto them.
    *   Learns the color/shape of these blobs to match your photos perfectly (30k iterations).
*   **Step 3: LangSplat Preprocessing:** *Env: `langsplat_v2`*
    *   Uses **SAM (Segment Anything Model)** to cut out objects in 2D.
    *   Uses **CLIP** to give those cutouts a "meaning" (embedding).
*   **Step 4: LangSplat Training:** *Env: `langsplat_v2`*
    *   Teaches the 3D Gaussians to carry these "meanings". Now a blob isn't just "red", it's "part of a chair".
*   **Step 5: LangSplat Rendering (Optional):** *Env: `langsplat_v2`*
    *   (Disabled by default) Renders qualitative images of the semantic field.
*   **Step 6: SplatGraph Generation:** *Env: `langsplat_v2`*
    *   Groups these semantic blobs into distinct 3D objects (e.g., "Chair 1", "Table").
    *   Calculates 3D Bounding Boxes (OBB) for physics simulation.


#### Phase 4: Import to GaussGym
To bring your reconstructed scene into the RL/Simulation environment, we need to structure the data and generate navigation meshes. We have automated this process:

1.  **Run the Import Script:**
    ```bash
    ./scripts/import_splatgraph_to_gaussgym.sh <scene_name> <path_to_data> <target_gauss_gym_path>
    # Example:
    # ./scripts/import_splatgraph_to_gaussgym.sh ramen ./data/ramen ./data/ramen/gauss_gym
    ```
    This script will:
    *   Link the Point Cloud and Language Features.
    *   Convert camera metadata to standard `transforms.json`.
    *   Generate a `slice_0000.npz` mesh for robot navigation.

#### Phase 5: Simulation
1.  **Visualize (Optional):** Use `visualize.py` to check if "Chair" actually highlights the chair.
2.  **Run GaussGym:**
    ```bash
    gauss_play --task=t1 --env.num_envs 1 \
        --terrain.scenes.iphone_data.repo_id=local:/absolute/path/to/data/ramen/gauss_gym \
        --terrain.scenes.grand_tour.split_frac=0.0 \
        --terrain.scenes.arkit.split_frac=0.0
    ```

## Troubleshooting & Common Failure Modes

### CUDA Out of Memory (OOM)
The most common failure is running out of VRAM (CUDA Out of Memory) during the Gaussian Splatting or LangSplat training steps. This typically happens if the dataset is too large for your GPU.

**Symptoms:**
*   Pipeline crashes with `RuntimeError: CUDA out of memory`.
*   System becomes unresponsive during training.

**Solutions:**
1.  **Reduce Image Count:** Ensure you are using **fewer than 300 images**.
    *   *Fix:* Delete every 2nd or 3rd image in your `images/` folder to reduce the count (e.g., aim for 100-150).
2.  **Downscale Images:** High-resolution images (4K+) consume significantly more VRAM.
    *   *Fix:* Resize your input images to roughly 1920x1080 or 1600x1200 before running the pipeline.
3.  **Close Other Apps:** Ensure no other GPU-intensive applications (games, other training runs, Viser) are running.

### COLMAP Failures
If COLMAP fails to register images (creating a sparse point cloud with very few or zero points):
*   **Cause:** Images lack texture or overlap.
*   **Fix:** Ensure your photos have at least 60-70% overlap. Add texture to the scene (place the object on a patterned rug/newspaper) if the background is a plain white wall.

## Getting Started

For a complete "From Zero to Hero" guide on how to setup, capture data, and run the pipeline, please see the [Scripts Documentation](scripts/README.md).
