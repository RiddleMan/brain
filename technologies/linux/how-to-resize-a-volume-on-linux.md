# How to resize a volume on linux
In this guide, we're going to increase the size of a hard drive in a Proxmox hypervisor to later use this space in linux. To achieve that, we're going to create a new partition which is going to be used to increase the size of volume group and logical volume. This all could be done with minimal downtime and shouldn't take more than 5 minutes to perform.

## Step 1. Resize a volume on VM (Proxmox)
1. Shutdown a machine and go to it's details.
2. Click on _Hard disk_ you want to resize
3. Click _Resize disk_
4. Define how much disk space you want to increment for the volume
5. Run the machine
![[proxmox-disk-resize.png]]
## Step 2. Extend logical volume on linux
Based on: https://www.redhat.com/sysadmin/resize-lvm-simple

### Inspect volumes

#### pvs
Inspects all physical volumes

Command:
```bash
pvs
```

Output:
```

  PV         VG        Fmt  Attr PSize   PFree
  /dev/sda3  ubuntu-vg lvm2 a--  <49.00g <24.50g
```

#### vgs
Inspects all volume groups

Command:
```bash
vgs
```

Output:
```
  VG        #PV #LV #SN Attr   VSize   VFree
  ubuntu-vg   1   1   0 wz--n- <49.00g <24.50g
```

#### lvs
Inspects all logical volumes

Command:
```bash
lvs
```

Output:
```
  LV        VG        Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv ubuntu-vg -wi-ao---- 24.50g
```

### Create a new partition for free space\
Determine what disk you want to resize

Command:
```bash
lsblk
```

Output:
```
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0  44.5M  1 loop /snap/certbot/2511
loop1                       7:1    0  44.6M  1 loop /snap/certbot/2539
loop2                       7:2    0   115M  1 loop /snap/core/13886
loop3                       7:3    0 114.9M  1 loop /snap/core/14056
loop4                       7:4    0  55.6M  1 loop /snap/core18/2620
loop5                       7:5    0  67.8M  1 loop /snap/lxd/22753
loop6                       7:6    0  55.6M  1 loop /snap/core18/2632
loop7                       7:7    0  63.2M  1 loop /snap/core20/1634
loop8                       7:8    0  91.8M  1 loop /snap/lxd/23991
loop9                       7:9    0  63.2M  1 loop /snap/core20/1695
sda                         8:0    0   100G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     1G  0 part /boot
└─sda3                      8:3    0    49G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0  24.5G  0 lvm  /
sr0                        11:0    1  1024M  0 rom
```

In our case this is a `/dev/sda`. Proceed with `fdisk` to create a new partition in the empty space.
```
$ fdisk /dev/sda

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

GPT PMBR size mismatch (104857599 != 209715199) will be corrected by write.
The backup GPT table is not on the end of the device. This problem will be corrected by write.

Command (m for help): n
Partition number (4-128, default 4):
First sector (104855552-209715166, default 104855552):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (104855552-209715166, default 209715166):

Created a new partition 4 of type 'Linux filesystem' and of size 50 GiB.

Command (m for help): t
Partition number (1-4, default 4): 4
Partition type (type L to list all types): 8e

Type of partition 4 is unchanged: Linux filesystem.

Command (m for help): w
The partition table has been altered.
Syncing disks.

$ lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0  44.5M  1 loop /snap/certbot/2511
loop1                       7:1    0  44.6M  1 loop /snap/certbot/2539
loop2                       7:2    0   115M  1 loop /snap/core/13886
loop3                       7:3    0 114.9M  1 loop /snap/core/14056
loop4                       7:4    0  55.6M  1 loop /snap/core18/2620
loop5                       7:5    0  67.8M  1 loop /snap/lxd/22753
loop6                       7:6    0  55.6M  1 loop /snap/core18/2632
loop7                       7:7    0  63.2M  1 loop /snap/core20/1634
loop8                       7:8    0  91.8M  1 loop /snap/lxd/23991
loop9                       7:9    0  63.2M  1 loop /snap/core20/1695
sda                         8:0    0   100G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     1G  0 part /boot
├─sda3                      8:3    0    49G  0 part
│ └─ubuntu--vg-ubuntu--lv 253:0    0  24.5G  0 lvm  /
└─sda4                      8:4    0    50G  0 part
sr0                        11:0    1  1024M  0 rom
```

Create a physical volume on a newly created partition:
```
$ pvcreate /dev/sda4
  Physical volume "/dev/sda4" successfully created.
```

Inspect available volume groups:
```
$ vgdisplay
  --- Volume group ---
  VG Name               ubuntu-vg
  ...
  VG Size               <49.00 GiB
  PE Size               4.00 MiB
  Total PE              12543
  Alloc PE / Size       6272 / 24.50 GiB
  Free  PE / Size       6271 / <24.50 GiB
```

Extend volume group _ubuntu-vg_ by a new space.
```
$ vgextend ubuntu-vg /dev/sda4
  Volume group "ubuntu-vg" successfully extended
```

Verify whether volume group has been resized.
```
$ vgdisplay
  --- Volume group ---
  VG Name               ubuntu-vg
  ...
  VG Size               98.99 GiB
```

Next step is to extend a logical volume to fill empty space.
Inspect logical volume:
```
$ lvdisplay
  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  ...
  LV Size                24.50 GiB
```

Resize logical volume by the size of free space from the last output from `vgdisplay` command.
```
$ lvextend -L +74.49G /dev/ubuntu-vg/ubuntu-lv
  Rounding size to boundary between physical extents: 74.49 GiB.
  Size of logical volume ubuntu-vg/ubuntu-lv changed from 24.50 GiB (6272 extents) to 98.99 GiB (25342 extents).
  Logical volume ubuntu-vg/ubuntu-lv successfully resized.
```

Verify that all the space has been used.
```
$ vgdisplay
  --- Volume group ---
  VG Name               ubuntu-vg
  ...
  VG Size               98.99 GiB
  PE Size               4.00 MiB
  Total PE              25342
  Alloc PE / Size       25342 / 98.99 GiB
  Free  PE / Size       0 / 0

$ lvdisplay
  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  ...
  LV Size                98.99 GiB
```

The last step is to resize a filesystem to use the available space.
```
$ resize2fs /dev/ubuntu-vg/ubuntu-lv
resize2fs 1.45.5 (07-Jan-2020)
Filesystem at /dev/ubuntu-vg/ubuntu-lv is mounted on /; on-line resizing required
old_desc_blocks = 4, new_desc_blocks = 13
The filesystem on /dev/ubuntu-vg/ubuntu-lv is now 25950208 (4k) blocks long.
```

And the last verify:
```
$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
udev                               952M     0  952M   0% /dev
tmpfs                              199M  1.7M  198M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   98G   20G   74G  21% /
tmpfs                              994M     0  994M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              994M     0  994M   0% /sys/fs/cgroup
/dev/sda2                          976M  181M  728M  20% /boot
```