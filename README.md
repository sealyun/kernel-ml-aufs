RHEL-AUFS-Kernel: `kernel-ml` with AUFS Support
=============================================================================

This repository contains the specfile and config files to build [kernel-ml](http://elrepo.org/tiki/kernel-ml) kernels that include AUFS for use with Docker. The Docker spec files that were part of the [original repo](https://github.com/sciurus/docker-rhel-rpm.git) are no longer included.

***
### Downloading Prebuilt Packages

The simplest method for using these packages is to download them directly from https://yum.spaceduck.org/. Install the [.repo](https://yum.spaceduck.org/rhel-aufs-kernel/rhel-aufs-kernel.repo) file into `/etc/yum.repos.d` to get updates automatically.

Please keep in mind that new packages are built as my spare time allows, and that updates to this repo will often appear before the packages are built. If you can't wait, you can build these yourself using one of the methods below.

***
### Prerequisites

Before building the packages, be sure to install [fedora-packager](https://dl.fedoraproject.org/pub/epel/6/x86_64/repoview/fedora-packager.html) and add yourself to the *mock* group.

Be aware that building the kernel can take a long time (at least half an hour, up to several hours if you're building on an older machine).

To build the packages, there are two options.

***
### Using the Build Script

Run the `build_kernel.sh` script, and answer all three questions:

    $ ./build_kernel.sh
    What kernel version do you want to build? (major version only)
    3.19
    What architecture do you want to build for? (i686, i686-NONPAE, x86_64)
    x86_64
    What version of CentOS/RHEL do you want to build for? (6 or 7)
    7

This will build your packages automatically. Logs for the build can be found in two places:
* `$(REPO_LOCATION)/build/logs` has the output from each command in separate log files.
* `$(REPO_LOCATION)/build/output` contains the `mock` logs so you can see where the build went wrong (if it went wrong at all).

If all goes well, your new RPMs will be moved to `~/RPMs`. The `build` directory will remain behind for analysis.

***
### Building Packages Manually

If you'd rather run through the steps manually, you can do so with the instructions below. Be sure to change `epel-6-x86_64` to `epel-7-x86_64` and the filename for the source RPM if you're building for EL7.

Linux 4.x will use the newer AUFS 4 tree in its packages. Linux 3.x will use AUFS 3.x.

In the example below, we're building `kernel-ml` 3.19.0 with the latest commit out of the AUFS tree (`f60288dc0e0aab77ca545f42d785ec280f4700b9`) at the time of writing. When you build your kernel versions, be sure to update this step to the latest commit.

    spectool -g -C kernel-ml-aufs kernel-ml-aufs/kernel-ml-aufs-3.19.spec
    git clone git://git.code.sf.net/p/aufs/aufs3-standalone -b aufs3.19
    pushd aufs3-standalone
    git archive f60288dc0e0aab77ca545f42d785ec280f4700b9 > ../kernel-ml-aufs/aufs3-standalone.tar
    popd
    mock -r epel-6-x86_64 --buildsrpm --spec kernel-ml-aufs/kernel-ml-aufs-3.19.spec --sources kernel-ml-aufs --resultdir output
    mock -r epel-6-x86_64 --rebuild --resultdir output output/kernel-ml-aufs-3.19.0-1.el6.src.rpm

The resulting RPMs will be placed in a directory named `output`.

***
### Installing the Packages

Once your packages are built, you can install them with `yum`. `cd` to the appropriate directory, and run:

    yum localinstall --nogpgcheck kernel-ml-aufs-3.19.0-1.el6.x86_64.rpm

In order to use docker on EL6, you'll need to install it out of EPEL:

    yum install docker-io

For EL7, Docker packages are in the main repos:

    yum install docker

Reboot and choose the AUFS kernel from your GRUB menu (or edit GRUB to change your default kernel).

If everything is working as expected, you should see that AUFS is your storage driver when you run `docker info`:

    [bnied@buildbox ~]$ sudo docker info
    Containers: 0
    Images: 0
    Storage Driver: aufs
    Root Dir: /var/lib/docker/aufs
    Backing Filesystem: xfs
    Dirs: 0
    Dirperm1 Supported: true
    Execution Driver: native-0.2
    Kernel Version: 4.0.2-1.el7.centos.x86_64
    Operating System: CentOS Linux 7 (Core)
    CPUs: 2
    Total Memory: 1.938 GiB
    Name: buildbox.local
    ID: BZQL:SPMI:Y23R:KBPK:SHR2:UTIN:Q2SS:N6SE:3DL2:PNKK:YP5D:OANX
    WARNING: No swap limit support

If you're still seeing `devicemapper` as your storage backend, you'll need to change Docker's configuration to use AUFS. Open `/etc/sysconfig/docker`, and change line 4 from:

    OPTIONS='--selinux-enabled'
to:

    OPTIONS='--selinux-enabled --storage-driver=aufs'

Restart the Docker service, and it should switch to the correct backend.
