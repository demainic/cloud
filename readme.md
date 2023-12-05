### Task 1

#### Setup the VM's

First you need to set up 2 VM's, one named "target" and one named "initiator" 

![image](https://github.com/demainic/cloud/assets/79651776/0765d083-7569-4a79-945f-a1bdad4557c3)

Pay attention to add the Port 3260 to the security Group

#### Connect to the VM's

Open two Terminals and connect to both VM's, the target and the initiator:
```
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa centos@160.85.31.236
```

#### a) Create a compressed (lz4) ZFS pool with 60% storage efficiency and a total of 300 MB of usable storage
The following commands have to be run on the target.

1. make a new directory
2. create images that have 300mb Storage with 60%
```
for i in {1..5}; do fallocate -l 100M device$i.img; done
```
3. make a pool out of them
```
sudo zpool create mypool2 ~/newDevice/device*.img
```
4. set the lz4 compression
```
sudo zfs set compression=lz4 mypool2
```
5. confirm the compression
```
sudo zfs get compression mypool2
```
![image](https://github.com/demainic/cloud/assets/79651776/16a2bcb6-f125-4906-a586-4fed2e3995a1)


#### b) Add redundancy to the pool by introducing one hot spare drive
The following commands have to be run on the target.
1. create an additional image
```
fallocate -l 100M ~/newDevice/hotspare.img
```
2. add the spare image   
```
sudo zpool add mypool2 spare ~/newDevice/hotspare.img
```
3. verify the status
```
zpool status mypool2
```
![image](https://github.com/demainic/cloud/assets/79651776/3bb2866c-4f04-4423-80e8-dea7b213bb76)

#### c) Create a 30MB block device (ZFS volume) on the “mypool” pool with the name “myvolume”

1. create a volume with 30MB
```
sudo zfs create -V 30M mypool2/myvolume
```
2. verify the volume
```
sudo zfs list mypool2/myvolume
```
3. verify the size of the created volume
```
sudo zfs get volsize mypool2/myvolume
```

#### d) Expose the “myvolume” block device over iscsi

1. clear existing iSCSI configuration
```
sudo targetcli clearconfig confirm=true
```
![image](https://github.com/demainic/cloud/assets/79651776/d9abad8f-66e3-45c7-9ee0-b152b327f6c5)

2. set up the backstore block device
```
sudo targetcli /iscsi set discovery_auth enable=0
```
![image](https://github.com/demainic/cloud/assets/79651776/b081cf86-e5d5-4a66-957b-869fd8207448)

3.create an iSCSI target:
```
sudo targetcli /backstores/block create name=blockmyvolume dev=/dev/zvol/mypool2/myvolume
sudo targetcli /iscsi create
```
![image](https://github.com/demainic/cloud/assets/79651776/6305d522-86b2-478b-ba9f-292472e31c43)


4. find the IQN
```
sudo targetcli ls /iscsi/ | grep iqn
```
![image](https://github.com/demainic/cloud/assets/79651776/3f38fc63-a3e9-4bf9-9b48-df6465cfc99c)

5. Assign the target IQN to an environment variable:
```
export TARGET_IQN=iqn.2003-01.org.linux-iscsi.target.x8664:sn.8d821927272c 
```
![image](https://github.com/demainic/cloud/assets/79651776/41588b31-9219-4945-b94d-259ea5aeb5b4)

6. Create a LUN for the block device:
```
sudo targetcli /iscsi/${TARGET_IQN}/tpg1/luns create /backstores/block/blockmyvolume
```
![image](https://github.com/demainic/cloud/assets/79651776/48caee56-de3a-4ca6-9904-7ac5f083ab36)


7. Disable authentication:
```
sudo targetcli /iscsi/$TARGET_IQN/tpg1 set attribute authentication=0
```
![image](https://github.com/demainic/cloud/assets/79651776/4fa6221d-cb17-4cea-b1b7-ce44c98ce4eb)

8. on the **initiator** read the IQN name
```
sudo cat /etc/iscsi/initiatorname.iscsi
```
![image](https://github.com/demainic/cloud/assets/79651776/bb0ae994-643a-4c7b-92fb-52b0ef745c35)

9.Back on the **target** create the acl rule
```
sudo targetcli /iscsi/${TARGET_IQN}/tpg1/acls create iqn.1994-05.com.redhat:9fbf494cf22d
```
![image](https://github.com/demainic/cloud/assets/79651776/0f5a2f5c-5007-4a41-a72f-6c55ac45f296)

10. save and restart the target
```
sudo targetcli saveconfig
sudo service target restart
```
![image](https://github.com/demainic/cloud/assets/79651776/ed344a34-2935-40c4-8f98-d2b845176ce4)


### Mount iSCSI volume on initiator
Everything except Point 7 is on the initiator
1. Clean any existing iSCSI records
```
sudo iscsiadm -m node --logout
sudo iscsiadm -m node -o delete
```
![image](https://github.com/demainic/cloud/assets/79651776/586174f8-99bf-42ec-9cc5-596644da3d6a)

2. Discover any iSCSI target 
```
sudo iscsiadm -m discovery -t st -p 10.10.2.10
```
![image](https://github.com/demainic/cloud/assets/79651776/7dca1ba1-f87b-48a3-9e25-d5a71d8b4793)

3. Access the iSCSI storage
```
sudo iscsiadm -m node --targetname "iqn.2003-01.org.linux-iscsi.target.x8664:sn.8d821927272c" --portal "10.10.2.10:3260" --login
```
4.  Verify that a new block device is available on the system
```
dmesg
```
![image](https://github.com/demainic/cloud/assets/79651776/6ef154a8-b600-457d-8398-412b0f0d1aeb)

```
sudo fdisk -l
```
![image](https://github.com/demainic/cloud/assets/79651776/a6f52349-73d2-4144-949d-a2175c1c19d1)


5. Format the block devices and mount them on a destination folder you created
![image](https://github.com/demainic/cloud/assets/79651776/69eb4b33-b76f-4f5e-bf0e-40303753af76)
![image](https://github.com/demainic/cloud/assets/79651776/6c2d05b9-d3fe-4e1c-bcfa-e0a255532c7b)

6. write a text file to the 
```
sudo bash -c "echo 'hello world' > test.txt"
```
![image](https://github.com/demainic/cloud/assets/79651776/14df3139-5fec-4d67-bc41-6a76b97835b6)

7. unmount device
```
sudo umount ~/iscsi_mount
```
8. disconnect
```
sudo iscsiadm -m node --logout
```
9. reconnect
```
sudo iscsiadm -m node --targetname "iqn.2003-01.org.linux-iscsi.target.x8664:sn.8d821927272c" --portal "10.10.2.10:3260" --login
```
10. remount
```
sudo mount /dev/sda ~/iscsi_mount
```
11. On the intitatior, write a small script that writes data into a file on your remote block-level device,
then sleeps for 5s, and continues to do so N times
```
for i in {1..10}; do
    echo "Data $i" | sudo tee -a ~/iscsi_mount/data.txt
    sleep 5
done
```
![image](https://github.com/demainic/cloud/assets/79651776/6dc0835d-a69d-4e5a-ab21-c1b10ea52b80)

12. install tcpdump on target and start capturing data
```
sudo yum install tcpdump
sudo tcpdump -i eth0 port 3260 -w iscsi-capture.pcap
```
13. Download file local
```
scp -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa centos@160.85.31.188:iscsi-capture.pcap /Downloads/iscsi-capture.pcap
```
![image](https://github.com/demainic/cloud/assets/79651776/1bddab2b-862d-4675-90d7-e5a0fa5ff5c0)

13. use wireshark to analyze the data
-> could not download the file

14.Simulate device failures by zeroing out the disk images we made with fallocate
on target and observer:
```
dd if=/dev/zero of=block1 bs=4M count=1
sudo zpool scrub mypool
sudo zpool status mypool
```
![image](https://github.com/demainic/cloud/assets/79651776/f449fb10-8c37-4cf8-ac20-94467b34b952)


### Task 2

#### Overview

automate.sh is a Bash script designed to automate the process of creating ZFS volumes and configuring them as iSCSI targets. It simplifies the setup of block storage in a cloud environment.

#### Prerequisites

- A Linux system with ZFS and targetcli installed.
- An existing ZFS pool (e.g., mypool).
- The TARGET_IQN environment variable must be set to the iSCSI Qualified Name (IQN) of your iSCSI target.
```
Export TARGET_IQN = {target_IQN}
```
To get the Target_IQN:
```
sudo targetcli ls /iscsi/ | grep iqn
```

#### Usage
1. **VOLUME_NAME**: The name of the volume to be created.
2. **VOLUME_SIZE**: The size of the volume (e.g., 10MB, 1GB).

Example:
```
./automate.sh xyz 20MB
```

#### Changes to automate.sh
    1. Create a ZFS Volume:
      This command will create a ZFS volume with the specified name and size within your ZFS pool. Assuming your pool is named mypool, you will append the volume name to the pool name.

    2. Create a Backstore:
    This step involves creating a backstore in the targetcli configuration for the ZFS volume you created. The backstore name convention will be blockVOL, where VOL is the volume name.

    3. Create LUNs:
    After creating the backstore, the next step is to create a Logical Unit Number (LUN) for it. This makes the storage available as an iSCSI target.

**Final automate.sh**

```
#!/bin/bash

VOLUME_NAME=$1
VOLUME_SIZE=$2

if [ "$#" -ne 2 ]; then
    echo "Usage: ./automate.sh <volume-name> <volume-size>"
    echo "Example: ./automate.sh light 10MB"
    exit 1
fi

[ -z "$TARGET_IQN" ] && { echo "Need to set TARGET_IQN "; exit 1; }

# Create ZFS volume
sudo zfs create -V $VOLUME_SIZE mypool/${VOLUME_NAME}

# Create a backstore
sudo targetcli /backstores/block create name=block${VOLUME_NAME} dev=/dev/zvol/mypool/${VOLUME_NAME}

# Create LUNs
sudo targetcli /iscsi/${TARGET_IQN}/tpg1/luns create /backstores/block/block${VOLUME_NAME}

# Save the configuration
sudo targetcli / saveconfig

echo "Volume $VOLUME_NAME with size $VOLUME_SIZE created and configured as an iSCSI target."
```

#### Transfer to the target

The script can now be transferd to the target:
```
 scp -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa Downloads/automate.sh centos@160.85.31.188:cloud-lab
```

#### Run

Let it run as seen under usage.
Don't forget to give the shell script executable:
```
chmod +x automate.sh 
```
After that, you can run the following commands to make sure, the the LUNs are created:
```
sudo targetcli ls
```
![image](https://github.com/demainic/cloud/assets/79651776/03b11709-031d-4ee9-b9eb-38796370504b)

```
sudo zfs list
```
![image](https://github.com/demainic/cloud/assets/79651776/63d93375-dd62-433e-9838-3332f8ddb6bb)



