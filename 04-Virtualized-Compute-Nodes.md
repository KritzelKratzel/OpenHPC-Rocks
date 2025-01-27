# Virtualized Compute-Nodes

This recipe outlines how to setup a virtual compute-node with VMware Fusion. This description can be skipped for real-world compute-nodes.

Create a new custom virtual machine for **Rocky Linux 64-bit** with a new virtual disk and call it `compute-0-0` following the Rocksclusters definition (http://central-7-0-x86-64.rocksclusters.org/roll-documentation/base/7.0/install-compute-nodes.html). The first digit indicates the cabinet number and the second digit the node number. Open the settings of the virtualized compute-node and apply the following changes from the otherwise given default settings.

<img src="./assets/VMware-Fusion-compute-node-settings.png" alt="VMware-Fusion-compute-node-settings" style="zoom:50%;" />

Deactivate Default Applications.

<img src="./assets/VMware-Fusion-compute-node-default-apps.png" alt="VMware-Fusion-compute-node-default-apps" style="zoom:50%;" />

Select two processor cores with 6144 MB RAM.

<img src="./assets/VMware-Fusion-compute-node-cpu-ram.png" alt="VMware-Fusion-compute-node-cpu-ram" style="zoom:50%;" />

Deactivate 3D graphics support.

<img src="./assets/VMware-Fusion-compute-node-3d-graphics.png" alt="VMware-Fusion-compute-node-3d-graphics" style="zoom:50%;" />

Connect Network Adapter to virtualized private network and generate a new MAC address. Save this address for later.

<img src="./assets/VMware-Fusion-compute-node-network.png" alt="VMware-Fusion-compute-node-network" style="zoom:50%;" />

Modify present harddisk to the following settings.

<img src="./assets/VMware-Fusion-compute-node-harddisk.png" alt="VMware-Fusion-compute-node-harddisk" style="zoom:50%;" />

Remove CD/DVD Drive, Sound Card and USB Controller. Disable Isolation.

<img src="./assets/VMware-Fusion-compute-node-isolation.png" alt="VMware-Fusion-compute-node-isolation" style="zoom:50%;" />

Apply advanced settings:

<img src="./assets/VMware-Fusion-compute-node-advanced.png" alt="VMware-Fusion-compute-node-advanced" style="zoom:50%;" />

These steps create a new machine `compute-0-0` in VMware's Virtual Machine Library. Activate the current machine in the list (just one click) and select from the VMware menue `Virtual Machine` -> `Power On to Firmware`. This opens a simplified version of a classic legacy BIOS menue.

<img src="./assets/VMware-Fusion-compute-node-bios-main.png" alt="VMware-Fusion-compute-node-bios-main" style="zoom:50%;" />

> [!TIP]
>
> VMware blocks the computer mouse while working interactively in text-mode consoles. Hit Control+Command to release the mouse.

Jump to the `Boot` section and change order of devices as shown below. Notice that keyboard map in this mode is not your local map, but `en_US`.

<img src="./assets/VMware-Fusion-compute-node-bios-boot.png" alt="VMware-Fusion-compute-node-bios-boot" style="zoom:50%;" />

Jump to `Exit` section and hit `Exit Saving Changes`. The virtual machine reboots, but cannot be detected by the frontend-node yet. Select select from the VMware menue `Virtual Machine` -> `Shutdown` to switch off the compute-node ignoring the warning.

<img src="./assets/VMware-Fusion-compute-node-premature-shutdown.png" alt="VMware-Fusion-compute-node-premature-shutdown" style="zoom:50%;" />

Repeat the sequence for other virtual compute-nodes, e.g. `compute-0-1`, `compute-0-2`, etc. and continue with re recipe [OpenHPC-Rocks Compute-Node Integration](./05-OpenHPC-Rocks-Compute-Node-Integration.md).