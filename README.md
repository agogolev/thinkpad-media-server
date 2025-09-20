# Turn your old Lenovo ThinkPad X series into self-hosted home media server with Xubuntu and Jellyfin

I used an old ThinkPad X230 because I happened to have one. It had already been upgraded with an SSD, which is perfect for media storage.

## OS and Paritions

Download the latest xubuntu minimal distro and create a bootable thumb drive: [xubuntu.org](https://xubuntu.org)

Install Xubuntu on a small partition, leaving the rest for the media partition that will be mounted to `/srv/media`. We want `/srv/media` to be on a separate partition so that OS upgrades or changes don't affect the media library.

Here is what my partitions look like in GParted:

[image]

## Permissions

Create media group:
```
sudo groupadd media
```

Create a Jellyfin user. Note that you want the home directory to be on the system partition for Docker to work correctly.
```
sudo adduser --system --home /home/jellyfin --shell /user/sbin/nologin --gid %gid% jellyfin
```
Add jellyfin user to the media group:
```
sudo usermod -aG media a
```
Grant permissions for jellyfin:media to `/srv/media` partition
```
sudo chown -R jellyfin:media /srv/media && sudo chmod -R 755 /srv/media
```
To make all newly created files accessible, set a default ACL (Access Control List):

```
sudo setfacl -d -m g:media:rx /srv/media
```

## Docker & Jellyfin

Do not install Docker via Snap, as the Snap version does not work well with external partitions.
Use apt to install Docker and pull the Jellyfin container from https://hub.docker.com/r/jellyfin/jellyfin:
```
docker pull jellyfin/jellyfin:2025XXXXXXX
```
Per https://jellyfin.org/docs/general/installation/container, create docker-compose.yml for Jellyfin
```
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    user: uid:gid
    network_mode: 'host'
    volumes:
      - /home/jellyfin/config:/config
      - /home/jellyfin/cache:/cache
      - type: bind
        source: /srv/media
        target: /media
        read_only: false
    restart: 'unless-stopped'
    # Optional - alternative address used for autodiscovery
    environment:
      - JELLYFIN_PublishedServerUrl=http://home.media
    # Optional - may be necessary for docker healthcheck to pass if running in host network mode
    extra_hosts:
      - 'host.docker.internal:host-gateway'
```
Bring up the docker container
```
sudo docker compose up
```

### Disable Sleep on Lid Close

```
sudo nano /etc/systemd/logind.conf
```

set HandleLidSwitch=ignore, LidSwitchIgnoreInhibited=no

Run `sudo service systemd-logind restart` for changes to take place.

### Restart Jellyfin Every Day

Open your crontab file for editing by running the following command in your terminal. You'll likely need to use sudo to ensure the cron job has the necessary permissions to manage Docker.
```
sudo crontab -e
```

If this is your first time, you may be asked to choose a text editor.

Add the cron job line to the end of the file:
```
0 0 * * * sudo
```

Save and exit the editor. Cron will automatically update the schedule.

### This is it!

The Jellyfin should show up on your local network: http://192.168.1.110:3923
Access it via Jellyfin app from your devices or directly via http on any browser.


## Power Management

Follow [rhea.dev](https://rhea.dev/articles/2017-07/Home-server-Power-saving) article to use powertop, if you want to be fancy about power usage. 






