# Slurm Docker Cluster

This is a multi-container Slurm cluster using docker-compose.  The compose file
creates named volumes for persistent storage of MySQL data files as well as
Slurm state and log directories.

## Containers and Volumes

The compose file will run the following containers:

* mysql
* slurmdbd
* slurmctld
* c1 (slurmd)
* c2 (slurmd)

The compose file will create the following named volumes:

* etc_munge         ( -> /etc/munge     )
* etc_slurm         ( -> /etc/slurm     )
* slurm_jobdir      ( -> /data          )
* var_lib_mysql     ( -> /var/lib/mysql )
* var_log_slurm     ( -> /var/log/slurm )

## Building the Docker Image

Build the image locally:

```console
docker build -t slurm-docker-cluster:19.05.1 .
```

Build a different version of Slurm using Docker build args and the Slurm Git
tag:

```console
docker build --build-arg SLURM_TAG="slurm-19-05-2-1" -t slurm-docker-cluster:19.05.2
```

> Note: You will need to update the container image version in
> [docker-compose.yml](docker-compose.yml).

You may use the image pushed to dockerhub. You can use the one from the original author, [giovtorres/slurm-docker-cluster](https://hub.docker.com/r/giovtorres/slurm-docker-cluster). I also pushed an image, this is the one I will use here.

## Single node mode

### Starting the Cluster

Run `docker-compose` to instantiate the cluster:

```console
docker-compose up -d
```

### Register the Cluster with SlurmDBD

To register the cluster to the slurmdbd daemon, run the `register_cluster.sh`
script:

```console
./register_cluster.sh
```

> Note: You may have to wait a few seconds for the cluster daemons to become
> ready before registering the cluster.  Otherwise, you may get an error such
> as **sacctmgr: error: Problem talking to the database: Connection refused**.
>
> You can check the status of the cluster by viewing the logs: `docker-compose
> logs -f`

### Accessing the Cluster

Use `docker exec` to run a bash shell on the controller container:

```console
docker exec -it slurmctld bash
```

From the shell, execute slurm commands, for example:

```console
[root@slurmctld /]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up 5-00:00:00      2   idle c[1-2]
```

### Submitting Jobs

The `slurm_jobdir` named volume is mounted on each Slurm container as `/data`.
Therefore, in order to see job output files while on the controller, change to
the `/data` directory when on the **slurmctld** container and then submit a job:

```console
[root@slurmctld /]# cd /data/
[root@slurmctld data]# sbatch --wrap="uptime"
Submitted batch job 2
[root@slurmctld data]# ls
slurm-2.out
```

### Stopping and Restarting the Cluster

```console
docker-compose stop
docker-compose start
```

### Deleting the Cluster

To remove all containers and volumes, run:

```console
docker-compose stop
docker-compose rm -f
docker volume rm slurm-docker-cluster_etc_munge slurm-docker-cluster_etc_slurm slurm-docker-cluster_slurm_jobdir slurm-docker-cluster_var_lib_mysql slurm-docker-cluster_var_log_slurm
```

## Swarm mode

### Starting the cluster

In order to set up Swarm, first a manager node has to be initialized:

```console
sudo docker swarm init
```

This will give you the command to add workers (you can get this command again by doing `docker swarm join-token worker`), something like:

```console
   docker swarm join \
    --token XXXXXX-X-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XXXXXXXXXXXXXXXXXXXXXXXXX \
    192.168.1.6:2377
```

Run this in every worker you want to join the swarm. Afterwards, run `docker swarm deploy` in the manager node:

```console
docker stack deploy --compose-file docker-compose.yml slurm
```

`deploy` is also used to update the cluster.


### Register the Cluster with SlurmDBD

Run the following from the master node:

```console
CID=$(docker service ps slurm_slurmctld -q --no-trunc)
# This step must be run from the node that runs the container
docker exec slurm_slurmctld.1.${CID} bash -c "/usr/bin/sacctmgr --immediate add cluster name=linux" && \
docker stack deploy --compose-file docker-compose.yml slurm
```

Where:
* *slurm* is the name of the stack
* *1* is the replica number

### Accessing the Cluster

```console
CID=$(docker service ps slurm_slurmctld -q --no-trunc)
# SAme as before, the exec only works from the node that runs the container
docker exec -it slurm_slurmctld.1.${CID} bash
```

Same `slurm` commands should work as with a single node: 

```console
[root@slurmctld /]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up 5-00:00:00      2   idle c[1-2]
```

### Submitting Jobs

See single node "Submitting Jobs"

### Deleting the Cluster

```console
docker stack rm slurm
```
