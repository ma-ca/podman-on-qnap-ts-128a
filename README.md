# Run podman on QNAP TS-128A

The QNAP TS-128A has an arm64 CPU.

Compile podman with statically linked binary of Podman for aarch64 / arm64.

In this document docker and qemu-user-static are used to compile a aarch64 binary on a x86_64 Ubuntu 20.10 VM.

## install docker on x86_64 Ubuntu 20.04 LTS

The follwing steps are run as root.

https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

```
apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
```

## build docker image with dependencies to compile podman

https://podman.io/getting-started/installation#building-from-scratch

Create a docker image with qemu-user-static that executes an aarch64 / arm64 binary on x86_64 host. The docker image is based on golang and includes all dependencies that are required to compile podman.

Using https://github.com/multiarch/qemu-user-static

```
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

Dockerfile-go-podman

```
FROM arm64v8/golang:1.17.5-bullseye

RUN apt-get update -qq \
 && apt-get install -yq --no-install-suggests --no-install-recommends \
      software-properties-common \
      build-essential \
      git \
      wget \
      curl \
      btrfs-progs \
      go-md2man \
      iptables \
      libassuan-dev \
      libbtrfs-dev \
      libc6-dev \
      libdevmapper-dev \
      libglib2.0-dev \
      libgpgme-dev \
      libgpg-error-dev \
      libprotobuf-dev \
      libprotobuf-c-dev \
      libseccomp-dev \
      libselinux1-dev \
      libsystemd-dev \
      pkg-config \
      runc \
      uidmap \
 && /bin/rm -rf /var/lib/apt/lists/*
```

Dockerfile-gcc-conmon

```
FROM arm64v8/debian:bullseye

RUN apt-get update -qq \
 && apt-get install -yq --no-install-suggests --no-install-recommends \
      gcc \
      git \
      libc6-dev \
      libglib2.0-dev \
      libseccomp-dev \
      pkg-config \
      make \
      runc \
 && /bin/rm -rf /var/lib/apt/lists/*
```

```
docker build -t go-podman-aarch64 -f Dockerfile-go-podman .
docker build -t gcc-conmon-aarch64 -f Dockerfile-gcc-conmon .
```

## compile podman

On Ubuntu run `git clone` and then `docker run` to compile podman.

```
git clone https://github.com/containers/podman.git -b v4.3.1
git clone https://github.com/containers/conmon.git -b v2.1.5

docker run -ti --rm -v $(pwd)/podman:/podman -w /podman \
-e CFLAGS='-static -pthread' \
-e LDFLAGS='-s -w -static-libgcc -static' \
-e EXTRA_LDFLAGS='-s -w -linkmode external -extldflags "-static -lm"' \
-e BUILDTAGS='static netgo osusergo exclude_graphdriver_btrfs exclude_graphdriver_devicemapper seccomp apparmor selinux' \
-e CGO_ENABLED=1 \
go-podman-aarch64 make

docker run -ti --rm -v $(pwd)/conmon:/conmon -w /conmon \
-e CFLAGS='-static -pthread' \
-e LDFLAGS='-s -w -static-libgcc -static' \
-e EXTRA_LDFLAGS='-s -w -linkmode external -extldflags "-static -lm"' \
-e CGO_ENABLED=1 \
gcc-conmon-aarch64 make
```

Copy the `podman` binary from `podman/bin` and the `conmon` binary from `conmon/bin` to TS-128A.

## prepare TS-128A

On the TS-128A download the additional precompiled binary files.

- crun 1.4.3 https://github.com/containers/crun/releases/
- cni-plugins v1.1.0 https://github.com/containernetworking/plugins/releases/

```
curl -L -o crun https://github.com/containers/crun/releases/download/1.4.3/crun-1.4.3-linux-arm64-disable-systemd

curl -sL https://github.com/containernetworking/plugins/releases/download/v1.1.0/cni-plugins-linux-arm64-v1.1.0.tgz | \
tar xzf -
```

On the TS-128A download config files

https://podman.io/getting-started/installation

https://github.com/containers/podman/tree/main/cni

```
mkdir -p /etc/cni/net.d
curl -qsSL https://raw.githubusercontent.com/containers/libpod/master/cni/87-podman-bridge.conflist | tee /etc/cni/net.d/87-podman-bridge.conflist

mkdir -p /etc/containers
curl -L -o /etc/containers/registries.conf https://src.fedoraproject.org/rpms/containers-common/raw/main/f/registries.conf
curl -L -o /etc/containers/policy.json https://src.fedoraproject.org/rpms/containers-common/raw/main/f/default-policy.json
curl -L -o /etc/containers/storage.conf https://github.com/containers/storage/raw/main/storage.conf
```

edit storage.conf and change `graphroot` to a directory of your choice.

```
graphroot = "/share/CACHEDEV1_DATA/containers/storage"
```

Create `graphroot` directory (make sure the disk space is sufficient to store podman images).

```
mkdir -p /share/CACHEDEV1_DATA/containers/storage
mkdir -p /share/CACHEDEV1_DATA/containers/tmp
```

```
export PATH=$PATH:$(pwd)
export TMPDIR=/share/CACHEDEV1_DATA/containers/tmp

[~] # podman --version
podman version 4.0.0-dev

[~] # conmon --version
conmon version 2.0.31
commit: 7e7eb74e52abf65a6d46807eeaea75425cc8a36c-dirty

[~] # crun --version
crun version 1.3
commit: 4f6c8e0583c679bfee6a899c05ac6b916022561b
spec: 1.0.0
+SELINUX +APPARMOR +CAP +SECCOMP +EBPF +YAJL
```

# compile QNAP kernel module

Note: podman needs to run with `--net host` if the kernel module `xt_comment` is missing, https://github.com/containers/podman/issues/4281

kernel compile with CONFIG_NETFILTER_XT_MATCH_COMMENT

## get TS-128A configuration and download kernel sources

- compile kernel module:
https://forum.qnapclub.de/thread/11807-howto-buildumgebung-kernelmodule-für-qnap-bauen/?postID=356787#post356787


https://sourceforge.net/projects/qosgpl/

TS-128A firmware version 4.5.4.1800 build 20210923

```
[~] # strings /lib/libc* | grep GCC | uniq
GCC: (Linaro GCC 5.3-2016.02) 5.3.1 20160113

[~] # strings /lib/libc.so.6 |grep 'GNU C'
GNU C Library (GNU libc) stable release version 2.23, by Roland McGrath et al.
Compiled by GNU CC version 5.3.1 20160113.

[~] # cat /proc/version
Linux version 4.2.8 (root@U16BuildServer54) (gcc version 5.3.1 20160113 (Linaro GCC 5.3-2016.02) ) #1 SMP Thu Sep 23 01:49:16 CST 2021

[~] # uname -a
Linux NAS31C658 4.2.8 #1 SMP Thu Sep 23 01:49:16 CST 2021 aarch64 GNU/Linux
```

TS-128A firmware version 5.0.0.1932 build 20220129

```
[~] # strings /lib/libc* | grep GCC | uniq
GCC: (Linaro GCC 5.3-2016.02) 5.3.1 20160113

[~] # strings /lib/libc.so.6 |grep 'GNU C'
GNU C Library (GNU libc) stable release version 2.23, by Roland McGrath et al.
Compiled by GNU CC version 5.3.1 20160113.

[~] # cat /proc/version
Linux version 4.2.8 (root@U16BuildServer106) (gcc version 5.3.1 20160113 (Linaro GCC 5.3-2016.02) ) #1 SMP Sat Jan 29 00:59:30 CST 2022

[~] # uname -a
Linux NAS31C658 4.2.8 #1 SMP Sat Jan 29 00:59:30 CST 2022 aarch64 GNU/Linux
```

## on Ubuntu 20.04 compile kernel module with docker image

Run the follwing steps on Ubuntu 20.10 LTS

```
mkdir -p build/kernel build/qts build/compiled
cd build/kernel/
curl -LO https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.2.8.tar.gz
tar xzf linux-4.2.8.tar.gz

cd ../qts/
# download GPL_QTS-4.5.4-20210802_Kernel.tar.gz from https://sourceforge.net/projects/qosgpl/

tar xzf GPL_QTS-4.5.4-20210802_Kernel.tar.gz GPL_QTS/kernel_cfg/TS-X28A/linux-4.2-arm64.config

cp GPL_QTS/kernel_cfg/TS-X28A/linux-4.2-arm64.config ../kernel/linux-4.2.8/.config

cd ../kernel/linux-4.2.8/

grep CONFIG_NETFILTER_XT_MATCH_COMMENT .config
# CONFIG_NETFILTER_XT_MATCH_COMMENT is not set
```

edit `.config`

```
sed -i "/CONFIG_NETFILTER_XT_MATCH_COMMENT/ s/^.*$/CONFIG_NETFILTER_XT_MATCH_COMMENT=m/" .config

grep CONFIG_NETFILTER_XT_MATCH_COMMENT .config
CONFIG_NETFILTER_XT_MATCH_COMMENT=m
```

Create docker image with gcc 5.3.1 and glibc 2.23

Dockerfile-gcc-5.3.1-aarch64

```
FROM arm64v8/ubuntu:16.04

RUN apt-get update -qq \
 && apt-get install -y --allow-downgrades \
      gcc \
      gcc-5=5.3.1-14ubuntu2 \
      binutils \
      build-essential \
      bc \
      bison \
      flex \
      libssl-dev \
      libelf-dev \
      cpp-5=5.3.1-14ubuntu2 \
      gcc-5-base=5.3.1-14ubuntu2 \
      libgcc-5-dev=5.3.1-14ubuntu2 \
      libcc1-0=5.3.1-14ubuntu2 \
      libgomp1=5.3.1-14ubuntu2 \
      libitm1=5.3.1-14ubuntu2 \
      libatomic1=5.3.1-14ubuntu2 \
      libasan2=5.3.1-14ubuntu2 \
      libubsan0=5.3.1-14ubuntu2 \
      libstdc++6=5.3.1-14ubuntu2 \
      g++ \
      g++-5=5.3.1-14ubuntu2 \
      libstdc++-5-dev=5.3.1-14ubuntu2 \
 && /bin/rm -rf /var/lib/apt/lists/*
```

```
docker build -t ubuntu-aarch64:xenial -f Dockerfile-gcc-5.3-aarch64 .

cd

docker run -ti --rm -v $(pwd)/build:/build -w /build ubuntu-aarch64:xenial bash
```

```
cd kernel/linux-4.2.8/

make oldconfig && make prepare & make scripts

make -C . M=net/netfilter
```

or compile in `docker run`

```
docker run -ti --rm --name compile -v $(pwd)/build:/build -w /build \
ubuntu-aarch64:xenial \
bash -c \
"cd kernel/linux-4.2.8/ \
make oldconfig && make prepare && make scripts && \
make -C . M=net/netfilter"
```

```
cd linux-4.2.8/net/netfilter/

tar cf ~/build/compiled/netfilter_ko.tar *.ko

tar cf netfilter_ko.tar -C linux-4.2.8/net/netfilter/ *.ko
```

## Copy tar to TS-128A and load module xt_comment

```
tar xvf netfilter_ko.tar xt_comment.ko
mv xt_comment.ko /mnt/HDA_ROOT/lib/modules/4.2.8/

insmod /mnt/HDA_ROOT/lib/modules/4.2.8/xt_comment.ko
```

or

```
mkdir -p /share/CACHEDEV1_DATA/.qpkg/Podman/bin
mkdir -p /share/CACHEDEV1_DATA/.qpkg/Podman/modules/4.2.8/

tar xvf netfilter_ko.tar -C /share/CACHEDEV1_DATA/.qpkg/Podman/modules/4.2.8/ xt_comment.ko
ln -s /share/CACHEDEV1_DATA/.qpkg/Podman/modules/4.2.8/xt_comment.ko /mnt/HDA_ROOT/lib/modules/4.2.8/xt_comment.ko

insmod /mnt/HDA_ROOT/lib/modules/4.2.8/xt_comment.ko
```

## Copy binary files to TS-128A

Copy binary files to `/share/CACHEDEV1_DATA/.qpkg/Podman/bin`
- `podman`
- `conmon`
- `crun`
- cni-plugins

```
mkdir -p /share/CACHEDEV1_DATA/.qpkg/Podman/bin

cp podman conmon crun /share/CACHEDEV1_DATA/.qpkg/Podman/bin/

cd /share/CACHEDEV1_DATA/.qpkg/Podman/bin/

ln -s $(pwd)/podman /usr/local/bin/
ln -s $(pwd)/conmon /usr/local/bin/
ln -s $(pwd)/crun /usr/local/bin/

mkdir -p /usr/local/libexec/cni

curl -sL https://github.com/containernetworking/plugins/releases/download/v1.1.0/cni-plugins-linux-arm64-v1.1.0.tgz | \
tar xzf - -C /usr/local/libexec/cni
```

## Create config

```
mkdir -p /share/CACHEDEV1_DATA/.qpkg/Podman/config

cd /share/CACHEDEV1_DATA/.qpkg/Podman/config

curl -qsSL https://raw.githubusercontent.com/containers/libpod/master/cni/87-podman-bridge.conflist | tee config/87-podman-bridge.conflist

curl -L -o config/registries.conf https://src.fedoraproject.org/rpms/containers-common/raw/main/f/registries.conf
curl -L -o config/policy.json https://src.fedoraproject.org/rpms/containers-common/raw/main/f/default-policy.json
curl -L -o config/storage.conf https://github.com/containers/storage/raw/main/storage.conf

mkdir -p /etc/cni/net.d
mkdir -p /etc/containers

ln -s $(pwd)/config/87-podman-bridge.conflist /etc/cni/net.d

ln -s $(pwd)/config/registries.conf /etc/containers/
ln -s $(pwd)/config/policy.json /etc/containers/
ln -s  $(pwd)/config/storage.conf /etc/containers/
```

edit storage.conf and change `graphroot` to a directory of your choice.

`graphroot` = `"/share/CACHEDEV1_DATA/containers/storage"`

```
sed -i 's#graphroot = .*#graphroot = "/share/CACHEDEV1_DATA/containers/storage"#' config/storage.conf
```

Create `graphroot` directory (make sure the disk space is sufficient to store podman images).

```
mkdir -p /share/CACHEDEV1_DATA/containers/storage
mkdir -p /share/CACHEDEV1_DATA/containers/tmp
```


# run Podman on TS-128A

```
[~] # podman run --name httpd --rm -p 8083:80 -d docker.io/library/httpd
Your kernel does not support pids limit capabilities or the cgroup is not mounted. PIDs limit discarded.

# -or-
[~] # podman run --cgroups disabled --name httpd --rm -p 8083:80 -d docker.io/library/httpd

[~] # curl localhost:8083
<html><body><h1>It works!</h1></body></html>

[~] # podman stop httpd
[~] # podman rm httpd
```

# run nginx in Podman

```
podman pull docker.io/library/nginx:alpine

podman run --cgroups disabled --rm --name nginx -p 8090:80 \
-v /some/content:/usr/share/nginx/html \
-v $(pwd)/default.conf:/etc/nginx/conf.d/default.conf \
-d \
docker.io/library/nginx:alpine
```

/etc/nginx/conf.d/default.conf

```
server {
    listen       80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```
