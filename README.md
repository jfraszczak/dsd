# canopy-volume-estimation


# Semantic Segmentation

## Set-up of Docker container
**Build a Docker image**
```
docker build -t fraszczak/segmentation:latest ./docker/huggingface/
```

**Run container**

```
docker run -it --rm --name fraszczak_segmentation_container --gpus "device=2" --cpuset-cpus 8-15 --runtime=nvidia --shm-size=8G --memory=28G --memory-swap=32G --volume /multiverse/datasets/fraszczak/canopy_analysis:/workspace/fraszczak/datasets fraszczak/segmentation:latest
```

## Dataset

**Dataset is expected to be stored in the following way:**

```
canopy-volume-estimation
└─── data
│   └─── segmentation
│       │  raw
│       └─── annotations
|       |   | instances_default.json (image annotations in COCO format)
|       | 
|       └─── images
|           | 1659006356_57917118.png
|           | 1659006356_723540306.png
|           | ...
```

## Commands for training, evaluation and dataset preparation

**Extract images from the bag file and save in separate output directory**
```
python3 -m src.segmentation.data.extract_images_from_bag
```

**Split dataset randomly into train, val, test subsets**
```
python3 -m src.segmentation.data.split_dataset hydra.output_subdir=null hydra.run.dir=.
```

**Split dataset into train, val, test subsets assuming the last 25 images to be a test set**
```
python3 -m src.segmentation.data.split_dataset_custom hydra.output_subdir=null hydra.run.dir=.
```

**Show training dataset (requires to enable X11 forwarding for Docker)**
```
python3 -m src.segmentation.visualization.show_training_dataset hydra.output_subdir=null hydra.run.dir=.
```

**Train**
```
python3 -m src.segmentation.models.train
```

**Visualize predictions of validation set (requires to enable X11 forwarding for Docker)**
```
python3 -m src.segmentation.models.predict hydra.output_subdir=null hydra.run.dir=.
```

**Evaluate model on test set**
```
python3 -m src.segmentation.models.evaluate hydra.output_subdir=null hydra.run.dir=.
```

**Save metrics throughout models' training + create csv file with best metrics for each model**
```
python3 -m src.segmentation.visualization.compare_models
```

<br /> <br /> <br />



# 3D reconstructions processing
* *This module is written in the most shaky way as outputs of different recosntructions have different formats and various configurations were testes such as using reconstructions
* consisting of only one side. In case of problems please feel free to contact me* *

**Reconstructions are assumed to be stored in the following way:**
```
canopy-volume-estimation
└─── data
│   └─── reconstructions
│       │  2021
│       └─── row_1
|           └─── rtabmap_reconstruction
|                └─── rgbd
|                    | rtabmap_calib
|                    | rtabmap_depth
|                    | ...
```

**Run point cloud processing pipeline to perform segmentation and estimate volumes**
```
python3 -m src.3d_reconstruction.collect_measurements
```

Following snippet of code from collect_measurements.py demonstrates how processing is performed and which parameters shall be specified
```
def collect_measurements_based_on_trunks(path: str, year: int, row: int, reconstruction_type: str, data_source: str) -> None:
    processor = ReconstructionProcessor(path=path, data_source=data_source, reconstruction_type=reconstruction_type, dir=reconstruction_type + '_reconstruction', one_sided=True)
    processor.process(one_side=False, read_segmentations=True)
    trunks_locations = processor.get_trunks_positions()
    canopy = processor.get_canopy_cloud()
    processor.get_segmented_cloud()

    volume_estimator = VolumeEstimator()
    volume_estimator.load_cloud(canopy)
    volume_estimator._get_function_approximated_volume(start=0, end=10, voxel_size=0.03, increment=0.25)
    volume_estimator.compute_volumes(measurements_gps=gps_measurements[year], datum=get_datum(row), voxel_size=0.03, trunks_locations=trunks_locations)

path = 'data/3d_reconstruction/reconstructions/2022/row_4'
collect_measurements_based_on_trunks(path=path, year=2022, row=4, reconstruction_type='rtabmap', data_source='rgbd')
```

**Transform output directory of ART-SLAM reconstruction into an appropriate format used for further processing**
```
python3 -m src.3d_reconstruction.scripts.transform_art_slam
```

**Set appropriate timestamps for /d435i messages of 2021 bags**
```
python3 -m src.3d_reconstruction.scripts.correct_2021_bags
```


<br /> <br /> <br />

# Setup of X11 forwarding
In order to launch GUI applications from the level of Docker container SSH X11 forwarding shall be enabled according to the following steps:

## Local Computer
* Install Xming X server
* Set system variable DISPLAY = localhost:0.0
* Launch X server (remeber to set a display number to 0 or accordingly to the specified in system variables port)

## Remote Machine
* In /.ssh/config file include following lines to enable X11 forwarding
```
ForwardAgent yes
ForwardX11 yes
ForwardX11Trusted yes
```
* At this point it should already be possible to run GUI application from the level of remote machine (can be verified by for example running xmessage command)
* Afterwards, following commands should be executed in bash in order to fix the authentication
```
XAUTH=/tmp/.docker.xauth
xauth nlist $DISPLAY | sed -e 's/^..../ffff/' | xauth -f $XAUTH nmerge -
chmod 777 $XAUTH
```
* While executing docker run following folders should be mounted to provide docker container with the X socket
```
-v /tmp/.X11-unix/:/tmp/.X11-unix
/tmp/.docker.xauth:/tmp/.docker.xauth
```
and network type should be set to host so that the container has the same overview of the network as the host

```
-- network host
```

* Now executing GUI applications from the level of Docker container should be possible
