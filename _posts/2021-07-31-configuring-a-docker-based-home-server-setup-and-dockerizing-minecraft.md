---
layout: post
title: "Configuring a Docker-Based Home Server - Setup and Dockerizing Minecraft" 
---

In running a home Minecraft and movie server in the past, I encountered a few annoying issues that made maintaining the server a chore, and caused a ton of issues when it came to reliability, when all I really wanted to do was enjoy the services I was providing.

To cure this ailment, I'm setting out to find a good low-maintenance solution to self-hosting and using it as an excuse to play around with [Docker](https://www.docker.com/). While I have a bit of experience setting up a [development environment in docker](https://github.com/uga-robotics/hexacopter-env), I have not used the `docker-compose` feature or tried to manage services with it, which is what I plan to do in this project.

## Setting Up The Server Box and Dev Setup

My server box is a hodgepodge of my old desktop parts, which probably needs to be upgraded soon, but that's a problem for future me. Here is a parts list:

| Part          | Model                                |
|---------------|--------------------------------------|
| Processor     | [intel i5 4670k](https://www.amazon.com/gp/product/B00CO8TBOW/ref=ppx_yo_dt_b_asin_title_o01_s00?ie=UTF8&psc=1)                       |
| RAM           | [Corsair LPX 16Gb (2x8) DDR3 1666 Mhz](https://www.amazon.com/Corsair-CML16GX3M2A1600C9-Vengeance-Desktop-Memory/dp/B009YOJZ5E/) |
| Motherboard   | [MSI B85M-E45 (m-atx, LGA 1150)](https://www.amazon.com/MSI-B85M-E45-Desktop-Motherboard-Processor/dp/B00TP5WKY0/)       |
| SSD (OS)      | [ADATA Premier SP550 240GB](https://www.amazon.com/gp/product/B013J7Q338/)            |
| HDD (Storage) | [Seagate ST4000VN008](https://www.amazon.com/gp/product/B07H289S79/)                  |
| Power Supply  | [EVGA 100-BT-0450-K1](https://www.amazon.com/gp/product/B01N9X3F8F/)                  |
| Case          | [Silverstone SST-SG13B-Q-USA](https://www.amazon.com/gp/product/B07MNBWRJT/)          |

The box is running [Ubuntu 20.04 LTS](http://www.releases.ubuntu.com/20.04/) in the 240GB SSD with the [XFCE desktop](https://xfce.org/). The 4TB Seagate drive will be my main storage drive for anything that doesn't need quick access (movies, music, GitLab, other large storage). I've got the server hooked up to my secondary monitor over HDMI, and I'm using [Synergy](https://symless.com/synergy) to share my keyboard/mouse with my desktop. Though I have the server desktop available to me, I plan on doing most of the setup over SSH. I just like having the option to do good ole' point and click, and it's very useful for organizing/moving/renaming files, which I'll need to do a lot of for the movie server. 

Once I've got Synergy setup to share my mouse and keyboard and all the software updated, we can start setting up some development stuff both on my desktop and on the server.

Using the [Ubuntu Linux instructions on the Docker website](https://docs.docker.com/engine/install/ubuntu/), I installed both Docker and `docker-compose` on my desktop (also Ubuntu Linux) and my server box. I'll be doing the bulk of development and testing on my desktop, so I've set up a [Git repo](https://github.com/ktkoffroth/mc-compose) to keep track of the changes and provide a source of entertainment for some of the more experienced readers.

## Dockerizing A Minecraft Server

Once the Git repo is created, we'll go ahead and populate it with the usual suspects: A README I intend to spruce up but never do, a Dockerfile and `docker-compose.yml`, and a `scripts`.

To describe the Docker container setup we'll need, I've pasted the Dockerfile below, and I'll go through it:

```dockerfile
FROM openjdk:16.0.1-slim
LABEL maintainer="Kevin Koffroth <ktkoffroth@gmail.com>"
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update -q && \
    apt-get upgrade -yq && \
    apt-get install -yq git wget
CMD ["bash"]
```

Since all Minecraft servers for version 1.17+ will run on Java 16, we'll use the OpenJDK 16 Slim image from Docker Hub as our base image. After setting the usual maintainer (me!) and arguments (`DEBIAN_FRONTEND=noninteractive` disables dialog boxes during the build), I update and upgrade the image.

Next, I install `git` and `wget`, which are needed to build the Minecraft server I'll be using, [SpigotMC](https://www.spigotmc.org/). Part of the functionality I planned for this server is the ability to update to the latest version of SpigotMC semi-automatically. The Spigot community uses a build tool called, erm, `BuildTools` to generate the server Jars, which requires `git` to function. We'll see how `wget` is used momentarily. 

## Managing the Container with `docker-compose`

Now we enter unexplored territory for me. Having never used `docker-compose`, the first place I went for guidance was the [official documentation](https://docs.docker.com/compose/), which gave me a great overview of the purpose and features of it. 

You may be thinking to yourself, "why is this guy using `docker-compose` for production?", well the answer is that I felt it was all I *needed* for container management. While I could spend a good chunk of time learning and setting up [Kubernetes](https://kubernetes.io/) to manage my relatively small fleet of services, I felt I could learn a bit more about DevOps and containers by building my own automation on top of the admittedly limited (by comparison) feature set of `docker-compose`.

Now, on to the interesting bits. Below, I've pasted the contents of the `docker-compose.yml` file for inspection:

```yml
version: "3.9"

# Bridge Network for MC server and any related services
networks:
    bridge-network:
      external:
        name: bridge-network  

# Services definition
services:
    # Minecraft server 
    mc:
        image: "ktkoffroth/spigot-mc:0.4"
        restart: always
        stdin_open: true
        tty: true
        env_file: ./env/mc-variables.env
        networks:
            - bridge-network
        volumes: 
            - /media/ktkoffroth/NAS/MC:/root/MC
            - ./scripts:/root/scripts
        ports:
            - "25565:25565"
        entrypoint: /root/scripts/entrypoint.sh
```

First, I've set up a bridge network for the `docker-compose` instance by creating a regular docker network, called `bridge-network`, and declaring it in the compose file using the external key. This custom network is mostly for future proofing, as it will allow me to connect containers from multiple docker-compose instances together and provide services for each other.

Next is the meat and potatoes of the file, the `services` definition. Now, I have one service called `mc`, which uses the latest build of the docker container we defined above, [hosted on my Docker Hub account](https://hub.docker.com/repository/docker/ktkoffroth/spigot-mc).  In the future, if I would like to run a few more servers, such as ones running different mod packs, I could simply copy and modify this service definition and have it up relatively quickly.

`restart`, `stdin_open`, and `tty` are all relatively common parameters for a service, which restart the service (always), open up stdin, and enable TTY respectively.

`env_file` points to a `.env` file with environment variables we'd like the container to use. In this case the file we point to, `mc-variables.env` has one environment variable called `VANILLA_VERSION`, which controls the version of Spigot we'd like to build and run in the container. We'll see how this is used in a bit:

```
# Spigot Version
VANILLA_VERSION=1.17.1
```

`networks` attaches the service to the network we defined earlier. Again, more future proofing.

`volumes` lets us provide data on the host OS to the container. In this case, I provide the path to the Minecraft server files, on the 4 TB NAS drive mentioned earlier, and the `scripts` directory from the repository.

`ports` allows the user to route port-specific traffic from the docker container to the host OS. In this case, Minecraft uses port 25565, so I simply route 25565 from the container to the host.

`entrypoint` specifies a script to run on container startup. The script in question is as below:

```bash
#!/bin/bash

# go to /root/MC/BuildTools directory, get latest version of spigot BuildTools,
# and run BuildTools to get latest version of spigot server .jar
cd /root/MC/BuildTools/
wget -N https://hub.spigotmc.org/jenkins/job/BuildTools/lastSuccessfulBuild/artifact/target/BuildTools.jar
git config --global --unset core.autocrlf
rm -rf apache-maven-* BuildData Bukkit CraftBukkit Spigot work BuildTools.log.txt
java -jar BuildTools.jar --rev $VANILLA_VERSION

# move server .jar to server files directory, overwriting if necessary
mv -u spigot-$VANILLA_VERSION.jar ../Spigot-Vanilla/spigot-$VANILLA_VERSION.jar

# move to server files directory, edit start.sh for AutoRestart plugin, and run start command
cd ../Spigot-Vanilla
sed -i "s/^java.*/java -Xms2G -Xmx2G -XX:+UseG1GC -jar spigot-$VANILLA_VERSION.jar/" ./start.sh
java -Xms2G -Xmx2G -XX:+UseG1GC -jar spigot-$VANILLA_VERSION.jar

```

This script is semi-automatically updates the server for me. Whenever I run `docker-compose up`, this script grabs the latest version of `BuildTools` from the community site, sets up the build environment, and runs it with the requested server version as the environment variable I set earlier. It then moves the newly created server Jar to the correct location, edits the `start.sh` script the server uses to periodically restart (it helps with server performance), and finally launches the server jar!

Now, all that needs to be done to upgrade the server to the latest version is to change the `.env` file, kill the docker process (e.g. `docker kill`) and restart `docker-compose` using `docker-compose up -d`!

## Conclusion

This was a relatively short project and I learned a ton about `docker-compose`. This setup also has tons of room for improvement. For instance, I'd like my server to automatically detect an update to the Spigot server, and apply it without having to manually change a version number. It would also be helpful if it could do the same for my server plugins. Oh well, I'll let future me figure that stuff out. 
