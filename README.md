# iSCSI Configuration Implementation with Detailed Steps and Explanations

<br>

## Introduction

-  *This guide demonstrates how to set up an iSCSI (Internet Small Computer Systems Interface) server (target) and client (initiator).*

-  *Goal: Enable the server to provide shared storage over a network, which the client can access as if it were a locally attached disk.*

  

## Benefits of iSCSI

-  Centralized Storage Management: Easy management and allocation of storage -  
   resources.

-  Cost Efficiency: Uses existing TCP/IP networks, reducing the need for specialized 
   hardware.

-   Flexibility: Enables scalable storage solutions.

-  High Availability: Can be configured for redundancy and fault tolerance.
   Prerequisites

## Prerequisites

### 1.Server Machine Requirements:

 -  A Linux-based system (Debian 12 or CentOS 7 VMs recommended).
   
 -  At least one or more additional disks for storage.


### Client Machine Requirements:

  - A Linux-based system (Debian 12 or CentOS 7 VMs recommended). with the open- 
    iscsi or iscsi-initiator-utils package 
     installed.

### 3. Network Connectivity:

  - Both server and client should be connected on the same network.
  - Ensure firewall rules allow iSCSI communication (TCP port 3260).



<br>
<br>


## iSCSI Configuration on Debian 12

  ### *On iSCSI Server VM Machine (Target)*

### 1. Install Required Packages
  - Command (Run on server):

```yml
apt-get update

apt-get install lvm2 targetcli-fb

```

  - *apt-get update: Updates the list of available packages and their versions.*
  - *apt-get install lvm2 targetcli-fb: Installs lvm2 (Logical Volume Manager) for 
     managing disks and targetcli-fb (Target CLI) for configuring iSCSI targets.*

### 2. Prepare the Disk
  - Command (Run on server):

```yml
lsblk
```
  - *lsblk: Lists all available block devices (disks) on the system.
    Command (Run on server):*

```yml
pvcreate /dev/sdb

vgcreate vg_iscsi /dev/sdb

lvcreate -n lv_iscsi-disk-01 -L 1G vg_iscsi

lvs
```

  - *pvcreate /dev/sdb: Creates a physical volume on the /dev/sdb disk.*
  - *vgcreate vg_iscsi /dev/sdb: Creates a volume group named vg_iscsi using 
     /dev/sdb.*
  - *lvcreate -n lv_iscsi-disk-01 -L 1G vg_iscsi: Creates a logical volume of 1 GB 
    size named lv_iscsi-disk-01 inside the vg_iscsi volume group.*
  - *lvs: Displays the details of logical volumes.*

### 3. Configure iSCSI Target
  - *Command (Run on server):*

```yml
targetcli
```

  - *targetcli: Starts the Target CLI tool to manage iSCSI targets.
      Within the Target CLI shell:*

  #### Within the Target CLI shell:
```yml

cd backstores/block
create block1 /dev/mapper/vg_iscsi-lv_iscsi--disk--01

cd ../../iscsi
create iqn.2024-12.cdac.acts.hpcsa.sbm:disk1

cd iqn.2024-12.cdac.acts.hpcsa.sbm:disk1/tpg1/acls
create iqn.1993-08.org.debian:01:84104998b5d

cd ../luns
create /backstores/block/block1

exit

```

  - *cd backstores/block: Navigates to the block storage section in targetcli.*
  - *create block1 /dev/mapper/vg_iscsi-lv_iscsi--disk--01: Creates a block storage        backend called block1 using the logical volume created earlier.*


  - *cd ../../iscsi: Moves to the iSCSI target configuration section.*
  - *create iqn.2024-12.cdac.acts.hpcsa.sbm:disk1: Creates an iSCSI target with the 
     name iqn.2024-12.cdac.acts.hpcsa.sbm:disk1.*

  - *create iqn.1993-08.org.debian:01:84104998b5d: Creates an access control list   
    (ACL) entry for the initiator, allowing access from the specified IQN.*

  - *create /backstores/block/block1: Associates the block storage backend (block1) 
    with the target.*
  - *exit: Exits the targetcli shell.*






### 4. Start and Verify the iSCSI Target Service
  - Command (Run on server):

```yml

systemctl restart rtslib-fb-targetctl
systemctl status rtslib-fb-targetctl
```

  - *systemctl restart rtslib-fb-targetctl: Restarts the iSCSI target service to 
    apply changes.*
  - *systemctl status rtslib-fb-targetctl: Checks the status of the iSCSI target 
     service.*


    <br>
    <br>

    
## On iSCSI Client VM Machine(Initiator)
<br>

### 1. Install Required Package
  - Command (Run on client):

```yml
apt-get install open-iscsi
```
  - *apt-get install open-iscsi: Installs the iSCSI initiator utilities to allow the 
     client machine to connect to iSCSI targets.*

    
### 2. Configure the Initiator
  - Command (Run on client):

```yml
vi /etc/iscsi/initiatorname.iscsi
```

  - *vi /etc/iscsi/initiatorname.iscsi: Edits the initiator name file. Set it to the 
     IQN (iSCSI Qualified Name) provided by the server.*

<br>

### Example:
  - *If the server IQN is iqn.2024-12.cdac.acts.hpcsa.sbm:disk1, the client initiator IQN should look like:*

```yml
InitiatorName=iqn.2022-12.acts.student:306631cea220
```
  *This name must be unique to each client and server pair to avoid conflicts. Once 
   you save the file with the correct initiator name, you can proceed to start the 
  iSCSI service as described in the previous steps.*


<br>


    
    
### 3. Discover and Log in to the iSCSI Target
  - Command (Run on client):

```yml
iscsiadm -m discovery -t sendtargets -p <server-ip>:3260 --login
```

  - *iscsiadm -m discovery -t sendtargets -p <server-ip>:3260 --login: Discovers and 
    logs into the iSCSI target at the specified server IP and port 3260.*

    
### 4. Partition, Format, and Mount the Disk
   - Command (Run on client):

#### 4.1 Identify the new disk:

```yml
fdisk -l
```
  - *fdisk -l: Lists all available disk partitions, including the new iSCSI disk.


#### 4.2 Partition the disk:
  
- Command (Run on client):

```yml

fdisk /dev/sdx

# Press `n` for a new partition, then `w` to write changes.

```
  - *fdisk /dev/sdx: Starts partitioning the new iSCSI disk (/dev/sdx).*


#### 4.3 Format the partition:
    
- Command (Run on client):

```yml
mkfs.xfs -f /dev/sdx
```
  - *mkfs.xfs -f /dev/sdx: Formats the partition /dev/sdx with the XFS filesystem.

#### 4.4 Mount the disk:

  - Command (Run on client):*

```yml
mkdir /mnt/disk-1

mount /dev/sdx /mnt/disk-1

df -Th
```

  - *mkdir /mnt/disk-1: Creates a directory to mount the iSCSI disk.*
  - *mount /dev/sdx /mnt/disk-1: Mounts the iSCSI disk to the specified directory.*
  - *df -Th: Displays the filesystem information to confirm the disk is mounted 
     correctly.*


### 5. Manage iSCSI Sessions
```yml
iscsiadm -m session

iscsiadm -m session -P 1

iscsiadm -m node --logoutall=all
```

  - *iscsiadm -m session: Shows the current iSCSI sessions.*
  - *iscsiadm -m node --logoutall=all: Logs out from all iSCSI sessions.*



### 5.1 Firewall Considerations.

  - *Make sure that the firewall is properly configured to allow iSCSI traffic. On 
    both the server and client machines, ensure that port 3260 is open, and disable 
    firewalls if required:*

 ```yml
 
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```





### 6. Verification: Create and Check a File
#### 6.1. Verify Disk Access
  - Command (Run on client):

```yml
cd /mnt/disk-1
touch testfile.txt
ls -lh
```
  - *cd /mnt/disk-1: Navigates to the mounted iSCSI disk directory.
  - touch testfile.txt: Creates a new file named testfile.txt in the iSCSI-mounted 
    directory.*
  - *ls -lh: Lists the files in the directory to confirm that the testfile.txt file 
     has been created successfully.*
    
#### 6.2. Verify File Persistence
  *To verify that the file persists even after unmounting and remounting the disk, 
   follow these steps:*  

  - Command (Run on client):

```yml
umount /mnt/disk-1
mount /dev/sdx /mnt/disk-1
ls -lh
```

  - *umount /mnt/disk-1: Unmounts the iSCSI disk.*
  - *mount /dev/sdx /mnt/disk-1: Remounts the iSCSI disk.*
  - *ls -lh: Lists the files in the directory to confirm that testfile.txt is still 
    present after remounting*

    






<br>
<br>

## Conclusion

*With the above implementation, we have successfully configured an iSCSI server and client. we also verified disk access by creating a file on the mounted iSCSI disk and ensured that the data persists across reboots. Now, the client can access storage from the server as if it were a locally connected disk.*

<br>
<br>




# Summary of Key Concepts

- iSCSI Target: Provides remote storage access to clients. It's configured on the 
   server side (with tools like targetcli).

- iSCSI Initiator: The client that connects to the iSCSI target. It is configured 
  using the open-iscsi package.

- LVM (Logical Volume Management): Used to manage storage on the server side, 
  enabling flexibility in creating and resizing storage volumes.

- Logical Unit Number (LUN): A unique identifier used to map storage devices to iSCSI targets.


<br>


### Benefits:

- Efficient network-based storage sharing.

- Flexibility in expanding and managing storage resources.

- Simplifies storage provisioning in virtualized environments.



 
