# HybridLateCascadeFusion

![image](https://github.com/user-attachments/assets/9014722d-f77d-4ef7-8eb4-18e88dfe35c6)

This repository is the code release for the paper "A Multimodal Hybrid Late-Cascade Fusion Network for Enhanced 3D Object Detection", published in the 2024 ECCV workshop "Multimodal Perception and Comprehension of Corner Cases in Autonomous Driving"
[Paper](https://arxiv.org/abs/2504.18419)

Please refer to [LCF3D](https://github.com/CarloSgaravatti/LCF3D) for an extension of this work to the single-view and multi-view scenarios.

## Getting Started

### 1. Clone the repository

```
https://github.com/CarloSgaravatti/HybridLateCascadeFusion.git
```

### 2. Install the python packages

We used torch 2.0.1 and cuda 11.7. Other versions, compatible with mmcv, mmdet and mmdet3d are still possible but not guaranteed to work. To fully reproduce our environment follow these steps.

```
conda create -n hlcf python=3.10
conda activate hlcf
pip install torch==2.0.1 torchvision==0.15.2 --index-url https://download.pytorch.org/whl/cu117
pip install openmim
mim install mmengine
```

Install mmcv from source to make it compatible with the cuda version used to compile torch:
```
mim install git+https://github.com/open-mmlab/mmcv.git@v2.1.0 -v
```

Install mmdetection:
```
mim install git+https://github.com/open-mmlab/mmdetection.git@v3.3.0
```

Install mmdetection3d from our code (which is based on version 1.4.0):
```
cd HybridLateCascadeFusion/src/mmdetection3d
pip install -e . -v
```

It may be necessary to install the following library versions for compatibility with mmdet and mmdet3d:
```
pip install numpy==1.23.0 numba==0.59.1 scipy==1.13.0
```

## Data preparation

Follow these [instructions](https://mmdetection3d.readthedocs.io/en/latest/advanced_guides/datasets/kitti.html) to prepare the KITTI dataset. Additionally, please download the right color images from [here](https://www.cvlibs.net/datasets/kitti/eval_object.php?obj_benchmark=3d) and place them into the dataset folder. After data preparation, you should have the following structure:

```
kitti
├── ImageSets
│   ├── test.txt
│   ├── train.txt
│   ├── trainval.txt
│   ├── val.txt
├── testing
│   ├── calib
│   ├── image_2
│   ├── image_3
│   ├── velodyne
│   ├── velodyne_reduced
├── training
│   ├── calib
│   ├── image_2
│   ├── image_3
│   ├── label_2
│   ├── velodyne
│   ├── velodyne_reduced
├── kitti_infos_train.pkl
├── kitti_infos_val.pkl
├── kitti_infos_test.pkl
├── kitti_infos_trainval.pkl
```

## Run the code

See [config.md](./config.md) for a detailed explanation on how to set the hyperparameters and the single-modal models for this method.

### Pretrained weights

We provide the weights of the [Faster RCNN](https://drive.google.com/file/d/19624AZ_tneus6eSmKv0OHEi-aWb9kIyt/view?usp=drive_link) and the [Frustum Localizer](https://drive.google.com/file/d/1mZWBZ_DLZv4ofumRuOvAI0yPUi5W-lyb/view?usp=sharing) that we have used. You can find the corresponding configuration files for mmdetection and mmdetection3d in this [folder](./src/model_configs/). For the LiDAR detector, you can use the pretrained models from mmdet3d, e.g. [PointPillars](https://github.com/open-mmlab/mmdetection3d/tree/main/configs/pointpillars).

### Demo and Visualization
To run a demo of the code:

```
python $HOME/HybridLateCascadeFusion/src/demo.py \
  -output_dir /path/to/output/folder \
  -img_path_left /path/to/img_left/file.png \
  -img_path_right /path/to/img_right/file.png \
  -lidar_path /path/to/lidar/file.bin \
  -calib_path /path/to/calib/file.txt \
  -late_fusion_config /path/to/calib/cfg.json \
  -device cuda:0 \
  --visualize
```
Where the calibration file should be in the usual KITTI format, ```img_path_left``` should be the left image (described by the camera matrix P2 in KITTI) and ```img_path_right``` should be the right image (described by the camera matrix P3 in KITTI). By setting ```--visualize``` you should expect to have in the folder specified into ```-output_dir``` the following files:
```
output_path
├── fusion_detections_left_proj.png
├── lidar_detections_left_proj.png
├── predictions.pkl
├── rgb_detections_left.png
├── rgb_detections_right.png
```
where ```fusion_detections_left_proj.png``` shows the output 3d bounding boxes, projected in the left image, ```lidar_detections_left_proj.png``` contains the 3d bounding boxes of the LiDAR branch, before fusion, ```rgb_detections_left.png``` and ```rgb_detections_right.png``` show the 2D bounding boxes of the left and right images.

### Evaluation on KITTI validation set

To run the evaluation on the KITTI dataset:
```
python $HOME/HybridLateCascadeFusion/src/eval.py \
  -output_dir /path/to/output \
  -kitti_root_path /path/to/kitti/training \
  -annotation_file_eval /path/to/kitti/kitti_infos_val.pkl \
  -validation_split_path /path/to/kitti/ImageSets/val.txt \
  -late_fusion_config /path/to/calib/cfg.json \
  -device cuda:0
```

### Remarks

Feel free to experiment with any 2D Camera-based Object Detector, 3D LiDAR-based Object Detector and configurations of the hyperparameters. The code support any 3D model of [mmdet3d](https://github.com/open-mmlab/mmdetection3d) and any 2D model of [mmdet](https://github.com/open-mmlab/mmdetection) or [ultralytics](https://github.com/ultralytics/ultralytics).

#### Training a Frustum Localizer

While for the 2D RGB detector and the 3D LiDAR detector you can rely on standard procedures from [mmdet](https://github.com/open-mmlab/mmdetection) and [mmdet3d](https://github.com/open-mmlab/mmdetection3d), respectively, for our Frustum Localizer you follow these steps:

1. Create the dataset for the Frustum Localizer:
```
python $HOME/HybridLateCascadeFusion/src/create_frustum_dataset.py \
  --out_path /path/to/output \
  --kitti_path /path/to/kitti \
  --min_points_per_frustum 10
```

2. Create a configuration file

You can refer to the original Frustum Localizer [config](./src/model_configs/frustum_pointnet.py) for an example of configuration file. You need to change the ```data_root``` variable with the output path of the previous step. Feel free to experiment with different hyperparameters for the architecture of the model. Please refer to [Frustum PointNet](https://arxiv.org/abs/1711.08488) for a detailed description of the model.

3. Train the model
```
python $HOME/HybridLateCascadeFusion/src/mmdetection3d/tools/train.py \
	/path/to/config.py \
	--work-dir="/where/to/save/the/checkpoints
```

## Citation

If you find this project useful, please consider citing our work.

```
@InProceedings{10.1007/978-3-031-91767-7_23,
  author="Sgaravatti, Carlo and Basla, Roberto and Pieroni, Riccardo and Corno, Matteo and Savaresi, Sergio M. and Magri, Luca and Boracchi, Giacomo",
  editor="Del Bue, Alessio and Canton, Cristian and Pont-Tuset, Jordi and Tommasi, Tatiana",
  title="A Multimodal Hybrid Late-Cascade Fusion Network for Enhanced 3D Object Detection",
  booktitle="Computer Vision -- ECCV 2024 Workshops",
  year="2025",
  publisher="Springer Nature Switzerland",
  address="Cham",
  pages="339--356",
  isbn="978-3-031-91767-7"
}
```

## Acknowledgements

This repository is built upon [mmdetection3d](https://github.com/open-mmlab/mmdetection3d) and [mmdetection](https://github.com/open-mmlab/mmdetection).
Our implementation of the Frustum Localizer is highly inspired by [Frustum PointNets](https://github.com/charlesq34/frustum-pointnets)
