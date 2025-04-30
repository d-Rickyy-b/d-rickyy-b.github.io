---
layout: single
classes: wide
title:  "Moving a docker volume to a local path"
date:   2021-11-14 08:00:00 +0200
category: Misc
tags: [docker, misc]
---

## Introduction

The best way to host applications on your NAS or VPS is probably Docker. Or to be more precise docker-compose. Basically I use docker-compose to run most of the software I host: My mail server, [uptime-kuma](https://github.com/louislam/uptime-kuma), [overleaf](https://github.com/overleaf/overleaf), [pastepwn](https://github.com/d-Rickyy-b/pastepwn), [Jitsi](https://github.com/jitsi/docker-jitsi-meet) and much more.

For website analytics I use the privacy-friendly tool [plausible](https://plausible.io/). Self-hosted of course.
And recently I needed to move data from a docker volume to a local directory.
Its default `docker-compose.yml` file uses docker volumes like this:

```yml
services:
  plausible_db:
    image: postgres:12
    restart: always
    volumes:
      - db-data:/var/lib/postgresql/data

[...]

volumes:
  db-data:
    driver: local
```

With this config, docker creates a new named volume internally ("db-data").
You may now ask what's the bad thing about it?
Backing up your docker-composed based projects is more complicated, because you need to remember to back up the volume directory as well.

To find the actual volume directory in the file system, we can use the inspect subcommand of docker:
`$ docker inspect plausible_db_1`. Under "HostConfig" -> "Binds" we find the imported volumes/paths.

There are multiple mount types in Docker.
The four we are interested in for this blog post are:

1. Anonymous volumes
2. Named volumes
3. Directly mounted directories
4. The read-write layer of the container

**Named volumes** is what we saw in the docker-compose file shown earlier.
These are mostly used for storing essential data and described in the `docker-compose.yml` file.

**Anonymous volumes** are created by using the `VOLUME` statement in the Dockerfile.
That means they are created automatically, without additional configuration after starting a container.

For essential data (that should be contained in a backup) I always use **directly mounted directories** in the same directory as the docker-compose.yml file is located in.
That makes creating & restoring your backup a lot easier because you have everything in one place.

**The read-write layer of the container** is the directory where all changes to the container made at runtime are stored by default.

## Steps to move the volume

### 1. Locate the volume directory

Usually all docker volumes are stored in the `/var/lib/docker/volumes/` directory.
But you can also find the volumes and corresponding directories with the docker inspect command.

{% raw %}
```bash
$ docker inspect -f "{{ .Mounts }}" <container_id>

[{volume e0301dce[...]34dda48556 /var/lib/docker/volumes/e0301dce[...]34dda48556/_data /data/configdb local rw true }]
```
{% endraw %}

In case you want to copy data from the container's read-write layer, you can use the following command:

{% raw %}
```bash
$ docker inspect -f "{{ .GraphDriver.Data.MergedDir }}" <container_id>
```
{% endraw %}

### 2. Move data to local directory

Now we can move the data to the target directory.
I prefer copying it to have a backup in case something goes wrong.
Make sure to use the `-p` switch to preserve the permissions.
Otherwise you'll likely run into issues.

`$ cp -rp /var/lib/docker/volumes/<volume_name_or_id>/_data/ /path/to/your/dir`

You can of course delete the volume later with `$ docker volume rm <volume_id>`

### 3. Change docker-compose file

In your `docker-compose.yml` now change all the occurrences of the volume name with the target path you copied the data to in step 2).
You can also use paths relative to the `docker-compose.yml` file.

```diff
     volumes:
-      - db-data:/var/lib/postgresql/data
+      - ./data/db-data:/var/lib/postgresql/data
```

### 4. Delete/recreate containers

Now stop and remove the running containers.

`$ docker-compose down`

That automatically **removes** all containers defined in the `docker-compose.yml` file, the defined networks and the default network. You can also use `stop` instead of `down`. Then you need to delete the containers yourself with `$ docker container rm <container_id>`.
After that's done, just restart the service: `$ docker-compose up -d`.  

Now the compose service should use the data from the local directory.

## Troubleshooting

In case you find a volume you don't know which container is using, simply use `$ docker ps -a --filter volume=VOLUME_NAME_OR_MOUNT_POINT` to find details about the container(s) that use the image.
