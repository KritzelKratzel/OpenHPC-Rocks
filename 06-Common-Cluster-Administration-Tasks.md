# Common Cluster Administration Tasks

Just an unsorted reference for common compute cluster administration tasks. In case of doubt or emergency try calling *the four sisters* in the given order:

```bash
wwctl overlay build # Build or update all compute-node container overlays
wwctl configure -a # Configure everything else
wwctl container build rocky [--force] # Build of update the compute-node container image
scontrol reconfigure # Reconfigure Slurm with the given data from Warewulf database
```

## General Administration Tasks

### User Administration

Add a new user `janedoe` with:

```bash
# Create new user
useradd -c "Jane Doe" -g 100 janedoe
# Set password for new user
passwd janedoe
# Sync users in container image and update image
wwctl container syncuser --write --build rocky
```

Remove an existing user with:

```bash
# Remove user
userdel -r janedoe
# Sync users in container image and update image
wwctl container syncuser --write --build rocky
```

Depending on the `update interval` setting in `/etc/warewulf/warewulf.conf` the propagation of the new user and its credential over the compute cluster may take some time.

### Rocky Linux Software Update

#### On Frontend-Node

Straightforward as with any other computer running under Rocky Linux (and others):

```bash
[root@vmCluster ~]# dnf update -y
Last metadata expiration check: 2:02:11 ago on Mon Jan 27 19:12:03 2025.
Dependencies resolved.
Nothing to do.
Complete!
[root@vmCluster ~]#
```

#### For Compute-Nodes

Updating compute-nodes means updating the corresponding container image first:

```bash
[root@vmCluster ~]# wwctl container shell rocky 
[rocky] Warewulf> dnf update -y    
Dependencies resolved.
Nothing to do.
Complete!
[rocky] Warewulf> exit
exit

The Rocky Linux community provides updates for the latest point release of
Rocky Linux 9. If you need to remain on a specific point release (e.g., Rocky
Linux 9.2) you may want to engage with a commercial support provider for
long-term support.

https://rockylinux.org/support

+ dnf clean all
63 files removed
Rebuilding container...
Created image for VNFS container rocky: /srv/warewulf/provision/container/rocky.img
Compressed image for VNFS container rocky: /srv/warewulf/provision/container/rocky.img.gz
[root@vmCluster ~]#
```

Then reboot all compute-nodes by running:

```bash
wwctl ssh compute-0-[0-3] reboot
```

Notice the range syntax in specifying the desired compute-nodes to be rebooted. After a while all four compute-nodes are back online:

```bash
[root@vmCluster ~]# wwctl ssh compute-0-[0-3] uptime
compute-0-1:  21:26:37 up 4 min,  0 users,  load average: 0.00, 0.01, 0.00
compute-0-3:  21:26:37 up 4 min,  0 users,  load average: 0.01, 0.05, 0.02
compute-0-0:  21:26:37 up 4 min,  0 users,  load average: 0.00, 0.01, 0.00
compute-0-2:  21:26:37 up 4 min,  0 users,  load average: 0.14, 0.05, 0.01
[root@vmCluster ~]#
```

### Check RAID Status on Frontend-Node

The customized RAID1, RAID10 and Volume Groups disk arrangement introduced in [Base Operation System Setup on Frontend Node](./02-Base-Operation-System-Setup-on-Frontend-Node.md) should be checked regularly. Run `lsblk` to see a somewhat graphical representation of devices, partitions, RAID, and Volume Group assignments.

```bash
[root@vmCluster ~]# lsblk 
NAME           MAJ:MIN RM    SIZE RO TYPE   MOUNTPOINTS
sda              8:0    0      1T  0 disk   
├─sda1           8:1    0   1015G  0 part   
│ └─md123        9:123  0 1014.9G  0 raid1  /tmp
├─sda2           8:2    0      4G  0 part   
│ └─md127        9:127  0      4G  0 raid1  /boot
├─sda3           8:3    0      4G  0 part   
│ └─md126        9:126  0      4G  0 raid1  [SWAP]
└─sda4           8:4    0      1G  0 part   
  └─md100        9:100  0      1G  0 raid1  /boot/efi
sdb              8:16   0      1T  0 disk   
├─sdb1           8:17   0   1015G  0 part   
│ └─md123        9:123  0 1014.9G  0 raid1  /tmp
├─sdb2           8:18   0      4G  0 part   
│ └─md127        9:127  0      4G  0 raid1  /boot
├─sdb3           8:19   0      4G  0 part   
│ └─md126        9:126  0      4G  0 raid1  [SWAP]
└─sdb4           8:20   0      1G  0 part   
  └─md100        9:100  0      1G  0 raid1  /boot/efi
sdc              8:32   0      1T  0 disk   
└─sdc1           8:33   0   1024G  0 part   
  └─md125        9:125  0 1023.9G  0 raid1  
    └─vg0-root 253:0    0 1023.9G  0 lvm    /
sdd              8:48   0      1T  0 disk   
└─sdd1           8:49   0   1024G  0 part   
  └─md125        9:125  0 1023.9G  0 raid1  
    └─vg0-root 253:0    0 1023.9G  0 lvm    /
sde              8:64   0      1T  0 disk   
└─sde1           8:65   0   1024G  0 part   
  └─md124        9:124  0      2T  0 raid10 
    └─vg1-home 253:1    0      2T  0 lvm    /home
sdf              8:80   0      1T  0 disk   
└─sdf1           8:81   0   1024G  0 part   
  └─md124        9:124  0      2T  0 raid10 
    └─vg1-home 253:1    0      2T  0 lvm    /home
sdg              8:96   0      1T  0 disk   
└─sdg1           8:97   0   1024G  0 part   
  └─md124        9:124  0      2T  0 raid10 
    └─vg1-home 253:1    0      2T  0 lvm    /home
sdh              8:112  0      1T  0 disk   
└─sdh1           8:113  0   1024G  0 part   
  └─md124        9:124  0      2T  0 raid10 
    └─vg1-home 253:1    0      2T  0 lvm    /home
sr0             11:0    1   1024M  0 rom    
[root@vmCluster ~]#
```

The current RAID status is permanently given in `/proc/mdstat`:

```bash
[root@vmCluster ~]# cat /proc/mdstat 
Personalities : [raid1] [raid10] 
md100 : active raid1 sda4[0] sdb4[1]
      1049536 blocks super 1.0 [2/2] [UU]
      bitmap: 0/1 pages [0KB], 65536KB chunk

md123 : active raid1 sdb1[1] sda1[0]
      1064161280 blocks super 1.2 [2/2] [UU]
      bitmap: 0/8 pages [0KB], 65536KB chunk

md124 : active raid10 sdg1[2] sdh1[3] sdf1[1] sde1[0]
      2147215360 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
      bitmap: 0/16 pages [0KB], 65536KB chunk

md125 : active raid1 sdc1[0] sdd1[1]
      1073607680 blocks super 1.2 [2/2] [UU]
      bitmap: 1/8 pages [4KB], 65536KB chunk

md126 : active raid1 sda3[0] sdb3[1]
      4193280 blocks super 1.2 [2/2] [UU]
      
md127 : active raid1 sda2[0] sdb2[1]
      4193280 blocks super 1.2 [2/2] [UU]
      bitmap: 0/1 pages [0KB], 65536KB chunk

unused devices: <none>
[root@vmCluster ~]#
```

Individual hard disk status (one symbol per disk):

- `U`: up and running
- `_`: disk down and broken.

### Extend Disk Space in `/home` with additional disks

Soon ...

## Slurm Workload Manager related Tasks

### SGE / Slurm Command Comparison

See https://hpcsupport.utsa.edu/foswiki/pub/Main/SampleSlurmSubmitScripts/SGEtoSLURMconversion.pdf

### Slurm Interactive Shell

```bash
srun --pty /bin/bash
```

### Batch Queue Administration

Disable and enable a single compute node:

```bash
# Drain: current job will be finalied, no new jobs wil be accepted
scontrol update nodename=compute-0-0 state=drain reason=noReason

# Enable
scontrol update nodename=compute-0-0 state=resume
```

Disable and enable all compute nodes:

```bash
# Drain: current job will be finalied, no new jobs wil be accepted
scontrol update nodename=all state=drain reason=noReason

# Enable
scontrol update nodename=all state=resume
```

### Slurm Logs

- On frontend-node: `/var/log/slurm/slurmctld.log`
- On compute-node: `/var/log/slurm/slurmd.log`

## Warewulf Provisioning related Tasks

### Help on Warewulfs `text/template` Engine

Warewulf's `text/template` engine is pretty cool, but poorly documented. To the authors knowledge there are only two help resources:

- https://warewulf.org/docs/main/contents/templating.html
- https://pkg.go.dev/text/template

To get a glimps on the availabe objects, attributes etc. it is helpful to directly investigate the data sturctures in Warewulf's source code ...

- Overlay: https://github.com/warewulf/warewulf/blob/main/internal/pkg/overlay/datastructure.go
- Node: https://github.com/warewulf/warewulf/blob/main/internal/pkg/node/datastructure.go

... and to compare this information with present overlay template data in:

- `/srv/warewulf/overlays/wwinit/rootfs`
- `/srv/warewulf/overlays/host/rootfs`
- `/srv/warewulf/overlays/generic/rootfs`
- `/srv/warewulf/overlays/debug/rootfs/warewulf/template-variables.md.ww`

Warewulf template files carry the additional suffix `.ww`, which will be removed upon production of respective text output. In partucular, the file `template-variables.md.ww` is interesting, as it can be used to produce an individual-per-compute-node MarkDown-file with all Warewulf variables, tags and attributes. Run the folloing command from the frontend-node for `compute-0-0`. It will generate a file `compute-0-0.md`.

```bash
wwctl overlay show --render compute-0-0 debug /warewulf/template-variables.md.ww > compute-0-0.md
```

### Add Compute-Node to Cluster

Soon ...

### Remove Compute-Node from Cluster

Soon ...

### Replace Compute-Node in Cluster

Soon ...

### Warwulf Logs

On frontend-node: `/var/log/warewulfd.log`

## References and Further Reading

https://github.com/openhpc/ohpc/releases/download/v3.2.GA/Install_guide-Rocky9-Warewulf4-SLURM-3.2-x86_64.pdf

https://cdrdv2-public.intel.com/671501/installguide-openhpc2-centos8-18jul21.pdf

https://warewulf.org/docs/v4.5.x/

https://www.admin-magazine.com/HPC/Articles/Warewulf-4

https://www.admin-magazine.com/HPC/Articles/Warewulf-4-GPUs/

https://www.admin-magazine.com/HPC/Articles/Warewulf-4-Time-and-Resource-Management
