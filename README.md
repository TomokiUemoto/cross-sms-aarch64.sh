# Cross SMS aarch64 Shell

This software is [OpenHPC][1] utility which enables to create aarch64
initial build image (BOS) on System Management Server (SMS) x86_64,
and facilitates to deploy and manage Compute Nodes both x86_64 and
aarch64.

Typically SMS x86_64 exports x86_64 application as /opt/ohpc/pub, and
CN x86_64 mounts it on /opt/ohpc/pub.

And SMS aarch64 exports aarch64 application as /opt/ohpc/pub, and
CN aarch64 mounts it on /opt/ohpc/pub as well.


In case of crossing the architecture which means that SMS x86_64
serves not only CN x86_64 but also CN aarch64, this software helps to
install aarch64 application into /opt/ohpc-aarch64/opt/ohpc/pub and
sets up BOS so that CN aarch64 can mounts it on /opt/ohpc/pub.

[1]: https://github.com/openhpc/ohpc "OpenHPC"


## Prerequisites

SMS x86_64 requires the following softwares have been installed.

* OpenHPC 1.3.8 (11 June 2019) following [CentOS 7.6 x86_64 Install
  guide with Warewulf + Slurm][2]
* Docker docker-1.13.1-102.git7f2769b.el7.centos.x86_64

Type the following commands to verify the software versions:

```sh
[root@x86_64 ~]# docker version | grep Package
 Package version: docker-1.13.1-102.git7f2769b.el7.centos.x86_64
 Package version: docker-1.13.1-102.git7f2769b.el7.centos.x86_64

[root@x86_64 ~]# rpm -qa | grep ohpc-base
ohpc-base-1.3.8-3.1.ohpc.1.3.8.x86_64
```

[2]: https://github.com/openhpc/ohpc/releases/download/v1.3.8.GA/Install_guide-CentOS7-Warewulf-SLURM-1.3.8-x86_64.pdf "CentOS 7.6 x86_64 Install guide with Warewulf + Slurm"


## Build and Install

Type the following four commands, *sms_ip* is your SMS x86_64 IP
address.

If you are behind organization's firewall, set HTTP_PROXY and
HTTPS_PROXY environment variables.

```sh
[root@x86_64 ~]# git clone https://github.com/NaohiroTamura/cross-sms-aarch64.sh

[root@x86_64 ~]# cd cross-sms-aarch64.sh

[root@x86_64 cross-sms-aarch64.sh]# make

[root@x86_64 cross-sms-aarch64.sh]# make install sms_ip=XX.XX.XX.XX
```

### make

What *make* does is to  build docker container named *sms-aarch64.sh*.

*qemu-aarch64-static* binary is derived from Ubuntu package, since
it's static binary it can run on any Linux.

If you prefer to compile *qemu-aarch64-static* from QEMU source code,
put your compiled *qemu-aarch64-static* under *usr/bin* directory so
as not to download from Ubuntu.

```sh
[root@x86_64 cross-sms-aarch64.sh]# wget http://security.ubuntu.com/ubuntu/pool/universe/q/qemu/qemu-user-static_3.1+dfsg-2ubuntu3.1_amd64.deb
[root@x86_64 cross-sms-aarch64.sh]# ar p qemu-user-static_3.1+dfsg-2ubuntu3.1_amd64.deb data.tar.xz | tar Jxvf - ./usr/bin/qemu-aarch64-static

[root@x86_64 cross-sms-aarch64.sh]# docker build -f Dockerfile.centos7 -t sms-aarch64.sh .
```

### make install

What *make install* does is the following four steps:

1. setup binfmt_misc for aarch64
2. export /opt/ohpc-aarch64/opt/ohpc from master server to docker
   container
3. create Docker NFS volume and local volume
   * Docker volume has to be empty. If not, docker doesn't initialize
     the volume at the first time invocation. Otherwise it causes
     inconsistency.
4. Install docker client shell, sms-aarch64.sh

```sh
# 1. setup binfmt_misc for aarch64
[root@x86_64 cross-sms-aarch64.sh]# cp -p etc/binfmt.d/aarch64.conf /etc/binfmt.d
[root@x86_64 cross-sms-aarch64.sh]# systemctl restart systemd-binfmt
[root@x86_64 cross-sms-aarch64.sh]# ll /proc/sys/fs/binfmt_misc/aarch64
-rw-r--r-- 1 root root 0 Aug 22 11:04 /proc/sys/fs/binfmt_misc/aarch64

# 2. export /opt/ohpc-aarch64/opt/ohpc from master server to docker container
[root@x86_64 cross-sms-aarch64.sh]# mkdir -p /opt/ohpc-aarch64/opt/ohpc
[root@x86_64 cross-sms-aarch64.sh]# echo "/opt/ohpc-aarch64/opt/ohpc 172.17.0.0/16(rw,no_subtree_check,no_root_squash)" >> /etc/exports
[root@x86_64 cross-sms-aarch64.sh]# exportfs -ra

# 3. create Docker NFS volume and local volume
[root@x86_64 cross-sms-aarch64.sh]# docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=${sms_ip},rw,nfsvers=3 \
  --opt device=:/opt/ohpc-aarch64/opt/ohpc ohpc-aarch64
[root@x86_64 cross-sms-aarch64.sh]# docker volume create yum-aarch64
[root@x86_64 cross-sms-aarch64.sh]# docker volume ls
DRIVER              VOLUME NAME
local               ohpc-aarch64
local               yum-aarch64

# 4. Install docker client shell
[root@x86_64 cross-sms-aarch64.sh]# install -o root -g root sms-aarch64.sh /usr/local/bin
```

## Usage

*sms-aarch64.sh* is mainly used for two purposes:

1. create aarch64 BOS image onto SMS x86_64 file system
2. install aarch64 OHPC application into SMS x86_64 file system

Heading title hereinafter refers to the section of OpenHPC 1.3.8
(11 June 2019) [CentOS 7.6 aarch64 Install guide with Warewulf +
Slurm][3].

Please notice that the section number of [CentOS 7.6 aarch64 Install
guide with Warewulf + Slurm][3] is slightly different from [CentOS
7.6 x86_64 Install guide with Warewulf + Slurm][2].

[3]:  https://github.com/openhpc/ohpc/releases/download/v1.3.8.GA/Install_guide-CentOS7-Warewulf-SLURM-1.3.8-aarch64.pdf "CentOS 7.6 aarch64 Install guide with Warewulf + Slurm"

### 3.1 Enable OpenHPC repository for local use

The ohpc-release package has been already installed onto CentOS
7.6.1810 container. Please take a look at Dockerfile.centos7.

### 3.2 Installation template

You can install the OpenHPC documentation package (docs-ohpc) into SMS
x86_64 instead of SMS aarch64 container.

### 3.3 Add provisioning services on master node

The base meta-packages have been already installed into the
container. Please take a look at Dockerfile.centos7.

### 3.4 Add resource management services on master node

The slurm server running on SMS x86_64 is used to serve to CN
aarch64. So you don't have to install the slurm server meta-package
into the container.

### 3.5 Complete basic Warewulf setup for master node

The warewulf should have been already set up on SMS x86_64. So you
don't have to set it up in the container.

### 3.6 Deﬁne compute image for provisioning

In order to create aarch64 BOS Image, you need to interact with
*sms-aarch64.sh* container. Please notice that the difference of the
prompts between *[root@x86_64 ~]$* and *[root@aarch64 /]#*

```sh
[root@x86_64 ~]$ sms-aarch64.sh
[root@aarch64 /]# export CHROOT=/opt/ohpc/admin/images/centos7.6
[root@aarch64 /]# mkdir -p $CHROOT/usr/bin
[root@aarch64 /]# cp -p /usr/bin/qemu-aarch64-static $CHROOT/usr/bin
[root@aarch64 /]# wwmkchroot centos-7 $CHROOT
[root@aarch64 /]# yum -y --installroot=$CHROOT install ohpc-base-compute
[root@aarch64 /]# cp -p /etc/resolv.conf $CHROOT/etc/resolv.conf
[root@aarch64 /]# yum -y --installroot=$CHROOT install ohpc-slurm-client
[root@aarch64 /]# yum -y --installroot=$CHROOT install ntp
[root@aarch64 /]# yum -y --installroot=$CHROOT install kernel
[root@aarch64 /]# yum -y --installroot=$CHROOT install lmod-ohpc
[root@aarch64 /]# exit
```

The warewulf database running on SMS x86_64, so you don't have to
anything to the container.

Please notice that the path */opt/ohpc/admin/images/centos7.6* in the
container is equivalent to the path
*/opt/ohpc-aarch64/opt/ohpc/admin/images/centos7.6* on SMS x86_64.

The environment variable *AARCH64_CHROOT* is chosen to prevent from
mixing up *CHROOT* inside the container with *CHROOT* outside the
container.

```sh
[root@x86_64 ~]# export AARCH64_CHROOT=/opt/ohpc-aarch64/opt/ohpc/admin/images/centos7.6

# Put ssh public key
[root@x86_64 ~]# cat ~/.ssh/cluster.pub >> $AARCH64_CHROOT/root/.ssh/authorized_keys

# Add NFS client mounts of /home and /opt/ohpc/pub to base image
[root@x86_64 ~]# echo "${sms_ip}:/home /home nfs nfsvers=3,nodev,nosuid 0 0" >> $AARCH64_CHROOT/etc/fstab
[root@x86_64 ~]# echo "${sms_ip}:/opt/ohpc-aarch64/opt/ohpc/pub /opt/ohpc/pub nfs nfsvers=3,nodev 0 0" >> $AARCH64_CHROOT/etc/fstab

# Export /home and OpenHPC public packages from master server
[root@x86_64 ~]# echo "/opt/ohpc-aarch64/opt/ohpc/pub *(ro,no_subtree_check,fsid=12)" >> /etc/exports
[root@x86_64 ~]# exportfs -ra

# Enable NTP time service on computes and identify master host as local NTP server
[root@x86_64 ~]# chroot $AARCH64_CHROOT systemctl enable ntpd
[root@x86_64 ~]# echo "server ${sms_ip}" >> $AARCH64_CHROOT/etc/ntp.conf
```

### 3.7 Finalizing provisioning conﬁguration

In order to create aarch64 bootstrap, please make sure to install
*warewulf-provision-initramfs-aarch64-ohpc* package into SMS x86_64.

The kernel version of the aarch64 OBS image is different from the
kernel version of SMS x86_64. So please check the version as follows.

The bootstrap image and Virtual Node File System (VNFS) image created
on x86_64 has ARCH attribute *x86_64* respectively. So please update
the ARCH as follows.

```sh
[root@x86_64 ~]# yum install -y warewulf-provision-initramfs-aarch64-ohpc

[root@x86_64 ~]# export WW_CONF=/etc/warewulf/bootstrap.conf
[root@x86_64 ~]# echo "drivers += updates/kernel/" >> $WW_CONF
[root@x86_64 ~]# echo "drivers += overlay" >> $WW_CONF

# check the kernel version of the aarch64 OBS image
[root@x86_64 ~]# chroot $AARCH64_CHROOT rpm -qa | grep kernel
kernel-4.14.0-115.10.1.el7a.aarch64
kernel-headers-4.14.0-115.10.1.el7a.aarch64

# specifty the kernel version
[root@x86_64 ~]# wwbootstrap --chroot $AARCH64_CHROOT 4.14.0-115.8.1.el7a.aarch64

# Notice that ARCH is x86_64
[root@x86_64 ~]# wwsh bootstrap list
BOOTSTRAP NAME            SIZE (M)      ARCH
4.14.0-115.8.1.el7a.aarch64 23.0          x86_64

# Update the ARCH
[root@x86_64 ~]# wwsh bootstrap set -y 4.14.0-115.8.1.el7a.aarch64 -a aarch64

# make sure that ARCH is updated to aarch64
[root@x86_64 ~]# wwsh bootstrap list
BOOTSTRAP NAME            SIZE (M)      ARCH
4.14.0-115.8.1.el7a.aarch64 23.0         aarch64

# Assemble Virtual Node File System (VNFS) image
[root@x86_64 ~]# wwvnfs --chroot $AARCH64_CHROOT

# Notice that ARCH is x86_64
[root@x86_64 ~]# wwsh vnfs list
VNFS NAME            SIZE (M)   ARCH       CHROOT LOCATION
centos7.6-aarch64    277.7      x86_64     /opt/ohpc-aarch64/opt/ohpc/admin/images/centos7.6

# Update the ARCH
[root@x86_64 ~]# wwsh vnfs set -y centos7.6-aarch64 -a aarch64

# make sure that ARCH is updated to aarch64
[root@x86_64 ~]# wwsh vnfs list
VNFS NAME            SIZE (M)   ARCH       CHROOT LOCATION
centos7.6-aarch64    277.7      aarch64    /opt/ohpc-aarch64/opt/ohpc/admin/images/centos7.6

# Add nodes to Warewulf data store
[root@x86_64 /]# wwsh node new ${c_name} --arch=aarch64 --ipaddr=${c_ip} --hwaddr=${c_mac} -D ${eth_provision}

# Define provisioning image for hosts
[root@x86_64 /]# wwsh provision set "${compute_regex}" --vnfs=centos7.6-aarch64 --bootstrap=4.14.0-115.8.1.el7a.aarch64 \
--files=dynamic_hosts,passwd,group,shadow,slurm.conf,munge.key,network

# Define provisioning image for hosts
[root@x86_64 /]# wwsh -y provision set em1n1 --kargs="net.ifnames=0 biosdevname=0 console=ttyAMA0,115200 rd.debug"

# Restart dhcp / update PXE
[root@x86_64 /]# systemctl restart dhcpd
[root@x86_64 /]# wwsh pxe update
```

### 3.8 Boot compute nodes

Boot the CN aarch64 via IPMI as CN x86_64.

### 4.1 Development Tools

```sh
[root@x86_64 ~]$ sms-aarch64.sh yum -y install ohpc-autotools
[root@x86_64 ~]$ sms-aarch64.sh yum -y install EasyBuild-ohpc
[root@x86_64 ~]$ sms-aarch64.sh yum -y install hwloc-ohpc
[root@x86_64 ~]$ sms-aarch64.sh yum -y install spack-ohpc
[root@x86_64 ~]$ sms-aarch64.sh yum -y install valgrind-ohpc
```
### 4.2 Compilers

```sh
[root@x86_64 ~]$ sms-aarch64.sh yum -y install gnu8-compilers-ohpc
[root@x86_64 ~]$ sms-aarch64.sh yum -y install llvm5-compilers-ohpc
```

### 4.3 MPI Stacks

```sh
[root@x86_64 ~]$ sms-aarch64.sh yum -y install openmpi3-gnu8-ohpc mpich-gnu8-ohpc
```

### 4.4 Performance Tools

```sh
[root@x86_64 ~]$ sms-aarch64.sh yum -y install ohpc-gnu8-perf-tools
[root@x86_64 ~]$ sms-aarch64.sh yum -y install lmod-defaults-gnu8-openmpi3-ohpc
```

### 4.5 Setup default development environment

```sh
[root@x86_64 ~]$ sms-aarch64.sh yum -y install lmod-defaults-gnu8-openmpi3-ohpc
```

### 4.6 3rd Party Libraries and Tools

```sh
[root@x86_64 ~]$ sms-aarch64.sh yum -y install ohpc-gnu8-serial-libs
[root@x86_64 ~]$ sms-aarch64.sh yum -y install ohpc-gnu8-io-libs
[root@x86_64 ~]$ sms-aarch64.sh yum -y install ohpc-gnu8-python-libs
[root@x86_64 ~]$ sms-aarch64.sh yum -y install ohpc-gnu8-runtimes

[root@x86_64 ~]$ sms-aarch64.sh yum -y install ohpc-gnu8-mpich-parallel-libs
[root@x86_64 ~]$ sms-aarch64.sh yum -y install ohpc-gnu8-openmpi3-parallel-libs
```