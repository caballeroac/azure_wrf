# Running WRF with CycleCloud
Objective of this repository is to deploy a HPC cluster with Azure CycleCloud and run WRF.

* Architecture
![Architecture](./images/Image1.JPG)


## Pre-requisites

* Azure Subcription
* Quota allocated for the specialized HPC vm HB120v3 
* Azure NetApp Files / Azure Files (NFS) / NFS on Azure Blob / Parallel Filesystem (Lustre/GPFS/BeeGFS...)
* Service Principal contributor role or Managed Identity permissions
* TBD

## CycleCloud

TBD

### 1. CycleCloud Installation

TBD 
#### 1.1 CycleCloud Configuration

TBD

### 2. HPC Cluster


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
sudo yum -y install python3
sudo yum -y install tcsh

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
Also change the number of threads for compilation:
```
build_jobs: 16
```

### 4. WRF Installation

We will run WRF on the HB series so we will use one of the HB120v3 as "compilation node" to be consistent with the architecture. So first thing is to sumbit an Interactive job to allocate a node in the cluster to compile WRF and the dependencies needed:

```
screen  # Optional, but is a good practice in case

# With Slurm 
salloc -N1 # This command will take 8-10 minutes until
srun hostname
ssh ip-0A060005   # Replace ip-0A060005 with the output of the srun commnad

# With PBSPro
qsub -I -q workq
```

Load spack environment settings:
```
. /shared/apps/spack/0.16.0/spack/share/spack/setup-env.sh
```

Now we can check what options are available for WRF with `spack info wrf` before we start:
```
Package:   wrf

Description:
    The Weather Research and Forecasting (WRF) Model is a next-generation
    mesoscale numerical weather prediction system designed for both
    atmospheric research and operational forecasting applications.

Homepage: https://www.mmm.ucar.edu/weather-research-and-forecasting-model

Maintainers: @MichaelLaufer @ptooley

Tags:
    None

Preferred version:
    4.2        https://github.com/wrf-model/WRF/archive/v4.2.tar.gz

Safe versions:
    4.2        https://github.com/wrf-model/WRF/archive/v4.2.tar.gz
    4.0        https://github.com/wrf-model/WRF/archive/v4.0.tar.gz
    3.9.1.1    https://github.com/wrf-model/WRF/archive/V3.9.1.1.tar.gz

Variants:
    Name [Default]            Allowed values          Description
    ======================    ====================    ========================

    build_type [dmpar]        serial, smpar,
                              dmpar, dm+sm
    compile_type [em_real]    em_real,
                              em_quarter_ss,
                              em_b_wave, em_les,
                              em_heldsuarez,
                              em_tropical_cyclone,
                              em_hill2d_x,
                              em_squall2d_x,
                              em_squall2d_y,
                              em_grav2d_x,
                              em_seabreeze2d_x,
                              em_scm_xy
    nesting [basic]           no_nesting, basic,
                              preset_moves,
                              vortex_following
    pnetcdf [on]              on, off                 Parallel IO support
                                                      through Pnetcdf library

Installation Phases:
    configure    build    install

Build Dependencies:
    hdf5    libpng    libtool  mpi       netcdf-fortran   perl       tcsh  zlib
    jasper  libtirpc  m4       netcdf-c  parallel-netcdf  pkgconfig  time

Link Dependencies:
    hdf5    libpng    mpi       netcdf-fortran   perl
    jasper  libtirpc  netcdf-c  parallel-netcdf  zlib

Run Dependencies:
    None

Virtual Packages:
    None

```

Load gcc 9.2 compilers and OpenMPI before we start with wrf installation:
```
module avail
module load gcc-9.2.0
module load mpi/openmpi

# spack external find (optional)
```

Now install wrf application with the following options:
``` 
alias python='/usr/bin/python3.6'
spack install wrf %gcc@9.2.0 ^openmpi@4.1.0    # Note that could take few hours to install and compile all dependencies.
```
It will give a error in line 292 when building. There is a bug in the WRF version, just replace the lines as indicated below:
```
         # num of compile jobs capped at 20 in wrf
-        num_jobs = str(min(int(make_jobs, 10)))
+        num_jobs = str(min(int(make_jobs), 10))
 
         # Now run the compile script and track the output to check for
         # failure/success We need to do this because upstream use `make -i -k`
```
Ammend in the same file in line 282
```
result_buf = csh( 
     "./compile", 
     "-j", 
     num_jobs, 
     self.spec.variants["compile_type"].value, 
     output=str, 
     error=str 
 ) 
```
Finally run again the command to install spack: 
```
spack install wrf %gcc@9.2.0 ^openmpi@4.1.0
```

Now WRF should be available in the modules/spack:
```
[azureuser@ip-0A060007 wrf]$ spack find wrf
==> 1 installed package
-- linux-centos7-zen2 / gcc@9.2.0 -------------------------------
wrf@4.2

$ spack load wrf@4.2

# Locate and go to WRF installation directory
$ spack cd -i wrf@4.2
$ pwd
$ ls -l ./main/wrf.exe
-rwxr-xr-x. 1 azureuser azureuser 46545352 Jul  8 16:50 ./main/wrf.exe
```

#### Benchmarks
The Conus 2.5Km test is available from www2.mmm.ucar.edu 
```
wget https://www2.mmm.ucar.edu/wrf/src/conus2.5km.tar.gz
```


### WRF 3.9.1 ####
```

LC_CTYPE="en_US.UTF-8"
LC_ALL="en_US.UTF-8"
LANG="en_US.UTF-8"


spack load  parallel-netcdf
NETCDF=`spack location -i parallel-netcdf`
export NETCDF=$NETCDF/bin

spack load jasper
JASPER=`spack location -i jasper`
export JASPERLIB=$JASPER/lib64
export JASPERINC=$JASPER/include

$JASPERLIB or $JASPERINC not found in environment. Using default values for library paths..

spack install wps@3.9.1 %gcc@9.2.0 ^openmpi@4.1.0

```

#### New x86_64 compilation 
```
spack uninstall -a target=x86_64
#spack uninstall -a target=zen2
. /shared/apps/spack/0.16.0/spack/share/spack/setup-env.sh
module purge
spack clean --all


sudo yum -y install tmux scl file gcc gcc-gfortran gcc-c++ glibc.i686 libgcc.i686 libpng-devel jasper \
  jasper-devel hostname m4 make perl tar bash tcsh time wget which zlib zlib-devel \
  openssh-clients openssh-server net-tools fontconfig libgfortran libXext libXrender \
  ImageMagick sudo epel-release git
  
module load gcc-9.2.0
module load mpi/openmpi

spack install hdf5+fortran+hl %gcc@9.2.0 ^openmpi
module load hdf5-1.10.7-gcc-9.2.0-ecjkzyn
spack install netcdf-fortran ^hdf5+fortran+hl %gcc@9.2.0 ^openmpi

module load netcdf-c-4.7.4-gcc-9.2.0-xsf3sp2
module load netcdf-fortran-4.5.3-gcc-9.2.0-nycl2jv

```
Environment Settings:

```
export PATH=$(spack location -i netcdf-c)/bin:$PATH
export PATH=$(spack location -i netcdf-fortran)/bin:$PATH
export NETCDF=$(spack location -i netcdf-fortran)
export HDF5=$(spack location -i hdf5)
export LD_LIBRARY_PATH=$HDF5/lib:$NETCDF/lib:$LD_LIBRARY_PATH
NETCDF_C=$(spack location -i netcdf-c)


setenv HFD5 /shared/apps/spack/hbv2/linux-centos7-zen2/gcc-9.2.0/hdf5-1.10.7-ecjkzyn65ypusunibu7uxffvdiqhqnmm
setenv NETCDF /shared/apps/spack/hbv2/linux-centos7-zen2/gcc-9.2.0/netcdf-fortran-4.5.3-nycl2jvt53u6bzgjrocfbgc2iaj2cwaf

setenv PATH $PATH\:`spack location -i netcdf-c`
setenv PATH $PATH\:`spack location -i netcdf-fortran`
setenv NETCDF_C `spack location -i netcdf-c`


```


#### TEST
```
qsub -I -q workq
csh
source /shared/apps/spack/0.16.0/spack/share/spack/setup-env.csh

module load gcc-9.2.0
module load mpi/openmpi
module load hdf5-1.10.7-gcc-9.2.0-ecjkzyn
module load netcdf-c-4.7.4-gcc-9.2.0-xsf3sp2
module load netcdf-fortran-4.5.3-gcc-9.2.0-nycl2jv

setenv PATH $PATH\:`spack location -i netcdf-c`
setenv PATH $PATH\:`spack location -i netcdf-fortran`
setenv NETCDF_C `spack location -i netcdf-c`

```






