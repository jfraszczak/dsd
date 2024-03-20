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

## Commands for training, evaluation and dataset preparation

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





## Setup of X11 forwarding
In order to launch GUI applications from the level of Docker container SSH X11 forwarding shall be enabled according to the following steps:


### Local Computer
* Install Xming X server
* Set system variable DISPLAY = localhost:0.0
* Launch X server (remeber to set a display number to 0 or accordingly to the specified in system variables port)

### Remote Machine
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
