
# invoiceplane-multi-arch

- [Introduction](#introduction)
  - [Contributing](#contributing)
  - [Issues](#issues)
- [Getting started](#getting-started)
  - [Installation](#installation)
  - [Quickstart](#quickstart)
  - [Persistence](#persistence)
- [Maintenance](#maintenance)
  - [Creating backups](#creating-backups)
  - [Restoring backups](#restoring-backups)
  - [Upgrading](#upgrading)
  - [Shell Access](#shell-access)

## Credit

Thanks to [sameersbn/docker-invoiceplane](https://github.com/sameersbn/docker-invoiceplane) for this project.

## Introduction

InvoicePlane is a self-hosted open source application for managing your quotes, invoices, clients and payments.

This is a fork from [sameersbn/docker-invoiceplane](https://github.com/sameersbn/docker-invoiceplane) to support invoiceplane running on arm64 architecture.

## Installation

Automated builds of the image are available on [Dockerhub](https://hub.docker.com/r/ruanbekker/invoiceplane) and is the recommended method of installation.

```bash
docker pull ruanbekker/invoiceplane:1.5.11
```

## Quickstart

The quickest way to start using this image is with [docker-compose](https://docs.docker.com/compose/).

```bash
wget https://raw.githubusercontent.com/ruanbekker/docker-invoiceplane/master/docker-compose.yml
```

Update the `INVOICEPLANE_URL` environment variable in the `docker-compose.yml` file with the url from which InvoicePlane will be externally accessible.

```bash
docker-compose up
```

Alternatively, you can start InvoicePlane manually using the Docker command line.

Step 1. Launch a MySQL container

```bash
docker run --name invoiceplane-mysql -itd --restart=always \
  --env 'DB_NAME=invoiceplane_db' \
  --env 'DB_USER=invoiceplane' --env 'DB_PASS=password' \
  --volume /srv/docker/invoiceplane/mysql:/var/lib/mysql \
  sameersbn/mysql:5.2.26
```

Step 2. Launch the InvoicePlane php-fpm container

```bash
docker run --name invoiceplane -itd --restart=always \
  --link invoiceplane-mysql:mysql \
  --env 'INVOICEPLANE_FQDN=invoice.example.com' \
  --env 'INVOICEPLANE_TIMEZONE=Asia/Kolkata' \
  --volume /srv/docker/invoiceplane/invoiceplane:/var/lib/invoiceplane \
  ruanbekker/invoiceplane:1.5.11 app:invoiceplane
```

Step 3. Launch a NGINX frontend container

```bash
docker run --name invoiceplane-nginx -itd --restart=always \
  --link invoiceplane:php-fpm \
  --volumes-from invoiceplane \
  --publish 10080:80 \
  ruanbekker/invoiceplane:1.5.11 app:nginx
```

Point your browser to [http://invoice.example.com:10080/setup](http://invoice.example.com:10080/setup) to complete the setup.

## Persistence

For InvoicePlane to preserve its state across container shutdown and startup you should mount a volume at `/var/lib/invoiceplane`.

> *The [Quickstart](#quickstart) command already mounts a volume for persistence.*

SELinux users should update the security context of the host mountpoint so that it plays nicely with Docker:

```bash
mkdir -p /srv/docker/invoiceplane
chcon -Rt svirt_sandbox_file_t /srv/docker/invoiceplane
```

## Maintenance

### Creating backups

The image allows users to create backups of the InvoicePlane installation using the `app:backup:create` command or the `invoiceplane-backup-create` helper script. The generated backup consists of uploaded files and the sql database.

Before generating a backup — stop and remove the running instance.

```bash
docker stop invoiceplane && docker rm invoiceplane
```

Relaunch the container with the `app:backup:create` argument.

```bash
docker run --name invoiceplane -it --rm [OPTIONS] \
  ruanbekker/invoiceplane:1.5.11 app:backup:create
```

The backup will be created in the `backups/` folder of the [Persistent](#persistence) volume. You can change the location using the `INVOICEPLANE_BACKUPS_DIR` configuration parameter.

> **NOTE**
>
> Backups can also be generated on a running instance using:
>
>  ```bash
>  docker exec -it invoiceplane invoiceplane-backup-create
>  ```

By default backups are held indefinitely. Using the `INVOICEPLANE_BACKUPS_EXPIRY` parameter you can configure how long (in seconds) you wish to keep the backups. For example, setting `INVOICEPLANE_BACKUPS_EXPIRY=604800` will remove backups that are older than 7 days. Old backups are only removed when creating a new backup, never automatically.

### Restoring Backups

Backups created using instructions from the [Creating backups](#creating-backups) section can be restored using the `app:backup:restore` argument.

Before restoring a backup — stop and remove the running instance.

```bash
docker stop invoiceplane && docker rm invoiceplane
```

Relaunch the container with the `app:backup:restore` argument. Ensure you launch the container in the interactive mode `-it`.

```bash
docker run --name invoiceplane -it --rm [OPTIONS] \
  ruanbekker/invoiceplane:1.5.11 app:backup:restore
```

A list of existing backups will be displayed. Select a backup you wish to restore.

To avoid this interaction you can specify the backup filename using the `BACKUP` argument to `app:backup:restore`, eg.

```bash
docker run --name invoiceplane -it --rm [OPTIONS] \
  ruanbekker/invoiceplane:1.5.11 app:backup:restore BACKUP=1417624827_invoiceplane_backup.tar
```

### Upgrading

To upgrade to newer releases:

  1. Download the updated Docker image:

  ```bash
  docker pull ruanbekker/invoiceplane:1.5.11
  ```

  2. Stop the currently running image:

  ```bash
  docker stop invoiceplane
  ```

  3. Remove the stopped container

  ```bash
  docker rm -v invoiceplane
  ```

  4. Start the updated image

  ```bash
  docker run -name invoiceplane -itd \
    [OPTIONS] \
    ruanbekker/invoiceplane:1.5.11
  ```

Point your browser to [http://invoice.example.com:10080/setup](http://invoice.example.com:10080/setup) to complete the upgrade.

### Shell Access

For debugging and maintenance purposes you may want access the containers shell. If you are using Docker version `1.3.0` or higher you can access a running containers shell by starting `bash` using `docker exec`:

```bash
docker exec -it invoiceplane bash
```
