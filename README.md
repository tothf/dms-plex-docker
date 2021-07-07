# How to migrate Plex from DMS package to Docker

As per today (6th of July 2021) DMS 7 is available but Plex doesn't quite working on it. Upgrade is not straightforward. So I decide to migrate my Plex to a docker container running it on the same Synology NAS so I can upgrade to DSM 7 withput issues.
It is very easy but it will need a bit of downtime.

## Pre-reqs
1. Make sure that you have enough space so you can copy your Plex shared folder to another location
1. Go to package center and install Docker and make sure it's running
   <img width="1477" alt="Docker" src="https://user-images.githubusercontent.com/6927606/124609584-55c56480-dea2-11eb-9e43-34a916a0d5a0.png">
1. Upgrade Plex to the latest version<br>
   Plex is a bit outdated in the Package Center. You can download latest version for DSM from [here](https://www.plex.tv/media-server-downloads/#plex-media-server)<br>
   The time when I write this article the latest version is *1.23.3.4707* same as in the docker image [here](https://hub.docker.com/r/plexinc/pms-docker/tags?page=1&ordering=last_updated)<br>
   After downloading the pkg file, open Pacage Center, click on "Manual install", upload and install the pkg file
1. Check if Plex works fine after the upgrade
1. Stop Plex in Package Center

## Migrate Plex to docker
1. Login to your Synology NAS via SSH as root<br>
   On Linux and Mac you can simply use the terminal, On Windows you can use [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)<br>
   In case you login with different user you can gain root access running ```sudo su -``` command
1. Pull the latest Plex docker image (make sure the latest version is same like the one you have installed on the NAS)<br>
   `docker pull plexinc/pms-docker:latest`<br>
   or you can specify the version you need<br>
   `docker pull plexinc/pms-docker:1.23.3.4707-ebb5fe9f3`
1. Create volumes for plex container<br>
   1. For Plex database and config: `docker volume create plex-config`
   1. For Plex transcode cache: `docker volume create plex-config`
1. Find docker volume directory and Plex shared folder<br>
   1. In case you have a single volume called *volume1*:<br>
     1. Plex folder should be: /volume1/Plex<br>
     1. Docker volume plex-config should be: /volume1/@docker/volumes/plex-config<br>
   1. In case you have multiple volumes<br>
     1. Please check in Control Panel under Shared Folder which volume is Plex on<br>
     1. Please check in Package Center which volume Docker is installed on
     1. Replace the volume name accordingly in the paths above
1. Sync Plex DB and config to plex-config Docker volume<br>
   This can take a while, depends on the size of your Plex inventory, so please be patient!<br>
   To start the sync run `rsync -a /volume1/Plex/Library /volume1/\@docker/volumes/plex-config/_data/`<br>
   This will sync all data bit-by-bit.
1. Once the sync is done we will need the user and group who owns the Plex shared folder<br>
   1. List the files under the plex-config volume: `ls -l /volume1/\@docker/volumes/plex-config/_data/`<br>
      Should be something like:<br>
      ```
      # ls -l /volume1/\@docker/volumes/plex-config/_data/
      total 0
      drwx------ 1 plex users 38 Jul  6 13:34 Library
       ```
      This means the user is called plex and users is the group
   1. Get the UID and GID by running `id plex`<br>
      Should be like:<br>
      ```
      # id plex
      uid=1027(plex) gid=100(users) groups=100(users),65536(video)
      ```
      This means UID is 1027 and GID is 100
1. The media folder<br>
   In Plex the libraries are pointed to different folders.<br>
   Let's say we have a Shared folder /volume1/Media. Under this we have Movies and TVShows folders where we have our content.<br>
   Since in Plex our libraries are pointed to /volume1/Media/Movies and /volume1/Media/TVShows we will mount /volume1/Media into our Plex container
1. Start the docker container<br>
   To start the plex container run the following command (Change time zone, path, PLEX_UID, PLEX_GID and docker image name accordingly):
   ```
   docker run \
   -d \
   --name plex \
   --network=host \
   --restart unless-stopped \
   -e TZ=Asia/Singapore \
   -e PLEX_UID=1027 \
   -e PLEX_GID=100 \
   -v plex-config:/config \
   -v plex-transcode:/transcode \
   -v /volume1/Media:/volume1/Media:ro \
   plexinc/pms-docker:latest
   ```
   I've uploaded a docker-compose file as well in case you would like to run it with docker-compose
1. Check if Plex running fine<br>
   The service should listen on port 32400<br>
   `netstat -nl |awk '/:32400/ {print}'`<br>
   Should be like:
   ```
   # netstat -nl |awk '/:32400/ {print}'
   tcp6       0      0 :::32400                :::*                    LISTEN
   ```
1. Login to WebUI or check Plex app on phone/tablet/TV/etc.<br>
   Open in webbrowser: http://your.nas.host.or.ip:32400/web<br>
   
   
At this point you can freely upgrade to DSM 7 and you Plex will work as it was before. Before or After upgrade you can remove Plex from DSM and delete the data.

Enjoy ;)
   
