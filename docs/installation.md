This document walks you through setting up an environment to run both normal
containers and `urunc`-based ones, including the installation of all `urunc`
supported guests and VM/Sandbox monitors.

We assume a clean Ubuntu 22.04 environment, though urunc works on several other
Linux distributions as well.

The installation guide is plitted in three parts:

1. Installation of common container tools and containerd configuration. In
   particualr, the following components will get installed and configured:

  - [runc](https://github.com/opencontainers/runc)
  - [containerd](https://github.com/containerd/containerd/)
  - [CNI plugins](https://github.com/containernetworking/plugins)
  - [nerdctl](https://github.com/containerd/nerdctl)
  - [devmapper](https://docs.docker.com/storage/storagedriver/device-mapper-driver/) or [blockfile](https://github.com/containerd/containerd/blob/main/docs/snapshotters/blockfile.md)
  
2. Installation of all supported monitors and additional tools. Specifically:

  - [virtiofs](https://virtio-fs.gitlab.io/)
  - [solo5-{hvt|spt}](https://github.com/Solo5/solo5)
  - [qemu](https://www.qemu.org/)
  - [firecracker](https://github.com/firecracker-microvm/firecracker)

2. Installation and configuration of `urunc`. For theis step, we will install:

  - git, make
  - [Go [[ versions.go ]]](https://go.dev/doc/install)
  - [urunc](https://github.com/urunc-dev/urunc)

> Note: Some steps may overwrite existing tools or services.

Let's go.

## Step 1: Install container related components and tools

We install the container-related components and tools that
provide the foundation for running both standard containers and urunc workloads.
If your system already has a functioning container setup with the necessary
tools, you can safely skip this part.

### Install runc or any other generic container runtime

In case of a Kubernetes environment, `urunc` delegates the management of normal
containers (such as pause and sidecar containers) to a typical low level
continaer runtime (e.g. runc, crun). For that purpose, we choose to use `runc`
`runc`, but any other low level container runtime can be used too.  We can build
runc from source [following the instructions in `runc`'s
repository](https://github.com/opencontainers/runc/tree/main#building) or
download the latest binary:

```bash
RUNC_VERSION=$(curl -L -s -o /dev/null -w '%{url_effective}' "https://github.com/opencontainers/runc/releases/latest" | grep -oP "v\d+\.\d+\.\d+" | sed 's/v//')
wget -q https://github.com/opencontainers/runc/releases/download/v$RUNC_VERSION/runc.$(dpkg --print-architecture)
sudo install -m 755 runc.$(dpkg --print-architecture) /usr/local/sbin/runc
rm -f ./runc.$(dpkg --print-architecture)
```

### Install containerd

For the time being, `urunc` has been properly tested with
[containerd](https://github.com/containerd/containerd) as a high-level runtime.
Therefor,e we will install it by fetching its latest release. For alternative
installation methods or other information, please check containerd's [Getting
Started](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
guide.

```bash
CONTAINERD_VERSION=$(curl -L -s -o /dev/null -w '%{url_effective}' "https://github.com/containerd/containerd/releases/latest" | grep -oP "v\d+\.\d+\.\d+" | sed 's/v//')
wget -q https://github.com/containerd/containerd/releases/download/v$CONTAINERD_VERSION/containerd-$CONTAINERD_VERSION-linux-$(dpkg --print-architecture).tar.gz
sudo tar Cxzvf /usr/local containerd-$CONTAINERD_VERSION-linux-$(dpkg --print-architecture).tar.gz
rm -f containerd-$CONTAINERD_VERSION-linux-$(dpkg --print-architecture).tar.gz
```

#### Install containerd service

We cna configure [containerd](https://github.com/containerd/containerd) to start
automatically at boot time with [systemd](https://systemd.io/). For that
purpose,w e will also setup the respective systemd service:.

```bash
CONTAINERD_VERSION=$(curl -L -s -o /dev/null -w '%{url_effective}' "https://github.com/containerd/containerd/releases/latest" | grep -oP "v\d+\.\d+\.\d+" | sed 's/v//')
wget -q https://raw.githubusercontent.com/containerd/containerd/v$CONTAINERD_VERSION/containerd.service
sudo rm -f /lib/systemd/system/containerd.service
sudo mv containerd.service /lib/systemd/system/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

#### Configure containerd

In order to configure `urunc` and other snapshotters to work well under
[containerd](https://github.com/containerd/containerd), we will use as a base
its default configuration.

```bash
sudo mkdir -p /etc/containerd/
sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak # There might be no existing configuration.
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
```

> We can easily migrate containerd's configuration from older to newer versions
> with `sudo containerd config migrate > /etc/containerd/config.toml`

### Block-based snapshotters

`urunc` can leverage the block-based snapshots of some snapshotters and directly
use the contianer's snapshot as a block device for a guest.  Currently, we have
tested and verified that `urunc` works properly with
[devmapper](https://github.com/containerd/containerd/blob/main/docs/snapshotters/devmapper.md)
and
[blockfile](https://github.com/containerd/containerd/blob/main/docs/snapshotters/blockfile.md).
Devmapper uses a thinpool for flexible management, while blockfile relies on a
pre-allocated scratch file, though it lacks ext2 support and thus isn’t
compatible with Rumprun unikernels.  Therefore, if we want to make use of this
feature, we will need to setup and configure one of these block-based
snapshotters or even both of them.

#### Setting and configuring devmapper

The first step for the devmapper snapshotter is to create a thinpool. We can
easily do that using the [respective scripts in urunc's
repository](https://github.com/urunc-dev/urunc/tree/main/script).

```bash
git clone https://github.com/urunc-dev/urunc.git
sudo mkdir -p /usr/local/bin/scripts
sudo mkdir -p /usr/local/lib/systemd/system/
sudo cp urunc/script/dm_create.sh /usr/local/bin/scripts/dm_create.sh
sudo cp urunc/script/dm_reload.sh /usr/local/bin/scripts/dm_reload.sh
sudo chmod 755 /usr/local/bin/scripts/dm_create.sh
sudo chmod 755 /usr/local/bin/scripts/dm_reload.sh
```

The above scripts create and reload respectively a thinpool that will be used
for the devmapper snapshotter. Therefore, to create the thinpool, we can run:

```bash
sudo /usr/local/bin/scripts/dm_create.sh
```

However, when the system reboots, we will need to reload the thinpool with:

```bash
sudo /usr/local/bin/scripts/dm_reload.sh
```

Alternatively, we can setup a [systemd](https://systemd.io/).service which automatically reloads the existing thinpool when a system reboots. In the same directory with the above scripts, `urunc`'s repository includes such a service:

```bash
sudo cp urunc/script/dm_reload.service /usr/local/lib/systemd/system/dm_reload.service
sudo chmod 644 /usr/local/lib/systemd/system/dm_reload.service
sudo chown root:root /usr/local/lib/systemd/system/dm_reload.service
sudo systemctl daemon-reload
sudo systemctl enable dm_reload.service
```

At last, we need to change containerd's configuration.

- In containerd v2.x:

```toml
[plugins.'io.containerd.snapshotter.v1.devmapper']
  pool_name = "containerd-pool"
  root_path = "/var/lib/containerd/io.containerd.snapshotter.v1.devmapper"
  base_image_size = "10GB"
  discard_blocks = true
  fs_type = "ext2"
```

- In containerd v1.x:

```bash
[plugins."io.containerd.snapshotter.v1.devmapper"]
  pool_name = "containerd-pool"
  root_path = "/var/lib/containerd/io.containerd.snapshotter.v1.devmapper"
  base_image_size = "10GB"
  discard_blocks = true
  fs_type = "ext2"
```

We can verify that the devmapper snapshotter is properly configured with:

```bash
$ sudo ctr plugin ls | grep devmapper
io.containerd.snapshotter.v1              devmapper                linux/amd64    ok
```

#### Setting and configuring blockfile

Setting up `blockfile` is straightforward.

- At first, we need to provide a scratch file, which will be used from the
  blockfile snapshotter:

```bash
sudo mkdir -p /opt/containerd/blockfile
sudo dd if=/dev/zero of=/opt/containerd/blockfile/scratch bs=1M count=500
sudo mkfs.ext4 /opt/containerd/blockfile/scratch
sudo chown -R root:root /opt/containerd/blockfile
```

- Then, we need to update containerd's configuration:

  -  In containerd v2.x:

```toml
[plugins.'io.containerd.snapshotter.v1.blockfile']
  fs_type = "ext4"
  mount_options = []
  recreate_scratch = true
  root_path = "/var/lib/containerd/io.containerd.snapshotter.v1.blockfile"
  scratch_file = "/opt/containerd/blockfile/scratch"
  supported_platforms = ["linux/amd64"]
```

  -  In containerd 1.x:

```toml
[plugins."io.containerd.snapshotter.v1.blockfile"]
  fs_type = "ext4"
  mount_options = []
  recreate_scratch = true
  root_path = "/var/lib/containerd/io.containerd.snapshotter.v1.blockfile"
  scratch_file = "/opt/containerd/blockfile/scratch"
  supported_platforms = ["linux/amd64"]
```

> Blockfile configuration options:
> - `root_path`: Directory for storing block files (must be writable by containerd).
> - `fs_type`: Filesystem type for block files (supported: ext4)
> - `scratch_file`: The path to the empty file that will be used as the base for the block files. 
> - `recreate_scratch`: If set to true, the snapshotter will recreate the scratch file if it is missing.

-  Then, we need to restart the containerd service

```bash
sudo systemctl restart containerd
```

-  Verify the blockfile snapshotter is available

```bash
$ sudo ctr plugin ls | grep blockfile
   io.containerd.snapshotter.v1           blockfile               linux/amd64    ok
```

### Install nerdctl

To easily run containers we can install `docker` or `nerdctl`. For `nerdctl`:

```bash
NERDCTL_VERSION=$(curl -L -s -o /dev/null -w '%{url_effective}' "https://github.com/containerd/nerdctl/releases/latest" | grep -oP "v\d+\.\d+\.\d+" | sed 's/v//')
wget -q https://github.com/containerd/nerdctl/releases/download/v$NERDCTL_VERSION/nerdctl-$NERDCTL_VERSION-linux-$(dpkg --print-architecture).tar.gz
sudo tar Cxzvf /usr/local/bin nerdctl-$NERDCTL_VERSION-linux-$(dpkg --print-architecture).tar.gz
rm -f nerdctl-$NERDCTL_VERSION-linux-$(dpkg --print-architecture).tar.gz
```

#### Install CNI plugins

In order to provide networking for the containers, we will need some CNI plugins.

```bash
CNI_VERSION=$(curl -L -s -o /dev/null -w '%{url_effective}' "https://github.com/containernetworking/plugins/releases/latest" | grep -oP "v\d+\.\d+\.\d+" | sed 's/v//')
wget -q https://github.com/containernetworking/plugins/releases/download/v$CNI_VERSION/cni-plugins-linux-$(dpkg --print-architecture)-v$CNI_VERSION.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-$(dpkg --print-architecture)-v$CNI_VERSION.tgz
rm -f cni-plugins-linux-$(dpkg --print-architecture)-v$CNI_VERSION.tgz
```

## Step 2: Install all supported monitors

For installing the supported monitors, we can either make use of the
[monitors-build repository](https://github.com/urunc-dev/monitors-build) or
install every monitor one by one.

### Option 1: Using the monitors-build repository

For convenience, we have created [monitor-builds
repository](https://github.com/urunc-dev/monitors-build) to provide a reference
setup for building and distributing static binaries of monitors and tools for
`urunc`. In the i[releases
page](https://github.com/urunc-dev/monitors-build/releases) there are archives
with all necessary artifacts. Therefore, we can choose a release based on the
monitors' versions and download it.

For example to download the monitors and tools witht he following versions:

- Firecracker v1.7.0
- Solo5 v0.9.3
- Virtiofsd v1.13.1
- Qemu v10.1.1 with a hash generated by the config files and the excluded files
  file under Qemu's directory.

```
wget https://github.com/urunc-dev/monitors-build/releases/download/FC-v1.7.0_S5-v0.9.3_VFS_-v1.13.1_QM-v10.1.1-d4dd3/release-FC-v1.7.0_S5-v0.9.3_VFS_-v1.13.1_QM-v10.1.1-d4dd3.tar.gz
tar xvf release-FC-v1.7.0_S5-v0.9.3_VFS_-v1.13.1_QM-v10.1.1-d4dd3.tar.gz
```

Then, we simply need to let `urunc` know where each monitor is placed.

### Option 2: Fetching or building from source

ALternatively, we can download or build the monitors.

#### Solo5

In the case of Solo5, we can only build it from scratch cloning the
[repository](https://github.com/Solo5/solo5). It is important to note that
`solo5-spt` requires `libseccomp-dev`.

```bash
git clone -b v[[ versions.solo5 ]] https://github.com/Solo5/solo5.git
cd solo5
./configure.sh  && make -j$(nproc)
sudo cp tenders/hvt/solo5-hvt /usr/local/bin
sudo cp tenders/spt/solo5-spt /usr/local/bin
```

### Qemu

For [qemu](https://www.qemu.org/) we can simply make use of the package manager:

```bash
sudo apt install qemu-system
```

### Firecracker

In the case of
[firecracker](https://github.com/firecracker-microvm/firecracker), we can grab a
binary from its releases: We choose to install version 1.7.0, since Unikraft has
some [issues](https://github.com/unikraft/unikraft/issues/1410) with newer
versions.

```bash
ARCH="$(uname -m)"
VERSION="v[[ versions.firecracker ]]"
release_url="https://github.com/firecracker-microvm/firecracker/releases"
curl -L ${release_url}/download/${VERSION}/firecracker-${VERSION}-${ARCH}.tgz | tar -xz
# Rename the binary to "firecracker"
sudo mv release-${VERSION}-${ARCH}/firecracker-${VERSION}-${ARCH} /usr/local/bin/firecracker
rm -fr release-${VERSION}-${ARCH}
```

### Virtiofsd

As an alternative to 9pfs, `urunc` is able to spawn guests using virtiofs. For that
purpose, we need to install virtiofsd under `/usr/libexec` where `urunc` expects to find
the statically linked binary of `virtiofsd`. For easier installation, we have uploaded
the respective binary in `https://s3.nbfc.io/nbfc-assets/github/urunc/bin/virtiofsd`.
Therefore, the installation can easily take place with the following instructions:

```
wget https://s3.nbfc.io/nbfc-assets/github/urunc/bin/virtiofsd
chmod +x virtiofsd
sudo cp virtiofsd /usr/libexec/
```

## Step 3: Install urunc and configure containerd

### Installing urunc

To install `urunc`, there are three options: a) building from source, b)
grabbing the binaries from the latest release, or c) grabbing the binaries from
the lastest commit in main.

#### Option 1: Building from source

In order to build `urunc` from source, we need to install Go.
Any version earlier than Go 1.20.6 will be sufficient.

```bash
GO_VERSION=[[ versions.go ]]
wget -q https://go.dev/dl/go${GO_VERSION}.linux-$(dpkg --print-architecture).tar.gz
sudo mkdir /usr/local/go${GO_VERSION}
sudo tar -C /usr/local/go${GO_VERSION} -xzf go${GO_VERSION}.linux-$(dpkg --print-architecture).tar.gz
sudo tee -a /etc/profile > /dev/null << EOT
export PATH=\$PATH:/usr/local/go$GO_VERSION/go/bin
EOT
rm -f go${GO_VERSION}.linux-$(dpkg --print-architecture).tar.gz
```

> Note: You might need to logout and log back in to the shell, in order to use
> Go.


After installing Go, we can clone and build `urunc`:

```bash
git clone https://github.com/urunc-dev/urunc.git
cd urunc
make && sudo make install
cd ..
```

#### Option 2: Install latest release

We can also install `urunc` from its latest
[release](https://github.com/urunc-dev/urunc/releases):

```bash
URUNC_VERSION=$(curl -L -s -o /dev/null -w '%{url_effective}' "https://github.com/urunc-dev/urunc/releases/latest" | grep -oP "v\d+\.\d+\.\d+" | sed 's/v//')
URUNC_BINARY_FILENAME="urunc_static_v${URUNC_VERSION}_$(dpkg --print-architecture)"
wget -q https://github.com/urunc-dev/urunc/releases/download/v$URUNC_VERSION/$URUNC_BINARY_FILENAME
chmod +x $URUNC_BINARY_FILENAME
sudo mv $URUNC_BINARY_FILENAME /usr/local/bin/urunc
```

And for `containerd-shim-urunc-v2`:

```bash
CONTAINERD_BINARY_FILENAME="containerd-shim-urunc-v2_static_v${URUNC_VERSION}_$(dpkg --print-architecture)"
wget -q https://github.com/urunc-dev/urunc/releases/download/v$URUNC_VERSION/$CONTAINERD_BINARY_FILENAME
chmod +x $CONTAINERD_BINARY_FILENAME
sudo mv $CONTAINERD_BINARY_FILENAME /usr/local/bin/containerd-shim-urunc-v2
```

#### Option 3: Install from latest artifacts (tip of the main branch)

We can also install `urunc` from binary builds of the main branch:

```bash
URUNC_VERSION=main
URUNC_BINARY_FILENAME="urunc_static_$(dpkg --print-architecture)"
wget -q https://s3.nbfc.io/nbfc-assets/github/urunc/dist/$URUNC_VERSION/$(dpkg --print-architecture)/$URUNC_BINARY_FILENAME
chmod +x $URUNC_BINARY_FILENAME
sudo mv $URUNC_BINARY_FILENAME /usr/local/bin/urunc
```

And for `containerd-shim-urunc-v2`:

```bash
CONTAINERD_BINARY_FILENAME="containerd-shim-urunc-v2_static_$(dpkg --print-architecture)"
wget -q https://s3.nbfc.io/nbfc-assets/github/urunc/dist/$URUNC_VERSION/$(dpkg --print-architecture)/$CONTAINERD_BINARY_FILENAME
chmod +x $CONTAINERD_BINARY_FILENAME
sudo mv $CONTAINERD_BINARY_FILENAME /usr/local/bin/containerd-shim-urunc-v2
```

### Add urunc runtime to containerd

We also need to add `urunc` as a runtime in containerd's configuration:

- In containerd 2.x:

```toml
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.urunc]
    runtime_type = "io.containerd.urunc.v2"
    container_annotations = ["com.urunc.unikernel.*"]
    pod_annotations = ["com.urunc.unikernel.*"]
    snapshotter = "devmapper"
```

- In containerd 1.x:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.urunc]
    runtime_type = "io.containerd.urunc.v2"
    container_annotations = ["com.urunc.unikernel.*"]
    pod_annotations = ["com.urunc.unikernel.*"]
    snapshotter = "devmapper"
```

At last, we need to restart containerd.

```bash
sudo systemctl restart containerd
```
## Run example unikernels

Now, let's run some unikernels for every VM/Sandbox monitor, to make sure
everything was installed correctly.

#### Run a Redis Rumprun unikernel over Solo5-hvt

```bash
sudo nerdctl run --rm -ti --runtime io.containerd.urunc.v2 harbor.nbfc.io/nubificus/urunc/redis-hvt-rumprun-block:latest
```
#### Run a Redis rumprun unikernel over Solo5-spt with devmapper

```bash
sudo nerdctl run --rm -ti --snapshotter devmapper --runtime io.containerd.urunc.v2 harbor.nbfc.io/nubificus/urunc/redis-spt-rumprun-raw:latest
```
#### Run a Nginx Unikraft unikernel over Qemu

```bash
sudo nerdctl run --rm -ti --runtime io.containerd.urunc.v2 harbor.nbfc.io/nubificus/urunc/nginx-qemu-unikraft-initrd:latest
```
#### Run a Nginx Unikraft unikernel over Firecracker

```bash
sudo nerdctl run --rm -ti --runtime io.containerd.urunc.v2 harbor.nbfc.io/nubificus/urunc/nginx-firecracker-unikraft-initrd:latest
```
