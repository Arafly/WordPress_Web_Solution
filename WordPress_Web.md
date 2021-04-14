```
araflyayinde@webserver ~]$   sudo lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0  200M  0 part /boot/efi
└─sda2   8:2    0 19.8G  0 part /
sdb      8:16   0   20G  0 disk 
sdc      8:32   0   20G  0 disk 
sdd      8:48   0   20G  0 disk 


[araflyayinde@webserver ~]$ sudo gdisk -l /dev/sdb
GPT fdisk (gdisk) version 0.8.10
Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present
Found valid GPT with protective MBR; using GPT.
Disk /dev/sdb: 41943040 sectors, 20.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 6F5EF0D5-1D6F-4672-BB88-E6472DAB90B6
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 41943006
Partitions will be aligned on 2048-sector boundaries
Total free space is 20971453 sectors (10.0 GiB)
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20973567   10.0 GiB    8E00  Linux LVM


[araflyayinde@webserver ~]$ sudo lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0  200M  0 part /boot/efi
└─sda2   8:2    0 19.8G  0 part /
sdb      8:16   0   20G  0 disk 
└─sdb1   8:17   0   10G  0 part 
sdc      8:32   0   20G  0 disk 
└─sdc1   8:33   0   10G  0 part 
sdd      8:48   0   20G  0 disk 
└─sdd1   8:49   0   10G  0 part 


[araflyayinde@webserver ~]$ which lvm
/usr/sbin/lvm

[araflyayinde@webserver ~]$ sudo pvcreate /dev/sdb1 /dev/sdc1 /dev/sdd1
  Physical volume "/dev/sdb1" successfully created.
  Physical volume "/dev/sdc1" successfully created.
  Physical volume "/dev/sdd1" successfully created.


[araflyayinde@webserver ~]$ sudo pvs
  PV         VG Fmt  Attr PSize  PFree 
  /dev/sdb1     lvm2 ---  10.00g 10.00g
  /dev/sdc1     lvm2 ---  10.00g 10.00g
  /dev/sdd1     lvm2 ---  10.00g 10.00g


[araflyayinde@webserver ~]$ sudo vgcreate vg-webdata /dev/sdb1 /dev/sdc1 /dev/sdd1
  Volume group "vg-webdata" successfully created

[araflyayinde@webserver ~]$ sudo vgs
  VG         #PV #LV #SN Attr   VSize   VFree  
  vg-webdata   3   0   0 wz--n- <29.99g <29.99g


[araflyayinde@webserver ~]$ sudo lvcreate -n communication-lv -L 14G vg-webdata
  Logical volume "communication-lv" created.
[araflyayinde@webserver ~]$ sudo lvcreate -n logs-lv -L 14G vg-webdata
  Logical volume "logs-lv" created.
[araflyayinde@webserver ~]$ sudo lvs
  LV               VG         Attr       LSize  Pool Origin Data%  Meta%  Move Log 
Cpy%Sync Convert
  communication-lv vg-webdata -wi-a----- 14.00g                                    
                
  logs-lv          vg-webdata -wi-a----- 14.00g    

[araflyayinde@webserver ~]$ sudo lsblk
NAME                              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                                 8:0    0   20G  0 disk 
├─sda1                              8:1    0  200M  0 part /boot/efi
└─sda2                              8:2    0 19.8G  0 part /
sdb                                 8:16   0   20G  0 disk 
└─sdb1                              8:17   0   10G  0 part 
  └─vg--webdata-communication--lv 253:0    0   14G  0 lvm  
sdc                                 8:32   0   20G  0 disk 
└─sdc1                              8:33   0   10G  0 part 
  ├─vg--webdata-communication--lv 253:0    0   14G  0 lvm  
  └─vg--webdata-logs--lv          253:1    0   14G  0 lvm  
sdd                                 8:48   0   20G  0 disk 
└─sdd1                              8:49   0   10G  0 part 
  └─vg--webdata-logs--lv          253:1    0   14G  0 lvm  


[araflyayinde@webserver ~]$ df -h
Filesystem                                 Size  Used Avail Use% Mounted on
devtmpfs                                   1.9G     0  1.9G   0% /dev
tmpfs                                      1.9G     0  1.9G   0% /dev/shm
tmpfs                                      1.9G  8.5M  1.9G   1% /run
tmpfs                                      1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda2                                   20G  2.2G   18G  12% /
/dev/sda1                                  200M   12M  189M   6% /boot/efi
tmpfs                                      379M     0  379M   0% /run/user/1000
tmpfs                                      379M     0  379M   0% /run/user/0
/dev/mapper/vg--webdata-communication--lv   14G   41M   13G   1% /var/www/html


[araflyayinde@webserver ~]$ sudo blkid
/dev/sda1: SEC_TYPE="msdos" UUID="D139-1579" TYPE="vfat" PARTLABEL="EFI System Part
ition" PARTUUID="7d417c64-815f-43db-b535-0e6a14f426d3" 
/dev/sda2: LABEL="root" UUID="fbf3ebcb-184f-47eb-bebc-14369017fcfe" TYPE="xfs" PART
UUID="e1bd5c9a-aa7d-4814-b39f-a029e2b60742" 
/dev/sdb1: UUID="hNO19b-piex-ScMu-B9mv-iX75-pWmh-ZWzAPu" TYPE="LVM2_member" PARTLAB
EL="Linux LVM" PARTUUID="726aa0ad-7d0a-4b3d-be75-1789e30799f1" 
/dev/sdc1: UUID="34wVAN-1K4A-yPGr-QzDx-k5mU-9Tqz-qIBzLH" TYPE="LVM2_member" PARTLAB
EL="Linux LVM" PARTUUID="b5a86d1e-4fe9-4c29-9c82-bab0c2d9b229" 
/dev/sdd1: UUID="qHAPNL-BDgx-RR2A-n1s7-GB32-1eRp-YlWqO9" TYPE="LVM2_member" PARTLAB
EL="Linux LVM" PARTUUID="f9785353-e0d1-4474-afbb-517503a3bf88" 
/dev/mapper/vg--webdata-communication--lv: UUID="535ced99-9d4c-4741-898f-ca217c0219
67" TYPE="ext4" 
/dev/mapper/vg--webdata-logs--lv: UUID="f55c76f1-9cb2-42f3-bb36-1e634529d94d" TYPE=
"ext4" 
[araflyayinde@webserver ~]$ sudo vi /etc/fstab
[araflyayinde@webserver ~]$ sudo vi /etc/fstab
[araflyayinde@webserver ~]$ sudo vi /etc/fstab
[araflyayinde@webserver ~]$ sudo mount -a
[araflyayinde@webserver ~]$ sudo systemctl daemon-reload
[araflyayinde@webserver ~]$ df -h
Filesystem                        Size  Used Avail Use% Mounted on
devtmpfs                          1.9G     0  1.9G   0% /dev
tmpfs                             1.9G     0  1.9G   0% /dev/shm
tmpfs                             1.9G  8.4M  1.9G   1% /run
tmpfs                             1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda2                          20G  2.5G   18G  13% /
/dev/sda1                         200M  6.9M  193M   4% /boot/efi
tmpfs                             374M     0  374M   0% /run/user/1000
/dev/mapper/vg--webdata-apps--lv   14G   41M   13G   1% /var/www/html
/dev/mapper/vg--webdata-logs--lv   14G   42M   13G   1% /var/log

```