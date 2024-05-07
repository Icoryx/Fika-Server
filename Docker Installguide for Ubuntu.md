## SPT/FIKA INSTALLATION ON UBUNTU
This guide includes installing Docker, setting up UFW with Docker, opening Ports and installing SPT/FIKA. For any Clientside configuration refer to the official Documentation for SPT/FIKA and their respective Discord servers.
Keep in mind that SPT is not associated with FIKA and will not provide any support for SPT as long as you use FIKA.

### Table of Contents
1. [Installing Docker on your server](#Installing-Docker-on-your-server)
2. [Setting up ufw with Docker and opening necessary ports](#Setting-up-ufw-with-Docker-and-opening-necessary-ports)
3. [FIKA.Docker Installation](#FIKA.Docker-Installation)
4. [SPT.Docker Installation](#SPT.Docker-Installation)

### Installing Docker on your server

Setup Docker's apt repository
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Install the Docker packages
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
   
Verify the installation by running this hello-world image:
```
sudo docker run hello-world
```
   
The result should say something along "Hello from Docker".

### Setting up ufw with Docker and opening necessary ports
**!!If you use UFW it's important to know that opening ports with docker containers will bypass any UFW Rules!!**

To fix this you can follow this guide and come back here afterwards: https://github.com/chaifeng/ufw-docker
If you decide to ignore this or already use a different fix you can continue with opening the necessary ports for SPT/FIKA:

	```ufw route allow proto tcp from any to any port 6969```
	```ufw route allow proto udp from any to any port 25565```

- If you run your server in your own network make sure to also allow these ports on your router/firewall.
- If you use a VPS make sure to check how your provider handles port forwarding. You may have to do some addition configuration.

You will also need to open these ports in your windows firewall. Atleast the P2P ports is required. Fortunately @DOKDOR created a tool that helps you with that:
https://github.com/DOKDOR/Fika-Firewall-Fixer

### Creating a new user for Docker
It is recommended to create a new non-root user to use for docker containers.
If you already have one use it and skip to cloning the repository.

```
adduser dockeruser
```

To be able to use docker with this user create a docker group and add them to it
```
sudo groupadd docker
sudo usermod -aG docker $USER
```

Log into the newly created user
```
su - dockeruser
```

### FIKA.Docker Installation
Now we can install SPT with FIKA.

To start create a new folder called "fika" and change to it.
```
mkdir fika
cd fika
```

Create a file "Dockerfile" using nano (or vi or whatever you know best, I know vi a bit better) and copy the following text into it.
```
vi Dockerfile
```

Copy this into the file:
```
##
## Dockerfile
## FIKA LINUX Container
##

FROM ubuntu:latest AS builder
ARG FIKA=HEAD^
ARG FIKA_BRANCH=v2.0
ARG SPT=HEAD^
ARG SPT_BRANCH=3.8.1
ARG NODE=20.11.1

RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
WORKDIR /opt

# Install git git-lfs curl
RUN apt update && apt install -yq git git-lfs curl
# Install Node Version Manager and NodeJS
RUN git clone https://github.com/nvm-sh/nvm.git $HOME/.nvm || true
RUN \. $HOME/.nvm/nvm.sh && nvm install $NODE
## Clone the SPT AKI repo or continue if it exist
RUN git clone --branch $SPT_BRANCH https://dev.sp-tarkov.com/SPT-AKI/Server.git srv || true

## Check out and git-lfs (specific commit --build-arg SPT=xxxx)
WORKDIR /opt/srv/project 
RUN git checkout $SPT
RUN git-lfs pull

## remove the encoding from aki - todo: find a better workaround
RUN sed -i '/setEncoding/d' /opt/srv/project/src/Program.ts || true

## Install npm dependencies and run build
RUN \. $HOME/.nvm/nvm.sh && npm install && npm run build:release -- --arch=$([ "$(uname -m)" = "aarch64" ] && echo arm64 || echo x64) --platform=linux
## Move the built server and clean up the source
RUN mv build/ /opt/server/
WORKDIR /opt
RUN rm -rf srv/
## Grab FIKA Server Mod or continue if it exist
RUN git clone --branch $FIKA_BRANCH https://github.com/project-fika/Fika-Server.git ./server/user/mods/fika-server
RUN \. $HOME/.nvm/nvm.sh && cd ./server/user/mods/fika-server && git checkout $FIKA && npm install
RUN rm -rf ./server/user/mods/FIKA/.git

FROM ubuntu:latest
WORKDIR /opt/
RUN apt update && apt upgrade -yq && apt install -yq dos2unix
COPY --from=builder /opt/server /opt/srv
COPY fcpy.sh /opt/fcpy.sh
# Fix for Windows
RUN dos2unix /opt/fcpy.sh

# Set permissions
RUN chmod o+rwx /opt -R

# Exposing ports
EXPOSE 6969
EXPOSE 6970
EXPOSE 6971

# Specify the default command to run when the container starts
CMD bash ./fcpy.sh
```
Save and close the file

Create another file called fcpy.sh
```
vi fcpy.sh
```

And copy the following into that file:
```
# fcpy.sh

#!/bin/bash
echo "FIKA Docker"

if [ -d "/opt/srv" ]; then
    start=$(date +%s)
    echo "Started copying files to your volume/directory.. Please wait."
    cp -r /opt/srv/* /opt/server/
    rm -r /opt/srv
    touch /opt/server/delete_me
    end=$(date +%s)
  
    echo "Files copied to your machine in $(($end-$start)) seconds."
    echo "Starting the server to generate all the required files"
    cd /opt/server
    chown $(id -u):$(id -g) ./* -Rf
    sed -i 's/127.0.0.1/0.0.0.0/g' /opt/server/Aki_Data/Server/configs/http.json
    NODE_CHANNEL_FD= timeout --preserve-status 40s ./Aki.Server.exe </dev/null >/dev/null 2>&1 
    echo "Follow the instructions to proceed!"
    exit 0
fi

if [ -e "/opt/server/delete_me" ]; then
    echo "Error: Safety file found. Exiting."
    echo "Please follow the instructions."
     sleep 30
    exit 1
fi

cd /opt/server && ./Aki.Server.exe

echo "Exiting."
exit 0
```
Save it and close it.

Build the mod
```
docker build --no-cache --label FIKA -t fika .
```

Now run the following command to create the initial files. Make sure to replace "{FULLSPTSERVERPATH}" with your full path to the /SPT.Docker/server/ folder:
```
docker run --pull=never --user $(id -u):$(id -g) -v {FULLSPTSERVERPATH}:/opt/server -p 6969:6969 -p 6970:6970 -p 6971:6971 -p 6972:6972 -it --name fika fika
```

For example:
```
docker run --pull=never --user $(id -u):$(id -g) -v /home/dockeruser/SPT.Docker/server/:/opt/server -p 6969:6969 -p 6970:6970 -p 6971:6971 -p 6972:6972 -it --name fika fika
```

After that has finished you may have to delete the "delete_me" file in the spt server folder again
```
cd ../SPT.Docker/server
rm delete_me
```

When you're done with that you can start the fika container and configure it so it starts on boot
```
docker start fika
docker update --restart unless-stopped fika
```

To check if everything's working as it should you can again look at the console output using this command
```
docker logs fika -f
```

You can now join the server and play with your friends.
You can stop your server like this
```
docker stop fika
```

To install additional mods you just put the mods files into the /SPT.Docker/server/user/mods/ folder like usual. Make sure you restart the fika docker afterwards or even better stop it before and start it again when you're finished:
```
docker restart fika
```

### SPT.Docker Installation without FIKA
This section explains how to install SPT without FIKA.

Clone the SPT git repository
```
git clone https://github.com/umbraprior/SPT.Docker
```

Change into the directory
```
cd SPT.Docker
```

Build the server using the following command (3.8.1). To change to a different Version you can change the hash to the full commit has from the aki git page of your desired version.
Using an older version might introduce problems when installing FIKA later. 
```
docker build --no-cache --build-arg SPT=79a5d32cb276e18a5b4405e1f7823cda4fe8e317 --label SPTAki -t sptaki .
```

Run the image once to create the initial files.
```
docker run --pull=never --user $(id -u):$(id -g) -v $PWD/server:/opt/server -p 6969:6969 -p 6970:6970 -p 6971:6971 -p 6972:6972 -it --name sptaki sptaki
```

Once finished go to {SPT.Docker Path}/server and delete the delete_me file
```
cd server/
rm delete_me
```

Start the aki.server and enable starting on boot.
```
docker start sptaki
docker update --restart unless-stopped sptaki
```

To see if it works you can use this command to show the aki.server console:
```
docker logs sptaki -f
```
To exit press CTRL+C

If you encounter any issues look at the official documentations and ask on the FIKA Discord server.

### Sources:
- Docker Installation: https://docs.docker.com/engine/install/ubuntu/
- Docker with UFW: https://github.com/chaifeng/ufw-docker
- SPT.Docker: https://github.com/umbraprior/SPT.Docker
- FIKA.Docker (incomplete documentation): https://hub.docker.com/r/k2rlxyz/fika
- FIKA Base Mod: https://github.com/project-fika/Fika-Documentation
