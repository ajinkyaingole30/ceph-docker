# Containerize Ceph

### Daemon container
This Dockerfile may be used to bootstrap a Ceph cluster with all the Ceph daemons running. To run a certain type of daemon, simply use the name of the daemon as $1. Valid values are:
```
- mon deploys a Ceph monitor
- osd deploys an OSD using the method specified by OSD_TYPE
- osd_directory deploys one or multiple OSDs in a single container using a prepared directory (used in scenario where the operator doesn't want to use --privileged=true)
- osd_directory_single deploys an single OSD per container using a prepared directory (used in scenario where the operator doesn't want to use --privileged=true)
- osd_ceph_disk deploys an OSD using ceph-disk, so you have to provide a whole device (ie: /dev/sdb)
```

mds deploys a MDS
rgw deploys a Rados Gateway

### Usage
You can use this container to bootstrap any Ceph daemon.

CLUSTER is the name of the cluster (DEFAULT: ceph)
### SELinux
If SELinux is enabled, run the following commands:
```
sudo chcon -Rt svirt_sandbox_file_t /etc/ceph
sudo chcon -Rt svirt_sandbox_file_t /var/lib/ceph
```
### Deploy a monitor
Given that monitors can not communicate through a NATed network, we need to use the --net=host to expose Docker's host machine network stack:
```
$ sudo docker run -d --net=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph \
-e MON_IP=192.168.0.20 \
-e CEPH_PUBLIC_NETWORK=192.168.0.0/24 \
ceph/daemon:latest-mimic mon
```
Here are the options available to you.
```
MON_IP is the IP address of your host running Docker.
MON_NAME is the name of your monitor (DEFAULT: $(hostname)).
CEPH_PUBLIC_NETWORK is the CIDR of the host running Docker. It should be in the same network as the MON_IP.
CEPH_CLUSTER_NETWORK is the CIDR of a secondary interface of the host running Docker. Used for the OSD replication traffic.
```
### Deploy Object Storage Daemon

The current implementation allows you to run a single OSD process per container. Following the microservice mindset, we should not run more than one service inside our container. In our case, running multiple OSD processes into a single container breaks this rule and will likely introduce undesirable behaviors. This will also increase the setup and maintenance complexity of the solution.

In this configuration, the usage of --privileged=true is strictly required because we need full access to /dev/ and other kernel functions. However, we support another configuration based on exposing OSD directories where the operators will do the appropriate preparation of the devices. Then he or she will simply expose the OSD directory and populating (ceph-osd mkfs) the OSD will be done by the entry point. The configuration I'm presenting now is easier to start with because you only need to specify a block device and the entry point will do the rest.

Those who do not want to use --privileged=true, can fall back on the second example.
```
$ sudo docker run -d --net=host \
--privileged=true \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph \
-v /dev/:/dev/ \
-e OSD_DEVICE=/dev/vdd \
ceph-daemon:latest-mimic osd_ceph_disk
```
If you don't want to use --privileged=true you can always prepare the OSD by yourself with the help of your configuration management of your choice.

Example without a privileged mode, in this example we assume that you partitioned, put a filesystem and mounted the OSD partition. To create your OSDs simply run the following command:
```
$ sudo docker exec <mon-container-id> ceph osd create.
```
Then run your container like so:
```
docker run -v /osds/1:/var/lib/ceph/osd/ceph-1 -v /osds/2:/var/lib/ceph/osd/ceph-2
```
```
$ sudo docker run -d --net=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph \
-v /osds/1:/var/lib/ceph/osd/ceph-1 \
ceph-daemon:latest-mimic osd_disk_directory
```
Here are the options available to you.
```
OSD_DEVICE is the OSD device, ie: /dev/sdb
OSD_JOURNAL is the device that will be used to store the OSD's journal, ie: /dev/sdz
HOSTNAME is the hostname of the hostname of the container where the OSD runs (DEFAULT: $(hostname))
OSD_FORCE_ZAP will force zapping the content of the given device (DEFAULT: 0 and 1 to force it)
OSD_JOURNAL_SIZE is the size of the OSD journal (DEFAULT: 100)
```
### Deploy Metadata Server

This one is pretty straightforward and easy to bootstrap. The only caveat at the moment is that we require the Ceph admin key to be available in the Docker. This key will be used to create the CephFS pools and the filesystem.

If you run an old version of Ceph (prior to 0.87) you don't need this, but you might want to know since it's always best to run the latest version!
```
$ sudo docker run -d --net=host \
-v /var/lib/ceph/:/var/lib/ceph \
-v /etc/ceph:/etc/ceph \
-e CEPHFS_CREATE=1 \
ceph-daemon:latest-mimic mds
```
Here are the options available to you.
```
MDS_NAME is the name of the Metadata server (DEFAULT: mds-$(hostname)).
CEPHFS_CREATE will create a filesystem for your Metadata server (DEFAULT: 0 and 1 to enable it).
CEPHFS_NAME is the name of the Metadata filesystem (DEFAULT: cephfs).
CEPHFS_DATA_POOL is the name of the data pool for the Metadata Server (DEFAULT: cephfs_data).
CEPHFS_DATA_POOL_PG is the number of placement groups for the data pool (DEFAULT: 8).
CEPHFS_DATA_POOL is the name of the metadata pool for the Metadata Server (DEFAULT: cephfs_metadata).
CEPHFS_METADATA_POOL_PG is the number of placement groups for the metadata pool (DEFAULT: 8).
```
### Deploy RADOS gateway
For the RADOS gateway, we deploy it with civetweb enabled by default. However, it is possible to use different CGI frontends by simply giving remote address and port.
```
$ sudo docker run -d --net=host \
-v /var/lib/ceph/:/var/lib/ceph \
-v /etc/ceph:/etc/ceph \
ceph-daemon:latest-mimic rgw
```
