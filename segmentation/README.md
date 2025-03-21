# KRoad 2D Semantic Segmentation with InternImage

This folder contains the implementation of the InternImage for semantic segmentation. 

Our segmentation code is developed on top of [MMSegmentation v0.27.0](https://github.com/open-mmlab/mmsegmentation/tree/v0.27.0).

## Usage

### Install

- Clone this repo:

```bash
git clone https://github.com/gawon1224/2024-Autonomous-Driving-AI-Challenge.git
cd 2024-Autonomous-Driving-AI-Challenge/segmentation
```

- Create a conda virtual environment and activate it:

```bash
conda create -n kroad python=3.7 -y
conda activate kroad
```

- Install `CUDA>=10.2` with `cudnn>=7` following
  the [official installation instructions](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html)
- Install `PyTorch>=1.10.0` and `torchvision>=0.9.0` with `CUDA>=10.2`:

For examples, to install torch==1.11 with CUDA==11.3 and nvcc:
```bash
conda install pytorch==1.11.0 torchvision==0.12.0 torchaudio==0.11.0 cudatoolkit=11.3 -c pytorch -y
conda install -c conda-forge cudatoolkit-dev=11.3 -y # to install nvcc
```

- Install other requirements:

  note: conda opencv will break torchvision as not to support GPU, so we need to install opencv using pip. 	  

```bash
conda install -c conda-forge termcolor yacs pyyaml scipy pip -y
pip install opencv-python
```

- Install `timm` and `mmcv-full` and `mmsegmentation':

```bash
pip install -U openmim
mim install mmcv-full==1.5.0
mim install mmsegmentation==0.27.0
pip install timm==0.6.11 mmdet==2.28.1
```

- Compile CUDA operators
```bash
cd ./ops_dcnv3
./make.sh
# unit test (should see all checking is True)
python test.py
```
- You can also install the operator using .whl files
[DCNv3-1.0-whl](https://github.com/OpenGVLab/InternImage/releases/tag/whl_files)

- Lastly, you should edit `/home/../anaconda3/envs/kroad/lib/python3.8/site-packages/mmcv/utils/config.py`, line 502
	
	Delete `verify=True` in `text, _ = FormatCode(text, style_config=yapf_style, verify=True)`

### Data Preparation

Prepare datasets according to the [guidelines](https://github.com/open-mmlab/mmsegmentation/blob/master/docs/en/dataset_prepare.md#prepare-datasets) in MMSegmentation.


### Evaluation

To evaluate our `InternImage` on validation dataset, run:

```bash
./dist_test.sh <config-file> <checkpoint> <gpu-num> --eval mIoU
```
You can download our checkpoint file from [here](https://drive.google.com/file/d/1GvMmIy9DAF6dWNmtfTtmw4DkeTvOmuWn/view?usp=sharing). Then place it to `segmentation/checkpoint_dir/`.

For example, to evaluate the `InternImage-B` with a single GPU:

```bash
python test.py configs/kroad/upernet_internimage_b_512x1024_320k_city2kroad.py checkpoints/best.pth --eval mIoU
```

For example, to evaluate the `InternImage-B` with a single node with 8 GPUs:

```bash
./dist_test.sh configs/kroad/upernet_internimage_b_512x1024_320k_city2kroad.py checkpoints/best.pth 8 --eval mIoU
```

### Training

To train an `InternImage` on KRoad dataset, run:

```bash
./dist_train.sh <config-file> <gpu-num>
```

For example, to train `InternImage-B` with 8 GPU on 1 node (total batch size 16), run:

```bash
./dist_train.sh configs/kroad/upernet_internimage_b_512x1024_320k_city2kroad.py 8
```
- If you resume learning from a specific point, add `resume_from = 'checkpoint_path.pth'` in a config file.
- You can download checkpoint files what you need from [here](https://huggingface.co/OpenGVLab/InternImage/tree/fc1e4e7e01c3e7a39a3875bdebb6577a7256ff91). Then place it to `segmentation/checkpoint_dir/` and add `load_from='checkpoing_path.pth'` in config files. 
	- If you don't do this, you can't use a model pretrained with **CITYSCAPES**, just pretrained with **IMAGENET**.
### Manage Jobs with Slurm

For example, to train `InternImage-B` with 8 GPU on 1 node (total batch size 16), run:

```bash
GPUS=8 ./slurm_train.sh <partition> <job-name> configs/kroad/upernet_internimage_b_512x1024_320k_city2kroad.py
```

### Image Demo
To inference a single/multiple image like this.
If you specify image containing directory instead of a single image, it will process all the images in the directory.:
```
CUDA_VISIBLE_DEVICES=0 python image_demo.py \
  data/kroad/images/validation/E_DCG_230829_141_FC_080.jpg \
  configs/kroad/upernet_internimage_b_512x1024_160k_city2kroad.py  \
  checkpoints/best.pth  \
  --palette kroad 
```

### Export

To export a segmentation model from PyTorch to TensorRT, run:
```shell
MODEL="model_name"
CKPT_PATH="/path/to/model/ckpt.pth"

python deploy.py \
    "./deploy/configs/mmseg/segmentation_tensorrt_static-512x512.py" \
    "./configs/kroad/${MODEL}.py" \
    "${CKPT_PATH}" \
    "./deploy/demo.png" \
    --work-dir "./work_dirs/mmseg/${MODEL}" \
    --device cuda \
    --dump-info
```

For example, to export `upernet_internimage_b_512x1024_320k_cityscapes2kroad` from PyTorch to TensorRT, run:
```shell
MODEL="upernet_internimage_b_512x1024_160k_cityscapes2kroad"
CKPT_PATH="/path/to/model/ckpt/upernet_internimage_b_512x1024_320k_cityscapes2kroad.pth"

python deploy.py \
    "./deploy/configs/mmseg/segmentation_tensorrt_static-512x512.py" \
    "./configs/ade20k/${MODEL}.py" \
    "${CKPT_PATH}" \
    "./deploy/demo.png" \
    --work-dir "./work_dirs/mmseg/${MODEL}" \
    --device cuda \
    --dump-info
```
