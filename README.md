# "Running Opencast in Containerized Environments" - Opencast Summit 2018

This repository contains presentation, notes, and deployment files for the "Running Opencast in Containerized Environments" workshop taking place at the Opencast Summit 2018 in Vienna.

**Note:** All commands should be run with root privileges.

## Using Docker Swarm

### Environment

This workshop uses four `n1-standard-4` VMs (4 vCPUs, 15 GB memory) from Google Compute Engine:

- **oc-01**<br>
  Internal IP: 10.132.0.100<br>
  Public IP: 35.205.36.231

- **oc-02**<br>
  Internal IP: 10.132.0.101<br>
  Public IP: 35.205.214.154

- **oc-03**<br>
  Internal IP: 10.132.0.102<br>
  Public IP: 35.189.209.85

- **oc-04**<br>
  Internal IP: 10.132.0.103<br>
  Public IP: 192.158.29.237

The following DNS records are configured:

Host                 | Type    | Target
-------------------- | ------- | -----------------
`oc.mtneug.de`       | `CNAME` | `oc-01.mtneug.de`
`oc-admin.mtneug.de` | `CNAME` | `oc-01.mtneug.de`
`oc-01.mtneug.de`    | `A`     | `35.205.36.231`
`oc-02.mtneug.de`    | `A`     | `35.205.214.154`
`oc-03.mtneug.de`    | `A`     | `35.189.209.85`
`oc-04.mtneug.de`    | `A`     | `192.158.29.237`

The NFS server and other administrative services will run on `oc-01`. Opencast, MariaDB, ActiveMQ, and nginx as proxy will run in separate containers with the following credentials:

- **Opencast**<br>
  User: `admin`<br>
  Password: `opencast`

- **MariaDB**<br>
  User: `opencast`<br>
  Password: `MARIADB_PW`

- **ActiveMQ**<br>
  User: `opencast`<br>
  Password: `ACTIVEMQ_PW`

### Basic setup

Update distribution and install basic packages:

```sh
oc-0x:$  apt update
oc-0x:$  apt dist-upgrade
oc-0x:$  apt install \
           apt-transport-https \
           ca-certificates \
           curl \
           gnupg2 \
           software-properties-common
```

(Optional): Who doesn't want to use the [fish shell](https://fishshell.com/)?

```sh
oc-0x:$  apt install fish
oc-0x:$  chsh -s /usr/bin/fish
```

### NFS

Install an NFS Server on `oc-01` end configure exports:

```sh
oc-01:$  apt install nfs-kernel-server
oc-01:$  mkdir -p /var/nfs/docker
oc-01:$  chmod 700 /var/nfs/docker
oc-01:$  echo "/var/nfs/docker 10.132.0.100(rw,no_root_squash,no_subtree_check) 10.132.0.101(rw,no_root_squash,no_subtree_check) 10.132.0.102(rw,no_root_squash,no_subtree_check) 192.158.29.237(rw,no_root_squash,no_subtree_check)" > /etc/exports
oc-01:$  exportfs -ra
```

Install the NFS client and mount the file system on all systems:

```sh
oc-0x:$  apt install nfs-common
oc-0x:$  echo "10.132.0.100:/var/nfs/docker /mnt/docker nfs4 rw 0 0" >> /etc/fstab
oc-0x:$  mkdir -p /mnt/docker
oc-0x:$  mount /mnt/docker
```

### Docker

[Configure](https://docs.docker.com/config/daemon/) Docker, especially the [storge driver](https://docs.docker.com/storage/storagedriver/select-storage-driver/):

```sh
oc-0x:$  mkdir -p /etc/docker
oc-0x:$  echo '{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "512m",
    "max-file": "3"
  },
  "max-concurrent-downloads": 5,
  "max-concurrent-uploads": 5,
  "init": true
}' > /etc/docker/daemon.json
```

[Install Docker](https://docs.docker.com/install/) depending on the your distribution:

```sh
oc-0x:$  curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
oc-0x:$  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian stretch stable"
oc-0x:$  apt update
oc-0x:$  apt install docker-ce
```

### Docker Swarm

Initialize a Swarm cluster:

```sh
oc-01:$  docker swarm init
```

Add all other nodes to the cluster:

```sh
oc-0x:$  docker swarm join --token SWMTKN-1-... 10.132.0.100:2377
```

Add labels to nodes:

```sh
oc-01:$  docker node update oc-01 --label-add "workload=normal"
oc-01:$  docker node update oc-02 --label-add "workload=normal"
oc-01:$  docker node update oc-03 --label-add "workload=heavy"
```

Install Weave net plugin

```sh
oc-0x:$  docker plugin install store/weaveworks/net-plugin:latest_release --grant-all-permissions
```

### Proxy

Copy configuration files:

```sh
local:$  git clone https://github.com/mtneug/containerized-opencast-workshop.git /tmp/workdir
```

Deploy proxy stack:

```sh
oc-01:$  cd /tmp/workdir/swarm/proxy
oc-01:$  docker stack deploy --prune -c docker-compose.yml proxy
```

### Opencast

Create folders for persistent storage:

```sh
oc-01:$  mkdir -p /mnt/docker/opencast-activemq-data
oc-01:$  chown 100:101 /mnt/docker/opencast-activemq-data

oc-01:$  mkdir -p /mnt/docker/opencast-mariadb-data
oc-01:$  chown 999:999 /mnt/docker/opencast-mariadb-data

oc-01:$  mkdir -p /mnt/docker/opencast-data
oc-01:$  chown 800:800 /mnt/docker/opencast-data
```

Deploy Opencast stack:

```sh
oc-01:$  cd /tmp/workdir/swarm/opencast
oc-01:$  docker stack deploy --prune -c docker-compose.yml opencast
```

## Using Kubernetes

### Environment

TODO:
