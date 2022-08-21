# SRSP - Sonarr Radarr SabNZBd Plex
The setup **S**onarr**R**adarr**S**abNZBd on Docker is possible with a singel [docker-compose.yml](docker-compose.yml) and a simple [srs.env](srs.env). The [srs.env](srs.env) is the file where all the configuration settings can be customized to ones setup. This documentation assumes the reader has a basic knowledge of the linux terminal and navigating the filesystem.

- [Quick Setup](#quick-setup)
- [Installing Docker on Ubuntu Server PVE](#installing-docker-on-ubuntu-server-pve)
  * [Install](#install)
  * [Configure Docker](#configure-docker)
  * [Setup Docker-Compose](#setup-docker-compose)
  * [Starting the containers](#starting-the-containers)
  * [Docker-compose.yml](#docker-composeyml)
    + [Container names and images](#container-names-and-images)
    + [Folder definitions](#folder-definitions)
      - [System folders](#system-folders)
        * [Example](#example)
      - [Media folders](#media-folders)
        * [Example with only one external harddrive for the libraries](#example-with-only-one-external-harddrive-for-the-libraries)
      - [Ports Users Groups](#ports-users-groups)
        * [Example with default ports](#example-with-default-ports)
      - [Plexclaim](#plexclaim)
  * [Mistakes](#mistakes)
    + [Wrong ENV file name](#wrong-env-file-name)
    + [Not running containers in the background](#not-running-containers-in-the-background)
- [SOURCES](#sources)

## Quick Setup
This QS supposes you already have a docker environement up and running.
Copy the [docker compose file](docker-compose.yml) and the [env file](srs.env) file to a local directory. Make sure that all the paths in the [env file](srs.env) file exist and will be reachable for the Docker containers. Fit ports, user- and group-id's and other settings to your environement.
**Important:** Rename the [env file](srs.env) [srs.env](srs.env) to `.env` -> [#Wrong ENV file name](#wrong-env-file-name) (take note that naming the env file like that will make it invisible to ls (use "ls -ahl"))
Then you just deploy the [docker compose file](docker-compose.yml) with:
```bash
cd C:\YMLDirector\ (in my case /data/compose)
docker-compose up -d
```
Lookup [#Not running containers in the background](#not-running-containers-in-the-background) for explanations about `-d`

## Installing Docker on Ubuntu Server PVE
Source: [Installing Docker and Utils on Ubuntu](https://wiki.ssdt-ohio.org/display/rtd/Install+Docker+and+Docker+tools+on+Ubuntu)

### Install
A few commands result in a quick and dirty Docker installation.
```bash
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] \ https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update
sudo apt-get install docker-ce
```

### Configure Docker
I modified the installation by adding a file `daemon.json` under `/etc/docker/`
```json
{
  "hosts" : ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"],
  "data-root" : "/data/docker"
}
```
This was to modify the directory wherein Docker saves its data. -> Also read [#System folders](#system-folders) 

By editing `/etc/systemd/system/docker.service.d/override.conf` you can then enable the settings from `daemon.json`. You can edit `override.conf` with the command `systemctl edit docker`:
```override.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
```

### Setup Docker-Compose
Docker-Compose needs to be downloaded from github and moved to a directory that is in the `environement` file. You can check these paths with `cat /etc/environement` 
In this case /usr/local/bin is in the `environement` file and the following commands will download and enable `docker-compose` to be usable in the Shell Terminal
```bash
curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Starting the containers
Follow the instructions from the [#Quick Setup](#quick-setup) chapter with your working environement. 

### Docker-compose.yml
This file defines the different Services and their respective configurations.

Al changable variables are defined in the [env file](srs.env) so I will comment on those options which you can then correlate to the [docker compose file](docker-compose.yml).

#### Container names and images
These should be selfexplanatory. The documentation from the the [sources](#sources) provided the image links. 
``` ENV
SABNZBDIMG=lscr.io/linuxserver/sabnzbd:latest
SABNZBDCONTAINERNAME=sabnzbd
```

#### Folder definitions
##### System folders
In the chapter [#Configure Docker](#configure-dockers) you are able to define a different storage folder for the docker installation. This enables you to keep all relevant docker data on a secured nas attached over nfs and run the containers on a small and efficient virtual machine.

Therefore all the folders under "#SOFTWARE CONFIGURATION FOLDER VARIABLES" in the [env file](srs.env) could also be defined to be on the nas storage, which facilitates a future migration and mitigates the damage caused by the failure of the virtual machine.

###### Example
*srsp* = SonarrRadarrSabnzbdPlex
``` Env
#SOFTWARE CONFIGURATION FOLDER VARIABLES
SABNZBDHOME=/nas/srsp/conf/sabnzbd
SONARRHOME=/nas/srsp/conf/sonarr
RADARRHOME=/nas/srsp/conf/radarr
PLEXHOME=/nas/srsp/conf/plex
```

##### Media folders
You will have to give srs access to your existing or new media libraries. So you will have to setup and mount your external harddrive/s, nas, etc. to the virtual machine with your running docker containers.

I recommend to set all paths under *"## Sabnzbd download folders"* to a storage space that is directly attached to the machine running the docker containers. 

Under *"## Media library folders for plex"* you have to set the path to the mountpoint of all your media storage spaces. If you have multiple harddrives or nas you would have to mount those under the path defined by *"SERIESFOLDER"* & *"MOVIESFOLDER"* like this:
``` shell
/medialibrary/media/series/
/medialibrary/media/series/nas1 *or hdd1*/series
/medialibrary/media/series/nas2 *or hdd2*/series
/medialibrary/media/series/nas3 *or hdd3*/series
```

###### Example with only one external harddrive for the libraries
``` ENV
# MEDIA FOLDER VARIABLES
## Sabnzbd download folders
DOWNLOADFOLDER=/dev/sda1/usenet/download
COMPLETEDDOWNLOADS=/dev/sdb1/usenet/complete
INCOMPLETEDOWNLOADS=/dev/sdb1/usenet/incomplete
## Media library folders for plex
SERIESFOLDER=/medialibrary/media/series
MOVIESFOLDER=/medialibrary/media/movies
```

##### Ports Users Groups
My OCD kicked in with the srs ports under *#PORTS* but you could set them to default (you can find the default ports in the [docker compose file](docker-compose.yml))

The user and group id's need to be fitted to your system, but if you built a virtual machine just for these services, you can leve them as is.

###### Example with default ports
``` ENV
#PORTS
SABNZBDPORT=8080
SONARRPORT=8989
RADARRPORT=7878
PLEXPORT=32400
```

##### Plexclaim
The [plex claim](https://www.plex.tv/claim/) is used to add the plex server to an account. This setting can stay empty. You can get your claim token from https://www.plex.tv/claim/

### Mistakes
#### Wrong ENV file name
I had some issues getting the containers running because it would not read my [srs.env](srs.env) file. This was due to my missunderstanding about the required filename. The version of `docker-compose` I installed (1.16.1) did not allow to specify a ENV file. Therefore [srs.env](srs.env) needed to be called `.env`
Here are the contents of the `compose` directory:
>/data/compose ls -a
.env  docker-compose.yml

The errors I would get if the file `.env` was called `srs.env`
>root@Usenet:/data/compose# docker-compose up
WARNING: The RADARRUID variable is not set. Defaulting to a blank string.
WARNING: The RADARRGID variable is not set. Defaulting to a blank string.
WARNING: The TIMEZONE variable is not set. Defaulting to a blank string.
WARNING: The RADARRHOME variable is not set. Defaulting to a blank string.
WARNING: The MOVIESFOLDER variable is not set. Defaulting to a blank string.
WARNING: The COMPLETEDDOWNLOADS variable is not set. Defaulting to a blank string.
WARNING: The RADARRCONTAINERNAME variable is not set. Defaulting to a blank string.
WARNING: The RADARRIMG variable is not set. Defaulting to a blank string.
WARNING: The RADARRPORT variable is not set. Defaulting to a blank string.
WARNING: The SONARRUID variable is not set. Defaulting to a blank string.
WARNING: The SONARRGID variable is not set. Defaulting to a blank string.
WARNING: The SONARRHOME variable is not set. Defaulting to a blank string.
WARNING: The SERIESFOLDER variable is not set. Defaulting to a blank string.
WARNING: The SONARRCONTAINERNAME variable is not set. Defaulting to a blank string.
WARNING: The SONARRIMG variable is not set. Defaulting to a blank string.
WARNING: The SONARRPORT variable is not set. Defaulting to a blank string.
WARNING: The SABNZBDUID variable is not set. Defaulting to a blank string.
WARNING: The SABNZBDGID variable is not set. Defaulting to a blank string.
WARNING: The SABNZBDHOME variable is not set. Defaulting to a blank string.
WARNING: The DOWNLOADFOLDER variable is not set. Defaulting to a blank string.
WARNING: The INCOMPLETEDOWNLOADS variable is not set. Defaulting to a blank string.
WARNING: The SABNZBDCONTAINERNAME variable is not set. Defaulting to a blank string.
WARNING: The SABNZBDIMG variable is not set. Defaulting to a blank string.
WARNING: The SABNZBDPORT variable is not set. Defaulting to a blank string.
ERROR: Version in "./docker-compose.yml" is unsupported. You might be seeing this error because you're using the wrong Compose file version. Either specify a supported version (e.g "2.2" or "3.3") and place your service definitions under the `services` key, or omit the `version` key and place your service definitions at the root of the file to use version 1.
For more on the Compose file format versions, see https://docs.docker.com/compose/compose-file/

#### Not running containers in the background
The Basic command to get a docker-compose.yml up and running is `docker-compose up`
When working in a terminal over ssh, the command will generate output and enter the `docker-compose cli` which you can exit with CTRL+C but this will also stopp all containers.

That is why it is necessary to run the `docker-compose` command with the `-d --detached` flag so that the containers will be startet in the background.

**Solution:**
```bash
cd C:\YMLDirector\ (in my case /data/compose)
docker-compose up -d
```

Example output without `-d` flag (note the last lines at the botton)
```bash
radarr     | [Info] Microsoft.Hosting.Lifetime: Content root path: /app/radarr/bin
sonarr     | [Info] SonarrBootstrapper: Starting Web Server
sabnzbd    | /usr/lib/python3.9/site-packages/cherrypy/process/servers.py:416: UserWarning: Unable to verify that the server is bound on 8080
sabnzbd    |   warnings.warn(msg)
sabnzbd    | 2022-07-10 17:28:51,631::INFO::[_cplogging:213] [10/Jul/2022:17:28:51] ENGINE Serving on http://:::8080
sabnzbd    | 2022-07-10 17:28:51,632::INFO::[_cplogging:213] [10/Jul/2022:17:28:51] ENGINE Bus STARTED
sabnzbd    | 2022-07-10 17:28:51,633::INFO::[SABnzbd:1474] Starting SABnzbd.py-3.6.0
sabnzbd    | 2022-07-10 17:28:51,645::INFO::[dirscanner:117] Dirscanner starting up
sabnzbd    | 2022-07-10 17:28:51,646::INFO::[panic:239] Launching browser with http://127.0.0.1:8080/sabnzbd
sabnzbd    | 2022-07-10 17:28:51,648::INFO::[notifier:123] Sending notification: SABnzbd - SABnzbd 3.6.0 started (type=startup, job_cat=None)
sabnzbd    | 2022-07-10 17:28:52,002::INFO::[zconfig:61] No bonjour/zeroconf support installed
sabnzbd    | 2022-07-10 17:28:52,004::INFO::[ssdp:108] Serving SSDP on 172.18.0.3 as SABnzbd
^CGracefully stopping... (press Ctrl+C again to force)
Stopping radarr  ... done
Stopping sabnzbd ... done
Stopping sonarr  ... done

```

## SOURCES
[linuxserver/sabnzbd](https://hub.docker.com/r/linuxserver/sabnzbd)
[linuxserver/plex](https://hub.docker.com/r/linuxserver/plex)
[linuxserver/radarr](https://hub.docker.com/r/linuxserver/radarr)
[linuxserver/sonarr](https://hub.docker.com/r/linuxserver/sonarr)
[Installing Docker and Utils on Ubuntu](https://wiki.ssdt-ohio.org/display/rtd/Install+Docker+and+Docker+tools+on+Ubuntu)
