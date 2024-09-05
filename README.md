# Versatile Head Alignment with Adaptive Appearance Priors

## TL;DR

- A photometric optimization pipeline based on differentiable rasterization, applied to human head alignment.
- A perturbation mechanism that implicitly extract and inject regional appearance priors adaptively during rendering.
- Enabling alignment of regions purely based on their appearance consistency, such as the hair, ears, neck, and shoulders, where no pre-defined landmarks are available.

## License

This work is made available under [CC-BY-NC-SA-4.0](./LICENSE). The repository is derived from the [multi-view head tracker of GaussianAvatars](https://github.com/ShenhanQian/GaussianAvatars/tree/main/reference_tracker), which is subjected to the following statements:

> Toyota Motor Europe NV/SA and its affiliated companies retain all intellectual property and proprietary rights in and to this software and related documentation. Any commercial use, reproduction, disclosure or distribution of this software and related documentation without an express license agreement from Toyota Motor Europe NV/SA is strictly prohibited.

On top of the original repository, we add support to monocular videos and provide a complete set of scripts from video preprocessing to result export for NeRF/3DGS-style applications.

## Setup

```shell
conda create --name VHAP -y python=3.10
conda activate VHAP

# Install CUDA and ninja for compilation
conda install -c "nvidia/label/cuda-12.1.1" cuda-toolkit ninja cmake  # use the right CUDA version
ln -s "$CONDA_PREFIX/lib" "$CONDA_PREFIX/lib64"  # to avoid error "/usr/bin/ld: cannot find -lcudart"
conda env config vars set CUDA_HOME=$CONDA_PREFIX  # for compilation

# Install PyTorch (make sure that the CUDA version matches with "Step 1")
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121
# or
conda install pytorch torchvision pytorch-cuda=12.1 -c pytorch -c nvidia
# make sure torch.cuda.is_available() returns True

pip install -e .
```

## Note

- We use an adjusted version of [nvdiffrast](https://github.com/ShenhanQian/nvdiffrast/tree/backface-culling) for backface-culling. To completely remove previous versions and compiled pytorch extensions, you can execute

  ```shell
  pip uninstall nvdiffrast
  rm -r ~/.cache/torch_extensions/*/nvdiffrast*
  ```

- We use [STAR](https://github.com/ShenhanQian/STAR/) for landmark detection by default. Alterntively, [face-alignment](https://github.com/1adrianb/face-alignment) is faster but less accurate.

## Download

Our code relies on FLAME. Downloaded asset from https://flame.is.tue.mpg.de/download.php and store them in below paths:

- `asset/flame/flame2023.pkl`  # FLAME 2023 (versions w/ jaw rotation)
- `asset/flame/FLAME_masks.pkl`  # FLAME Vertex Masks

> It is possible to use FLAME 2020 by download to `asset/flame/generic_model.pkl`. The `FLAME_MODEL_PATH` in `flame.py` needs to be updated accordingly.

## Usage

### Preprocess

This step extracts frames from video(s), then run foreground matting for each frame, which requires GPU.

#### NeRSemble dataset

```shell
SUBJECT="074"
SEQUENCE="EMO-1"

python vhap/preprocess_video.py \
--input data/nersemble/${SUBJECT}/${SEQUENCE} \
--downsample_scales 2 4 \
--matting_method background_matting_v2
```

- `--downsample_scales 2 4`: Generate downsampled versions of the images in scale 2 and 4.
- `--matting_method background_matting_v2`: Use BackGroundMatingV2 due to availability of background images.

#### Monocular videos

```shell
SEQUENCE="obama.mp4"

python vhap/preprocess_video.py \
--input data/monocular/${SEQUENCE} \
--matting_method robust_video_matting
```

- `--matting_method robust_video_matting`: Use RobustVideoMatting due to lack of a background image.

### Align and track faces

This step automatically detects facial landmarks if absent, then begin FLAME tracking. We initialize shape and appearance parameters on the first frame, then do a sequential tracking of following frames. After the sequence tracking, we conduct 30 epochs of global tracking, which optimize all the parameters on a random frame in each iteration.

#### NeRSemble dataset

```shell
SUBJECT="074"
SEQUENCE="EMO-1"
TRACK_OUTPUT_FOLDER="output/${SUBJECT}_${SEQUENCE}_v16_DS4_wBg_staticOffset"

python vhap/track_nersemble.py --data.root_folder "data/nersemble" \
--exp.output_folder $TRACK_OUTPUT_FOLDER \
--data.subject $SUBJECT --data.sequence $SEQUENCE \
--data.n_downsample_rgb 4
```

#### Monocular videos

```shell
SEQUENCE="bala"
TRACK_OUTPUT_FOLDER="output/${SEQUENCE}_whiteBg_staticOffset"

python vhap/track.py --data.root_folder "data/monocular" \
--exp.output_folder $TRACK_OUTPUT_FOLDER \
--data.sequence $SEQUENCE \
```

Optional arguments

- `--model.no_use_static_offset`: disable static offset for FLAME (very stable, but less aligned facial geometry)

  > Disabling static offset will automatically triggers `--model.occluded hair`, which is crucial to prevent the head from growing too larger to align with the top of hair.

- `--exp.no_photometric`: track only with landmark (very fast, but coarse)

### Export tracking results into a NeRF-style dataset

Given the tracked FLAME parameters from the above step, you can export the results to form a NeRF/3DGS style sequence, consisting of image folders and a `transforms.json`.

#### NeRSemble dataset

```shell
SUBJECT="074"
SEQUENCE="EMO-1"
TRACK_OUTPUT_FOLDER="output/${SUBJECT}_${SEQUENCE}_v16_DS4_wBg_staticOffset"
EXPORT_OUTPUT_FOLDER="export/${SUBJECT}_${SEQUENCE}_v16_DS4_whiteBg_staticOffset_maskBelowLine"

python vhap/export_as_nerf_dataset.py \
--src_folder ${TRACK_OUTPUT_FOLDER} \
--tgt_folder ${EXPORT_OUTPUT_FOLDER} --background-color white
```

#### Monocular videos

```shell
SEQUENCE="bala"
TRACK_OUTPUT_FOLDER="output/${SEQUENCE}_whiteBg_staticOffset"
EXPORT_OUTPUT_FOLDER="export/${SEQUENCE}_whiteBg_staticOffset_maskBelowLine"

python vhap/export_as_nerf_dataset.py \
--src_folder ${TRACK_OUTPUT_FOLDER} \
--tgt_folder ${EXPORT_OUTPUT_FOLDER} --background-color white
```

### Combine exported sequences as a union dataset (for the same person)

TODO

## Cite

Please kindly cite our repository and preceding paper if you find our software or algorithm useful for your research.

```bibtex
@article{qian2024versatile,
  title   = "Versatile Head Alignment with Adaptive Appearance Priors",
  author  = "Qian, Shenhan",
  year    = "2024",
  month   = "September",
  url     = "https://github.com/ShenhanQian/VHAP"
}
```

```bibtex
@inproceedings{qian2024gaussianavatars,
  title={Gaussianavatars: Photorealistic head avatars with rigged 3d gaussians},
  author={Qian, Shenhan and Kirschstein, Tobias and Schoneveld, Liam and Davoli, Davide and Giebenhain, Simon and Nie{\ss}ner, Matthias},
  booktitle={Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition},
  pages={20299--20309},
  year={2024}
}
```
