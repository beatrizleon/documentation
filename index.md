---
title: "Agile grasper"
sidebar:
  nav: "docs"
toc: true
---
 a
## Overview

This is the starting point for the Agile Grasper Documentation

![alt text](https://www.shadowrobot.com/wp-content/uploads/sgs1_crop.jpg "Logo Title Text 1" =250x)

## Getting Started

If you are unfamiliar with ROS, it is highly recommended that you read the [ROS Tutorials](http://www.ros.org/wiki/ROS/Tutorials) 
If you are unfamiliar with the terminal on Linux, you should look [here](https://askubuntu.com/questions/183775/how-do-i-open-a-terminal)
Shadow software is deployed using Docker. Docker is a container framework where each container image is a lightweight, stand-alone, executable package that includes everything needed to run it. It is similar to a virtual machine but with much less overhead. Follow the instructions in the next section to get the latest Docker container of the grasper up and running.

## Installation Instructions
This one-liner is able to install Docker, download the specified image and create a new container for you. It will also create a desktop icon to start the container and launch the hand.

Before setting up the docker container, the EtherCAT interface ID for the hand needs to be discovered. In order to do so, after plugging the hand’s ethernet cable into your machine and powering it up, please run
```shell
sudo dmesg
```
command in the console. At the bottom, there will be information similar to the one below:
```shell
[490.757853] IPv6: ADDRCONF(NETDEV_CHANGE): enp0s25: link becomes ready
```
In the above example, ‘enp0s25’ is the interface id that is needed. 

With this information you can run the one-liner using the following command
```shell
bash <(curl -Ls http://bit.do/launch-sh) -i [image_name] -n [container_name] -e [interface] -b [git_branch] -r [true/false] -g [true/false] -b [sr_config_branch for dexterous hand]
```

Posible options for the oneliner are:

* -i or --image             Name of the Docker hub image to pull
* -u or --user              Docker hub user name
* -p or --password          Docker hub password
* -r or --reinstall         Flag to know if the docker container should be fully reinstalled (default: false)
* -n or --name              Name of the docker container
* -e or --ethercatinterface Ethercat interface of the hand
* -g or --nvidiagraphics    Enable nvidia-docker (default: false)
* -d or --desktopicon       Generates a desktop icon to launch the hand (default: true)
* -b or --configbranch      Specify the branch for the specific hand (Only for dexterous hand)
* -sn or --shortcutname     Specify the name for the desktop icon (default: Shadow_Hand_Launcher)

To begin with, the one-liner checks the installation status of docker. If docker is not installed then a new clean installation is performed. If the required image is private, 
then a valid Docker Hub account with pull credentials from Shadow Robot's Docker Hub is required. Then, the specified docker image is pulled and a docker 
container is initialized. For all available images please refer to section above. Finally, a desktop shortcut is generated. This shortcut starts the docker container and launches 
the hand.

Usage example hand E:
```
bash <(curl -Ls http://bit.do/launch-sh) -i shadowrobot/dexterous-hand:kinetic -n hand_e_kinetic_real_hw -e enp0s25 -b shadowrobot_demo_hand -r true -g false
```

