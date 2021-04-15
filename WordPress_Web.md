

In this project, we'd demonstarte a Three-tier Architecture while also ensuring that the disks used to store files on the Linux servers are adequately partitioned and managed through programs such as gdisk and LVM respectively.

### Prerequisite
Two GCP VM (CentOS8). One to serve as a web server (This is where you will install Wordpress) and the other a database (DB) server

## Step 1 — Prepare the Web Server

1. Launch an VMinstance that will serve as “Web Server”. Create 3 volumes in the same availability zone as your Web Server EC2, each of 20 GiB (In case you're wondering how to go about that on GCP, this walks you through - <https://cloud.google.com/compute/docs/disks/add-persistent-disk#console>)
   
2. Use lsblk command to inspect what block devices are attached to the server. Notice names of your newly created devices. All devices in Linux reside in /dev/ directory. Inspect it with ls /dev/ and make sure you see all 3 newly created block devices there - their names will likely be sdb, sdc, sdd.

`$   sudo lsblk`

```
Output:

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk 
├─sda1   8:1    0  200M  0 part /boot/efi
└─sda2   8:2    0 19.8G  0 part /
sdb      8:16   0   20G  0 disk 
sdc      8:32   0   20G  0 disk 
sdd      8:48   0   20G  0 disk 

```

3. Use df -h command to see all mounts and free space on your server then Use the gdisk utility to create a single partition on each of the 3 disks

`sudo gdisk /dev/sdb`

Follow the prompts with an "n" 'enter'x3, then 'p' and a 'w' to write and save.

```
 GPT fdisk (gdisk) version 1.0.3

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries.

Command (? for help branch segun-edits: p
Disk /dev/xvdf: 20971520 sectors, 10.0 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): D936A35E-CE80-41A1-B87E-54D2044D160B
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 20971486
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20971486   10.0 GiB    8E00  Linux LVM

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): yes
OK; writing new GUID partition table (GPT) to /dev/xvdf.
The operation has completed successfully.
Now,  your changes has been configured succesfuly, exit out of the gdisk console and do the same for the remaining disks.
```

When you're done run, the command below to see the changes that's happened.

`$ sudo gdisk -l /dev/sdb`

Repeat the same process for the other drives i.e /dev/sdc and /dev/sdd. Then run:

`sudo lsblk`
```
Output: 

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
```

4. Install lvm2 package using sudo yum install lvm2. Run sudo lvmdiskscan command to check for available partitions.Then run this to check if the installation went well. 

`$ which lvm`

5. Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

`$ sudo pvcreate /dev/sdb1 /dev/sdc1 /dev/sdd1`

```
Output: 

  Physical volume "/dev/sdb1" successfully created.
  Physical volume "/dev/sdc1" successfully created.
  Physical volume "/dev/sdd1" successfully created.
```
Verify that your Physical volume has been created successfully by running:

`$ sudo pvs`

```
Output:

  PV         VG Fmt  Attr PSize  PFree 
  /dev/sdb1     lvm2 ---  10.00g 10.00g
  /dev/sdc1     lvm2 ---  10.00g 10.00g
  /dev/sdd1     lvm2 ---  10.00g 10.00g
```
6. Afterwards, use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG *vg-webdata*

`$ sudo vgcreate vg-webdata /dev/sdb1 /dev/sdc1 /dev/sdd1`

```
Output:

  Volume group "vg-webdata" successfully created
```

Verify that your VG has been created successfully by running 
`sudo vgs`

```
Output:

  VG         #PV #LV #SN Attr   VSize   VFree  
  vg-webdata   3   0   0 wz--n- <29.99g <29.99g
```

7. Use lvcreate utility to create 2 logical volumes. apps-lv (Use half of the PV size), and logs-lv Use the remaining space of the PV size. NOTE: communication-lv will be used to store data for the Website while, logs-lv will be used to store data for logs.

`$ sudo lvcreate -n communication-lv -L 14G vg-webdata`

```
Output:

  Logical volume "communication-lv" created.
```

`$ sudo lvcreate -n logs-lv -L 14G vg-webdata`

```
Logical volume "logs-lv" created.
```

Verify that your Logical Volume has been created successfully by running

`sudo lvs`

```
Output:

  LV               VG         Attr       LSize  Pool Origin Data%  Meta%  Move Log 
Cpy%Sync Convert
  communication-lv vg-webdata -wi-a----- 14.00g                                    
                
  logs-lv          vg-webdata -wi-a----- 14.00g  
```

Verify the entire setup 
`$ sudo lsblk`

```
Output: 

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
```

8. Next, use mkfs.ext4 to format the logical volumes with ext4 filesystem

```
sudo mkfs -t ext4 /dev/webdata-vg/communication-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```

9. Create /home/recovery/logs to store backup of log data

`sudo mkdir -p /home/recovery/logs`

10. Then, mount /var/www/html on communication-lv logical volume

`sudo mount /dev/webdata-vg/apps-lv /var/www/html/`

Verify the setup now using 

`$ df -h`

```
Output: 

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
```

11. Use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system

`sudo rsync -av /var/log /home/recovery/logs/`

12. You can now mount /var/log on logs-lv logical volume.

> Note that all the existing data on /var/log will be deleted, which is why step above is very crucial

`sudo mount /dev/vg-webdata/logs-lv /var/log`

13. Restore log files back into /var/log directory

`sudo rsync -av /home/recovery/logs/log/. /var/log`

14. The UUID of the device will be used to update the /etc/fstab file. Update /etc/fstab file so that the mount configuration will persist after restart of the server.

`$ sudo blkid`

```
Output:

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
```

`sudo vi /etc/fstab`

Update /etc/fstab in this format using your own UUID.

*fstab image

Test the configuration and reload the daemon

```
$ sudo mount -a
$ sudo systemctl daemon-reload
```

Verify your setup by running:
`$ df -h`

```
Output:

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

## Step 2 — Prepare the Database Server

We'd launch a second RedHat instance that will have a designated role - ‘DB Server’.We'd repeat the same exact steps as for the Web Server, but instead of *apps-lv*, create db-lv and mount it to /db directory instead of /var/www/html/ as was in the web server.
After the entire process, you should end up with something like this when you run:

`$ df -h`

```
Output:

Filesystem                       Size  Used Avail Use% Mounted on
devtmpfs                         1.9G     0  1.9G   0% /dev
tmpfs                            1.9G     0  1.9G   0% /dev/shm
tmpfs                            1.9G  8.4M  1.9G   1% /run
tmpfs                            1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda2                         20G  2.5G   18G  13% /
/dev/sda1                        200M  6.9M  193M   4% /boot/efi
tmpfs                            374M     0  374M   0% /run/user/1000
/dev/mapper/vg--database-db--lv   20G   45M   19G   1% /db

```

## Step 3 — Install Wordpress on your Web Server

1. Firstly update the server

`sudo yum -y update`

2. Install wget, Apache and it’s dependencies and start Apache.

`sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`

```
sudo systemctl enable httpd
sudo systemctl start httpd
```

3. Install PHP and it’s depemdencies and restart Apache.

```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
```

```
Output:

Remi's Modular repository for Enterprise Lin 4.0 MB/s | 745 kB     00:00    
Safe Remi's RPM repository for Enterprise Li  10 MB/s | 1.7 MB     00:00    
CentOS Linux 8 - AppStream
Name    Stream        Profiles                      Summary                  
php     7.2 [d][e]    common [d], devel, minimal    PHP scripting language   
php     7.3           common [d], devel, minimal    PHP scripting language   
php     7.4           common [d], devel, minimal    PHP scripting language   
Remi's Modular repository for Enterprise Linux 8 - x86_64
Name    Stream        Profiles                      Summary                  
php     remi-7.2      common [d], devel, minimal    PHP scripting language   
php     remi-7.3      common [d], devel, minimal    PHP scripting language   
php     remi-7.4      common [d], devel, minimal    PHP scripting language   
php     remi-8.0      common [d], devel, minimal    PHP scripting language   
Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled
```

`sudo systemctl restart httpd`

Enter your ip into your browser and you should see Apache's default page

![](https://github.com/Arafly/WordPress_Web_Solution/blob/master/assets/apache.PNG)

4. Download wordpress and copy wordpress to var/www/html

```
mkdir wordpress
cd   wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
cp -R wordpress /var/www/html/
```


5. Next is to configure some SELinux Policies

 ```
 sudo chown -R apache:apache /var/www/html/wordpress
 sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
 sudo setsebool -P httpd_can_network_connect=1
 ```

## Step 4 — Install MySQL on your DB Server
```
sudo yum update
sudo yum install mysql-server
```

Verify that the service is up and running by using `sudo systemctl status mysqld`, if it is not running, restart the service and enable it so it will be running even after reboot:

```
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```

For more secure installation you should run:
`$ sudo mysql_secure_installation`

## Step 5 — Configure DB to work with WordPress

We'd need to create a database, a wordpress user, and grant the user all privileges to the db.

```
sudo mysql -u root -p
mysql> CREATE DATABASE wordpress;
mysql> CREATE USER 'wp_user'@`%` IDENTIFIED BY 'mypass';
mysql> GRANT ALL ON wordpress.* TO 'wp_user'@'%';

mysql> SHOW DATABASES;
mysql> exit
```

Remember to address of the webserver to the *bind-address* and also open up the port 3306 on the firewall of your VM. You can decide to allow inbound traffic from everywhere (0.0.0.0/0) or specify just the ip of the web server. It's your choice (I chose the former as this VMs are ephemeral, so security is not really a concern).

## Step 6 — Configure WordPress to connect to remote database.

Install MySQL client and test that you can connect from your Web Server to your DB server by using mysql-client
sudo yum install mysql
sudo mysql -u admin -p -h <DB-Server-Private-IP-address>
Verify if you can successfully execute SHOW DATABASES; command and see a list of existing databases.
Change permissions and configuration so Apache could use WordPress:
Enable TCP port 80 in Inbound Rules configuration for your Web Server EC2 (enable from everywhere 0.0.0.0/0 or from your workstation’s IP)
Try to access from your browser the link to your WordPress http://<Web-Server-Public-IP-Address>/wordpress/

```
[araflyayinde@dbserver ~]$ sudo systemctl start mysqld
[araflyayinde@dbserver ~]$ sudo systemctl status mysqld
● mysqld.service - MySQL 8.0 database server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; disabled; vendor pr>
   Active: active (running) since Wed 2021-04-14 11:16:47 UTC; 4s ago
  Process: 139095 ExecStartPost=/usr/libexec/mysql-check-upgrade (code=exited,>
  Process: 138969 ExecStartPre=/usr/libexec/mysql-prepare-db-dir mysqld.servic>
  Process: 138944 ExecStartPre=/usr/libexec/mysql-check-socket (code=exited, s>
 Main PID: 139050 (mysqld)
   Status: "Server is operational"
    Tasks: 39 (limit: 23411)
   Memory: 427.0M
   CGroup: /system.slice/mysqld.service
           └─139050 /usr/libexec/mysqld --basedir=/usr
Apr 14 11:16:40 dbserver systemd[1]: Starting MySQL 8.0 database server...
Apr 14 11:16:41 dbserver mysql-prepare-db-dir[138969]: Initializing MySQL data>
Apr 14 11:16:47 dbserver systemd[1]: Started MySQL 8.0 database server.
lines 1-16/16 (END)
```

## Connect to DB
```
[araflyayinde@webserver ~]$ sudo mysql -u wordpress_user -p -h 10.154.0.5
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.21 Source distribution
Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement
.
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| wordpress          |
+--------------------+
2 rows in set (0.01 sec)
mysql> 
```