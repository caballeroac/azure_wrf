# Running WRF with CycleCloud
Objective of this repository is to deploy a HPC cluster with Azure CycleCloud and run WRF.
## Pre-requisites

* Azure Subcription
* Quota allocated for the specialized HPC vm HB120v3 
* 

## Installation

TBD

### 1. CycleCloud Installation and Configuration


#### 1.1 CycleCloud Installation and Configuration


### 2. Slurm 

**cloud-init section**

The following cloud-init section has to be present for all the partitions defined in CycleCloud. In this case HPC partition.
```
#!bin/bash
sudo yum -y install epel-release
sudo yum -y install htop
sudo yum -y groupinstall "Development Tools"
sudo yum -y install python-pathlib
sudo yum -y install screen
sudo yum -y install clang
if [[ -e /dev/nvme0n1 ]]; then
  # Setup RAID0 on HBv3 NVMe disks:
  mdadm --create --verbose /dev/md0 --level=stripe --raid-devices=2 /dev/nvme0n1 /dev/nvme1n1
  mkfs.ext4 /dev/md0
  mkdir -p /mnt/nvme
  mount /dev/md0 /mnt/nvme
  mkdir /mnt/nvme/scratch
  chmod 777 -R /mnt/nvme
fi

setenforce 0
# Disable SELinux
cat  << EOF > /etc/selinux/config
SELINUX=disabled
EOF
```

### 3. Spack Installation and Configuration

We will not use the default Spack installation, instead we will use the installation procedure described in [AzureHPC](https://github.com/Azure/azurehpc.git) 

From the Slurm Head node we will create the `/shared/apps` direcotry and will clone the AzureHPC reposiory. 
```
sudo mkdir /shared/apps/
sudo chown azureuser:azureuser /shared/apps/
cd /shared/apps
sudo yum install git -y 
git clone https://github.com/Azure/azurehpc.git
```

Now we have to make a change before we install Spack:
* Edit the file `/shared/apps/azurehpc/apps/spack/build_spack.sh` and change the `SHARED_APP` variable to point to to /shared/app. So `SHARED_APP=/shared/apps`

Install Spack with the following command:
```
cd /shared/apps/azurehpc/apps/spack
./build_spack.sh hbv2
```

Finally, make the last change before installing applications:
* Change the build_stage for a fast complilation of the applications. We will use the NVMe drives allocated in the HB120v3 nodes as scratch, so in the file `/shared/apps/spack/0.16.0/spack/etc/spack/defaults/config.yaml` make the following changes:
```
build_stage:
    - /mnt/nvme/scratch   # Note that the directory is created with cloud-init on VM startup.
```




