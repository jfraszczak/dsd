# canopy-volume-estimation


# Semantic Segmentation

Build a Docker image
```
docker build -t fraszczak/segmentation:latest ./docker/huggingface/
```

Run container

```
docker run -it --rm --name fraszczak_segmentation_container --gpus "device=2" --cpuset-cpus 8-15 --runtime=nvidia --shm-size=8G --memory=28G --memory-swap=32G --volume /multiverse/datasets/fraszczak/canopy_analysis:/workspace/fraszczak/datasets fraszczak/segmentation:latest
```

Dataset is expected to be stored in the following way:

```
canopy-volume-estimation <br />
└─── data <br />
│   └─── segmentation <br />
│       │  raw <br />
│       └─── annotations <br />
|       |   | instances_default.json (image annotations in COCO format) <br />
|       | <br />
|       └─── images <br />
|           | 1659006356_57917118.png <br />
|           | 1659006356_723540306.png <br />
|           | ... <br />
```


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
