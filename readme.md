![image](https://github.com/demainic/cloud/assets/79651776/0765d083-7569-4a79-945f-a1bdad4557c3)

Added Port 
![image](https://github.com/demainic/cloud/assets/79651776/5655aed9-1930-4067-81f1-4f705127ec12)

![image](https://github.com/demainic/cloud/assets/79651776/064dd689-046c-447b-aa71-679100a3011f)

![image](https://github.com/demainic/cloud/assets/79651776/3f470217-3230-49e4-adc1-0ba0f2f7e89c)


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



