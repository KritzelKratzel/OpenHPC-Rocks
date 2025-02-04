# CST-2025 Cluster Installation

[CST Studio Suite®](https://www.3ds.com/products/simulia/cst-studio-suite) is a high-performance 3D EM analysis software package for designing, analyzing and optimizing electromagnetic (EM) components and systems and can be deployed on compute-clusters. In this recipe a seamless integration of CST into OpenHPC-Rocks will be presented. As a CST-installation typically consumes about 20 GB in installation space (on frontend-node and on each compute-node), it is not wise to install CST directly into a compute-node container for deployment, because the resulting container size would literally explode. Instead, the following procedure is suggested which uses the opportunities offered by OpenHPC-Rocks and Warewulf stateless provisioning:

- Install CST (MainController, SolverServer, ...) on frontend-node only,
- NFS-export this directory to all compute-nodes,
- Run SolverServer on each compute-node from the exported CST-installation.

This idea requires just a few steps to be realized - here exemplarily demonstrated for a compute-node container called `rocky`.  If there is more than just one container, all following steps need to be performed individually per container.

## CST-2025 Install on Frontend-Node

Install typically required RPM packages on frontend-node:

```bash
$ dnf install -y libnsl.x86_64 mesa-libGLU.x86_64 motif.x86_64 xcb-util-image.x86_64 xcb-util-keysyms.x86_64 xcb-util-renderutil.x86_64 xcb-util-wm.x86_64 xorg-x11-server-Xvfb
```

Download CST-2025 Linux installation media, unpack it, run installer `installer.sh` and install at least MainController and SolverServer to default directory `/opt/cst/CST_Studio_Suite_2025`. Permanently-enable MainController-daemon and permanently-disable SolverServer-daemon on frontend-node:

```bash
$ systemctl enable --now cst-maincontroller2025.service
$ systemctl disable --now cst-solverserver2025.service
```

Edit `/etc/warewulf/warewulf.conf` and enable mounting of `/opt` on compute-nodes by setting `mount: true`.

```bash
...
nfs:
...
  - path: /opt
    export options: ro,sync,no_root_squash
    mount options: defaults
    mount: true
  systemd name: nfs-server
...
```

This will make the CST-installation available on all compute-nodes. Finally run

```bash
$ wwctl configure -a
```

to apply all changes.

> [!TIP]
>
> If there are CST service packs available, install them now onto the frontend-node installation as described by CST installation instructions:
>
> ```bash
> $ /opt/cst/CST_Studio_Suite_2025/update_with_supfile <patchfile.sup>
> ```

## CST-2025 Installs in Container for Compute-Nodes

Copy following files from CST-2025 install on frontend-node into container `rocky` - obeying the weird original naming scheme containing spaces:

```bash
# Create subdir /etc/xdg/CST AG/CST
$ mkdir -p /srv/warewulf/chroots/rocky/rootfs/etc/xdg/CST\ AG/CST
# Copy /etc/xdg/CST AG/CST DESIGN ENVIRONMENT.conf
$ cp /etc/xdg/CST\ AG/CST\ DESIGN\ ENVIRONMENT.conf /srv/warewulf/chroots/rocky/rootfs/etc/xdg/CST\ AG/CST
# Copy /etc/xdg/CST AG/CST DC Solver Control 2025.conf
$ cp /etc/xdg/CST\ AG/CST\ DC\ Solver\ Control\ 2025.conf /srv/warewulf/chroots/rocky/rootfs/etc/xdg/CST\ AG/CST
# Copy systemd script for SolverServer
$ cp /etc/systemd/system/cst-solverserver2025.service /srv/warewulf/chroots/rocky/rootfs/etc/systemd/system
```

All those original files carry the executable file mode bits, which will make `systemd` complaining about it. However, it is OK to leave those file as they are. Next jump into compute-node container `rocky` and perform the following operations:

```bash
$ wwctl container shell rocky

# Install required RPM-packages by CST-2025
[rocky] Warewulf> dnf install -y alsa-lib.x86_64 bind-utils.x86_64 cups-libs.x86_64 libXcomposite.x86_64 libXdamage.x86_64 libXtst.x86_64 libnsl.x86_64 libxkbcommon-x11.x86_64 mesa-libGLU.x86_64 motif.x86_64 nss.x86_64 shared-mime-info.x86_64 xcb-util-image.x86_64 xcb-util-keysyms.x86_64 xcb-util-renderutil.x86_64 xcb-util-wm.x86_64 xdg-utils.noarch xorg-x11-server-Xvfb

# Enable SolverServer
[rocky] Warewulf> systemctl enable cst-solverserver2025.service
# Add dependency on nfs-client.target
[rocky] Warewulf> systemctl add-requires cst-solverserver2025.service nfs-client.target

# Exit and rebuild the container
[rocky] Warewulf> exit
```

The additional dependency is introduced to make sure, the NFS-client on a compute-node is running and the exported `/opt` file system is mounted, before the CST SolverServer daemon is started. Reboot compute-node to apply changes.

## Customizing CST-Installation

Every customization in CST-2025 config files requires the compute-node container to be rebuild and the compute-node to be rebooted.

```bash
$ wwctl container shell rocky
[rocky] Warewulf> vim /etc/xdg/CST\ AG/CST/CST\ DC\ Solver\ Control\ 2025.conf
[rocky] Warewulf> vim /etc/xdg/CST\ AG/CST/CST\ DESIGN\ ENVIRONMENT.conf
[rocky] Warewulf> exit
```

## Update cluster-wide CST-Installation

CST service packs come as `*.sup` files, e.g. `CST_S2_v2025_LIN_SP2_2024-08-30_2024-12-16.sup`. The NFS-exported installation of CST-2025 can be updated easily by applying such service packs to the frontend-node CST-installation only. As CST is not aware of the present non-standard cluster installation, it is necessary to manually stop all running local SolverServers on all compute-nodes - here shown for a cluster comprising compute-nodes `compute-0-0` to `compute-0-3`.

```bash
$ wwctl ssh compute-0-[0-3] systemctl stop cst-solverserver2025
```

After all SolverServers are stopped apply the CST service pack and restart SolverServers again:

```bash
# Start update operartion
$ /opt/cst/CST_Studio_Suite_2025/update_with_supfile <patchfile.sup>
# Restart individual SolverServers on compute-nodes
$ wwctl ssh compute-0-[0-3] systemctl start cst-solverserver2025
```

`update_with_supfile` automatically stops and restarts MainController on frontend-node.

## Adding GPU-Support to Compute-Nodes

Following the [CST Studio Suite 2025 GPU Computing Guide](https://updates.cst.com/downloads/GPU_Computing_Guide_2025.pdf) the NVIDIA Driver Version **550.90.07** is required for operation. The actual NVIDIA driver version changes with each CST-release, but will be fixed over the life cycle of each release, regardless of NVIDIA's driver release policy and cycle. Therefore, simply adding the NVIDIA repository to `dnf` via ...

```bash
$ dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
```

... will not work, as it will not present the desired driver version **550.90.07** to the operator. An elegant way out of this dilemma is to create a [local repository](https://github.com/taw00/howto/blob/master/howto-setup-a-local-yum-dnf-repository.md) on the frontend-node, only for the desired GPU driver version.

### Installs on Frontend-Node

Install required tools on frontend-node:

```bash
$ dnf install -y wget createrepo_c
```

Create a new directory for containing all local repository data:

```bash
$ mkdir /var/local/NVIDIA-550.90.07
```

Download all relevant RPM packages (everything with 550.90.07 in name; only x86_64) from NVIDIA's server with `wget`:

```bash
$ wget -r -l1 -np -P /var/local/NVIDIA-550.90.07 -nv -nd "https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64" -A "*550\.90\.07*,dnf-plugin-nvidia*" -R "*i686*"
```

Initialize the local repository with:

```bash
$ cd /var/local/NVIDIA-550.90.07
$ createrepo_c .
```

Create file `/etc/yum.repos.d/NVIDIA-v550.90.07.repo` with the following content:

```bash
[NVIDIA-550.90.07]
name=NVIDIA Driver v550.90.07
baseurl=/var/local/NVIDIA-550.90.07
enabled=1
gpgcheck=1
gpgkey=https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/D42D0685.pub
```

Check the availability of the local repository:

```bash
$ dnf repolist
repo id                        repo name
...
NVIDIA-550.90.07               NVIDIA Driver v550.90.07
...
```

Edit `/etc/warewulf/warewulf.conf` to mount local repository file and directory to all containers:

```bash
...
container mounts:
- source: /etc/resolv.conf
  dest: /etc/resolv.conf
  readonly: true
- source: /etc/yum.repos.d/NVIDIA-v550.90.07.repo
  dest: /etc/yum.repos.d/NVIDIA-v550.90.07.repo
  readonly: true
- source: /var/local/NVIDIA-550.90.07
  dest: /var/local/NVIDIA-550.90.07
  readonly: true
...
```

Reload Warewulf settings:

```bash
$ systemctl reload warewulfd.service
```

### Installs in Container for Compute-Nodes

Create repository mount point in container (throws a warning message at the very first time - ignore it):

```bash
$ wwctl container exec rocky /usr/bin/mkdir /var/local/NVIDIA-550.90.07
```

Open compute-node container and install NVIDIA driver:

```bash
$ wwctl container shell rocky
[rocky] Warewulf> dnf install nvidia-driver
[rocky] Warewulf> systemctl enable dkms
[rocky] Warewulf> exit
```

### Automatic Creation of GPU-Devices 

It is necessary to verify the presence of Linux device nodes upon start-up of a compute node. Open compute-node container `rocky`, create two files, enable a new service and rebuild the container by calling `exit`. The desired content of both files is shown below.

```bash
$ wwctl container shell rocky
[rocky] Warewulf> vim /usr/local/bin/nvidia-devs.sh
[rocky] Warewulf> vim /etc/systemd/system/nvidia-devs.service
[rocky] Warewulf> systemctl enable nvidia-devs
[rocky] Warewulf> exit
```

Content and location of `/usr/local/bin/nvidia-devs.sh`:

```bash
#!/bin/bash

/sbin/modprobe nvidia

if [ "$?" -eq 0 ]; then
  # Count the number of NVIDIA controllers found.
  NVDEVS=`lspci | grep -i NVIDIA`
  N3D=`echo "$NVDEVS" | grep "3D controller" | wc -l`
  NVGA=`echo "$NVDEVS" | grep "VGA compatible controller" | wc -l`

  N=`expr $N3D + $NVGA - 1`
  for i in `seq 0 $N`; do
    mknod -m 666 /dev/nvidia$i c 195 $i
  done

  mknod -m 666 /dev/nvidiactl c 195 255

else
  exit 1
fi

/sbin/modprobe nvidia-uvm

if [ "$?" -eq 0 ]; then
  # Find out the major device number used by the nvidia-uvm driver
  D=`grep nvidia-uvm /proc/devices | awk '{print $1}'`

  mknod -m 666 /dev/nvidia-uvm c $D 0
else
  exit 1
fi

exit 0
```

Content and location of `/etc/systemd/system/nvidia-devs.service`:

```bash
[Unit]
Description=Create NVIDIA devices on startup
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/local/bin/nvidia-devs.sh
TimeoutStartSec=0
RemainAfterExit=no

[Install]
WantedBy=multi-user.target
```

This approach also works for compute-nodes without GPU cards at all. In that case the systemd script `nvidia-devs.service`  simply throws an error and exits. The following command queries the corresponding systemd journal on `compute-0-0`:

```bash
$ ssh compute-0-0 journalctl -u nvidia-devs
Feb 04 21:40:05 compute-0-0 bash[523]: modprobe: FATAL: Module nvidia not found in directory /lib/modules/5.14.0-503.19.1.el9_5.x86_64
Feb 04 21:40:05 compute-0-0 systemd[1]: Failed to start Create NVIDIA devices on startup.
$
```

## References

- https://updates.cst.com/downloads/CST-OS-Support.pdf
- https://updates.cst.com/downloads/GPU_Computing_Guide_2025.pdf
- https://forums.rockylinux.org/t/tutorial-for-nvidia-gpu/4234/12
- https://docs.redhat.com/en/documentation/red_hat_virtualization/4.3/html-single/setting_up_an_nvidia_gpu_for_a_virtual_machine_in_red_hat_virtualization/index#Detaching_the_GPU_device_from_the_host_nvidia_gpu_passthrough
- https://github.com/taw00/howto/blob/master/howto-setup-a-local-yum-dnf-repository.md
- https://www.admin-magazine.com/HPC/Articles/Warewulf-4-GPUs/
- https://imaxv.com/2022/07/31/2022/Use-Nvidia-GPU-with-Podmam/
