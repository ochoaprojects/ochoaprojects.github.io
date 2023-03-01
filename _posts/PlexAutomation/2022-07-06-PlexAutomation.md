---
title: Plex Automation
date: 2022-07-06 12:00:00 -500
categories: [Homelab, Plex]
tags: [docker,containers,webui,management,monitoring,automation,api,plex,sonarr,radarr,jackett,ombi,deluge]
---

How to build a complete Plex server with full automation for accepting and processing plex requests without any manual work.

## Table of Contents
- [The Stack](#the-stack)
- [Tips for managing your hard drives](#tips-for-managing-your-hard-drives)
    - [Optimizing for Ratios](#optimizing-for-ratios)
    - [HDD vs SSD](#hdd-vs-ssd)
    - [HDD Backups vs. Redundancy](#hdd-backups-vs-redundancy)
- [Installing mhddfs](#installing-mhddfs)
- [Mounting Multiple Drives as One](#mounting-multiple-drives-as-one)
- [Installing Docker](#installing-docker)
- [Plex Config](#plex-config)
- [Deluge Config](#deluge-config)
- [Jackett Config](#jackett-config)
- [Sonarr & Radarr Docker Config](#sonarr--radarr-docker-config)
- [Ombi Config](#ombi-config)
- [Start it up!](#start-it-up)

# The Stack

Here's the stack we'll be using. There will be a section describing the installation and configuration for each one of these.

`Docker` lets us run and isolate each of our services into a container. Everything for each of these services will live in the container except the configuration files which will live on our host.

`Plex` is a "client-server media player system". There are a few alternatives, but I chose Plex here because they have a client available on nearly every platform.

`Deluge` is going to act as our download client for our sonarr and radarr applications of the stack. It will be fed torrent files from jackett after it finds the best match based off criteria we will set.

`Jackett` is a tool that Sonarr and Radarr use to search indexers and trackers for torrents.

`Sonarr` is a tool for automating and managing your TV library. It automates the process of searching for torrents, downloading them then "moving" them to your library. It also checks RSS feeds to automatically download new shows as soon as they're uploaded! 

`Radarr` Is a fork of Sonarr that does all the same stuff but for Movies.

`Ombi` is a super simple web UI for sending requests to Radarr and Sonarr.

# Tips for managing your hard drives

Here are some tips for how to manage your hard drives and data.

## Optimizing for Ratios

I use private trackers to get my torrents, and this means I need to maintain a ratio. If you aren't familiar with this term, it basically means you should be uploading as much, if not more than you download.

The best way I've found to do this is to mount your drives directly to the machine that handles your downloads. This means you can configure Sonarr and Radarr to create hardlinks when a torrent finishes. With this enabled, a reference to the data will exist in your media directory and in your torrent directory so Deluge can continue seeding everything you download.

## HDD vs SSD

Can also be written as Space vs. Reliability and Speed. I'm a hoarder when it comes to media now, so I go with HDDs. This means I need to worry about my drives randomly dying.

*If you roll with SSDs, I envy you and you should skip this next section!*

## HDD Backups vs. Redundancy

HDDs are very prone to randomly dying and you need to prepare for this. You can either set up an array of drives and use something like RAID, but you usually end up losing some space and it's not very easy to expand. I'm building my library from scratch so I want to be able to expand the space on my server as I download more media.

So instead, I mount all of my drives and then use mhddfs to treat them as one file system, and what's really cool is that it automatically fills up your drives in order.

*update: I still haven't found a good way to keep backups of this data and am actually working on a way around this now.*

# Installing mhddfs

mhddfs contains a bug where you can occasionally run into a segfault error when mounting your drives. You'll want to install it via the patch in this repo, but it's a bit tricky.

The docker installation method is outdated and doesn't run so you'll have to install `fakeroot` and run the command from the Readme.

```bash
sudo apt-get install -y fakeroot
git clone https://github.com/vdudouyt/mhddfs-nosegfault.git ~/Downloads
cd ~/Downloads/mhddfs-nosegfault
fakeroot dpkg-buildpackage
```

It'll give you trouble about signing, ignore it and install the .deb package

```bash
dpkg -i ~/Downloads/mhddfs_0.1.39+nosegfault2_amd64.deb
```

# Mounting Multiple Drives as One
```bash
mkdir /mnt/hdd1
mkdir /mnt/hdd2
mkdir /mnt/media
```

To get your drives to mount on boot we have to edit your /etc/fstab file. Add the following lines at the end of your file and replace the drive IDs with your own.
```
UUID=fa605d83-106e-4143-bb20-deec7461f08c /mnt/hdd1 ext4 auto,nofail,noatime,rw,user 0 0
UUID=31a25f49-0905-47b9-9f48-e6a5c3f59293 /mnt/hdd2 ext4 auto,nofail,noatime,rw,user 0 0
mhddfs#/mnt/hdd1,/mnt/hdd2 /mnt/media fuse defaults,allow_other 0 0
```

This mounts both drives on `/mnt/hdd1` and `/mnt/hdd2` and then mounts them together via `mhddfs` on `/mnt/media`. Now let’s set up our file system with a folder for torrents and a couple for our Libraries.

```bash
mkdir /mnt/media/torrents
mkdir /mnt/media/Movies
mkdir /mnt/media/Shows
```

# Installing Docker
```bash
sudo apt-get update
sudo apt install docker.io
sudo systemctl start docker
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

I also like to keep my configs in one easy place in case I want to transfer them anywhere, so let’s create a folder to hold our docker container configs and create our `docker-compose.yml` file

```bash
mkdir ~/docker-services
touch ~/docker-services/docker-compose.yml
```

And add this to your docker-compose file. We will be filling in the services in the coming steps. If you are confused on how to add the services or how your file should look, here is a good resource on docker-compose.

```yaml
---
version: "2"
services:
```

# Plex Config
```yaml
plex:
    image: linuxserver/plex:latest
    container_name: plex
    environment:
      - PUID=1000
      - PGID=1000
      - VERSION=docker
    volumes:
      - ~/docker-services/plex/config:/config
      - /mnt:/mnt
    ports:
      - 32400:32400
    restart: unless-stopped
```

This will start your Plex server on port 32400, and add the volumes `/mnt` and `~/docker-services/plex/config` onto the container. If you are trying to move your current Plex configs over, run something like this

```bash
mv /var/lib/plexmediaserver/* ~/docker-services/plex/config/
```

Note that plex is looking for your config directory to contain a single directory `Library`. Look for that directory and copy it over.

# Deluge Config
```yaml
deluge:
    image: linuxserver/deluge
    container_name: deluge
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ~/docker-services/deluge/config:/config
      - /mnt/media/deluge:/mnt/media/deluge
      - /mnt/media/PlexMediaLibrary:/mnt/media/PlexMediaLibrary
    ports:
      - 8112:8112
    restart: unless-stopped
```

Notice how we mount our deluge drive on the container in the same location as the host, rather than something like `/downloads` (which is suggested over at linuxserver). This, plus the config below ensures Sonarr and Radarr send torrents to the right directory.

# Jackett Config
```yaml
jackett:
    image: linuxserver/jackett
    container_name: jackett
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
      - RUN_OPTS=run options here #optional
    volumes:
      - ~/docker-services/jackett/config:/config
      - /mnt/media/torrents/completed:/downloads
    ports:
      - 9117:9117
    restart: unless-stopped
```

This is super basic and just boots your Jackett service on port 9117. Doesn’t need much else.

# Sonarr & Radarr Docker Config
```yaml
sonarr:
    image: linuxserver/sonarr
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
      - UMASK_SET=022 #optional
    volumes:
      - ~/docker-services/sonarr/config:/config
      - /mnt/media:/mnt/media
    ports:
      - 8989:8989
    restart: unless-stopped
  radarr:
    image: linuxserver/radarr
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
    volumes:
      - ~/docker-services/radarr/config:/config
      - /mnt/media:/mnt/media
    ports:
      - 7878:7878
    restart: unless-stopped
```

Make sure those `PUID` and `GUID` match the ID for your user and group… and make sure that user has read-write permissions for `/mnt/media`. Sonarr and Radarr are going to try to be creating folders and files in there when they copy or hard link files over.

If you are running into issues, check the logs of the docker container or the logs in the web UI. It should tell you exactly where it’s having trouble. Then log into the user you set it to run as an attempt the same actions. See whats going on first hand.

# Ombi Config
```yamml
ombi:
    image: linuxserver/ombi
    container_name: ombi
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Los_Angeles
      - BASE_URL=/ombi #optional
    volumes:
      - ~/docker-services/ombi/config:/config
    ports:
      - 3579:3579
    restart: unless-stopped
```

This will open Ombi on port 3579 but sadly they don’t have SSL support by default. I can create a post on adding LetsEncrypt to this setup.

# Start it up!

Run this command to boot up all your services!

```bash
cd ~/docker-services
docker-compose up -d
```

You now have these services running locally. Go to their web UI’s and configure everything.

```
sonarr       @ http://localhost:8989
radarr       @ http://localhost:7878
jackett      @ http://localhost:9117
transmission @ http://localhost:9091
plex         @ http://localhost:32400
```

There is some final configuration you will need to do that is out of the scope of this tutorial, but I can help if need be.

You will have to

1. Configure Jackett with your indexers
2. Point Sonarr and Radarr to Jackett for indexers and Deluge as a download client
3. Tell Sonarr and Radarr to download Movies and Shows folders you created above
4. Tell Sonarr and Radarr to use Hardlinks instead of Copies in advanced settings
5. Configure Ombi to use Sonarr and Radarr for requests

Now go invite your friends to your Plex server