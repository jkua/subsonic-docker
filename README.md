# subsonic-docker

This is a simple Docker container for subsonic. You can customize some aspects of subsonic via environment
variables. See the table below for customization options.

## Environment Variables

| Variable Name                    | Description                                                                                               | Default Value      |
| :------------------------------- | :-------------------------------------------------------------------------------------------------------- | :----------------- |
| SUBSONIC_HOME                    | The home directory for subsonic (inside the container).                                                   | /var/subsonic      |
| SUBSONIC_HOST                    | The hostname or IP of the interface to bind to. You very likely don't want to change this in a container. | 0.0.0.0            |
| SUBSONIC_PORT                    | The port to bind subsonic to.                                                                             | 4040               |
| SUBSONIC_HTTPS_PORT              | The port to use for HTTPS.                                                                                | 0 (unused)         |
| SUBSONIC_CONTEXT_PATH            | The path in the URL under which subsonic will reside.                                                     | /                  |
| SUBSONIC_DB                      | The JDBC database string of the DB to use for subsonic. The default is an in-memory HSQLDB.               |                    |
| SUBSONIC_MAX_MEMORY              | The maximum amount of RAM in MB Subsonic may use                                                          | 150                |
| SUBSONIC_DEFAULT_MUSIC_FOLDER    | The default folder (within the container) for music.                                                      | /var/music         |
| SUBSONIC_DEFAULT_PODCAST_FOLDER  | The default folder (within the container) for podcasts.                                                   | /var/music/Podcast |
| SUBSONIC_DEFAULT_PLAYLIST_FOLDER | The default folder (within the container) for playlists.                                                  | /var/playlists     |
| SUBSONIC_UID                     | The numeric user id to use for the subsonic user                                                          | 1000               |
| SUBSONIC_GID                     | The numeric group id to use for the subsonic user group                                                   | 1000               |

## Tracker (MOD, S3M, etc) / MIDI support

The container has ffmpeg, mikmod (for screamtracker modules), timidity (for midi files), and LAME (for encoding screamtracker and midi files to MP3).
Subsonic is not, by default, configured to use mikmod or timitidy. You must add the configuration. In the transcoding section, add a configuration like
the following for screamtracker / etc files:

| Name               | Convert from                                                               | Convert to | Step 1             | Step 2                                                                  |
| :----------------- | :------------------------------------------------------------------------- | :--------- | :----------------- | :---------------------------------------------------------------------- |
| xm, mod, etc > mp3 | xm mod s3m 669 it stm amf dsm far gdm gt2 imf med mtm okt stx ult umx apun | mp3        | mikmod_stdout %s   | lame -r -b %b --tt %t --ta screamtracker --tl %l -S --resample 44.1 - - |
| mid > mp3          | mid                                                                        | mp3        | timidity_stdout %s | lame - -b 64                                                            |

The `mikmod_stdout` and `timidity_stdout` scripts are simple scripts that run mikmod and timidity sending
the output to stdout in wav format for lame to use for encoding. If you have no need to handle tracker or
midi files than you can safely ignore this configuration.

## Volumes

The container will use volumes for for the following directories within the container:

| Volume Path    | Description |
| :------------- | :---------- |
| /var/music     | Path where subsonic will look for music (by default).                 |
| /var/playlists | Path where subsonic will look for (and store) playlists (by default). |
| /var/subsonic  | Path where subsonic stores any state and configuration.               |

## Permissions

Subsonic will be run by a service user inside the container (subsonic) with the UID / GID configured via
the `SUBSONIC_UID`/`SUBSONIC_GID` environment variables (Default `1000`/`1000`). The permissions will be set
on any directory / volume you map onto `/var/subsonic` so that it is owned by subsonic:subsonic. If you
are transferring a subsonic installation to this container make sure to change ownership of everything
under `/var/subsonic` to subsonic:subsonic. You can do this with this command: `chmod -R subsonic:subsonic DIR`
where `DIR` is the directory you are mapping to `/var/subsonic`.

The permissions on any music, playlist, etc must be at least readable by subsonic:subsonic.

## Docker run

Here is an example `docker run` command that you can use to run the container:

```
docker run -it \
    -p "4040:4040/tcp" \
    -v /data/music:/var/music \
    -v /data/subsonic-data:/var/subsonic \
    --name="subsonic" \
    stuckj/subsonic:latest
```

Here's another example with environment variable customizations:

```
docker run -it \
    -p "8080:8080/tcp" \
    -eSUBSONIC_PORT=8080 \
    -eSUBSONIC_MAX_MEMORY=512 \
    -eSUBSONIC_UID=33 \
    -eSUBSONIC_GID=33 \
    -v /data/music:/var/music \
    -v /data/playlists:/var/playlists \
    -v /data/subsonic-data:/var/subsonic \
    --name="subsonic" \
    stuckj/subsonic:latest
```

### Sonos issues
If you have problems with Subsonic's Sonos support while running in Docker, this may be due to Sonos' use of UPnP. There are a few things that can be done to fix this, but note that the following steps reduce your server's network security and carries some level of additional risk!

[Removing network isolation](https://docs.docker.com/engine/network/) from the docker container may help. Add these parameters to the above `docker run` commands:
* `--net=host`
* `-eSUBSONIC_HOST=<your host IP>`

Note that with this, the `-p` port publishing parameters are no longer necessary.

If this still doesn't work, also try disabling the firewall on your host machine. On Ubuntu, try running `sudo ufw disable` and then restarting the container.

## Docker compose

Here is an example `docker-compose.yaml` if you choose to run with docker compose:

```
version: '2'
services:
  subsonic:
    image: stuckj/subsonic:latest
    restart: unless-stopped
    environment:
      - SUBSONIC_MAX_MEMORY=512
      - SUBSONIC_PORT=8080
      - SUBSONIC_UID=33
      - SUBSONIC_GID=33
    ports:
      - 8080:8080/tcp
    volumes:
      - /data/music:/var/music
      - /data/playlists:/var/playlists
      - /data/subsonic-data:/var/subsonic
```

## Running as a service
If you want to run subsonic as a service (not using Docker compose), modify [subsonic-docker.service](subsonic-docker.service) and install it:
1. Copy the file to `/etc/systemd/system/subsonic-docker.service`
2. `sudo systemctl daemon-reload`
3. `sudo systemctl enable subsonic-docker`
4. `sudo systemctl start subsonic-docker`

## TODOs

TODO: Setup auto-detection of new versions in Dockerfile (will need to scrape page).
