# Automated Media Server (Docker)

Starts a fully configured media server setup for my home media server instance.

## Usage

Make some changes to the compose volume mounts and environment vars

``` yml

version: "3.4"

# Include all required services configuration
# Ensure a few things are modified:
# 1. ENV vars
# 2. Volume mounts (for where you would like volumes mounted on your host)
# 3. Decide if you would like to mount /config on each service so you can keep a copy on your host of the config and make changes manually if necessary (outside of container)
services:
  # Front-end jellyfin (reverse-proxy)
  webserver:
    container_name: webserver
    image: nginx:mainline-alpine
    volumes:
      - web-root:/var/www/html # Volume mount for web-root (optional)
      - ./nginx:/etc/nginx/conf.d # Volume mount for nginx config folder (required)
    depends_on:
      - jellyfin
      - ombi
    ports:
      - 80:80
    restart: unless-stopped
    networks:
      - media-server-network
  # NOTE: There are no ports specified for this service since it is accessible using provided nginx configuration
  ombi:
    image: linuxserver/ombi
    container_name: ombi
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New York
    volumes:
      - /home/ben/mediaserver/ombi:/config # Change volume mount for config files
    networks:
      - media-server-network
    restart: unless-stopped
  # NOTE: Make sure that libraries are set to /tv and /movies in configuration
  jellyfin:
    image: linuxserver/jellyfin
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New York
      - UMASK_SET=<022> #optional
    volumes:
      - /home/ben/mediaserver/jellyfin:/config # Jellyfin config / settings
      - /media/jellyfin/tv:/data/tvshows # TV Shows location
      - /media/jellyfin/movies:/data/movies # Movies location
    ports:
      - 7359:7359/udp # Discovery on network
      - 1900:1900/udp # Discover on network by DNLA and clients
    devices:
      - /dev/dri:/dev/dri # Remove this if you do not want to use INTEL GPU for hardware excellerated transcoding
    networks:
      - media-server-network
    restart: unless-stopped
  # NOTE: Make sure to set download directory to /downloads
  deluge:
    image: linuxserver/deluge
    container_name: deluge
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New York
      - UMASK_SET=022 #optional
      - DELUGE_LOGLEVEL=error #optional
    volumes:
      - /home/ben/mediaserver/deluge:/config # Change path for volume mount
      - /media/jellyfin/downloads:/downloads # Change path for volume mount
    ports:
      - 127.0.0.1:8112:8112
    networks:
      - media-server-network
    restart: unless-stopped
  # NOTE: With Sonarr, when setting up downloader, make sure ip address is "deluge:8112"
  sonarr:
    image: linuxserver/sonarr
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New York
      - UMASK_SET=022 #optional
    volumes:
      - /home/ben/mediaserver/sonarr:/config # Change path for volume mount
      - /media/jellyfin/tv:/tv # Change path for volume mount
      - /media/jellyfin/downloads:/downloads # Change path for volume mount
    ports:
      - 127.0.0.1:8989:8989
    networks:
      - media-server-network
    restart: unless-stopped
  # NOTE: Similar to Sonarr, when setting up downloader make sure to enter "deluge:8112" for the host and port respectively
  radarr:
    image: linuxserver/radarr
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New York
      - UMASK_SET=022 #optional
    volumes:
      - /home/ben/mediaserver/radarr:/config # Change path for volume mount
      - /media/jellyfin/movies:/movies # Change path for volume mount
      - /media/jellyfin/downloads:/downloads # Change path for volume mount
    ports:
      - 127.0.0.1:7878:7878
    networks:
      - media-server-network
    restart: unless-stopped
  # NOTE: With Jackett, ensure blackhole directory is setup at /downloads
  # Also: Ensure trackers are setup from config
  jackett:
    image: linuxserver/jackett
    container_name: jackett
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New York
      - AUTO_UPDATE=true #optional
    volumes:
      - /home/ben/mediaserver/jackett:/config # Change path for volume mount
      - /media/jellyfin/downloads:/downloads # Change path for volume mount
    ports:
      - 127.0.0.1:9117:9117
    networks:
      - media-server-network
    restart: unless-stopped

volumes:
  web-root:
    driver: local

# NOTE: It is very important that if you are also running a OpenVPN service on the host, that you include the EXACT subnet address for the network interface to use.
#       The reason for this is to avoid overlapping of IPv4 addresses. This way as long as your openVPN service is not using the 172.16.57.0/24 subnet, you should not have any overlap.
networks:
  media-server-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.16.57.0/24

```

Start the services

```bash

docker-compose up -d

```