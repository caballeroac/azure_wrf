# Running WRF with CycleCloud
Objective of this repository is to deploy a HPC cluster with Azure CycleCloud and run WRF.

* Architecture
![Architecture](./images/Image1.JPG)


## Pre-requisites

* Azure Subcription
* Quota allocated for the specialized HPC vm HB120v3 
* Azure NetApp Files / Azure Files (NFS) / NFS on Azure Blob / Parallel Filesystem (Lustre/GPFS/BeeGFS...)
* Service Principal contributor role or Managed Identity permissions for CycleCloud
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
sudo yum -y install java
sudo yum -y install perl-devel 


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


### 1. WRF 3.9.1 ###

```
$ screen 
$ qsub -I -q workq -l walltime=05:00:00

$ sudo yum -y install tmux scl file gcc gcc-gfortran gcc-c++ glibc.i686 libgcc.i686 libpng-devel jasper \
  jasper-devel hostname m4 make perl tar bash time wget which zlib zlib-devel \
  openssh-clients openssh-server net-tools fontconfig libgfortran libXext libXrender \
  ImageMagick sudo epel-release git help2man
  
  
# COMPILERS TESTS 

$ mkdir -p $HOME/wrfpoc/zen3/Build_WRF
$ mkdir -p $HOME/wrfpoc/zen3/TESTS

$ which gfortran
/user/bin/gfortran
$ which cpp
/user/bin/cpp
$ which gcc
/user/bin/gcc

$ gcc --version
gcc (Ubuntu 5.3.1−14ubuntu2.1) 5.3.1 20160413
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY of FITNESS FOR A PARTICULAR PURPOSE.

$ cd $HOME/wrfpoc/zen3/TESTS
$ wget http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/Fortran_C_tests.tar
$ tar -xvf Fortran_C_tests.tar

# Test 1
$ gfortran TEST_1_fortran_only_fixed.f
$ ./a.out
SUCCESS test 1 fortran only fixed format

# Test 2
$ gfortran TEST_2_fortran_only_free.f90
$ ./a.out
Assume Fortran 2003: has FLUSH, ALLOCATABLE, derived type, and ISO C Binding
SUCCESS test 2 fortran only free format

# Test 3
$ gcc TEST_3_c_only.c
$ ./a.out
SUCCESS test 3 c only

# Test 4
$ gcc -c -m64 TEST_4_fortran+c_c.c
$ gfortran -c -m64 TEST_4_fortran+c_f.f90
$ gfortran -m64 TEST_4_fortran+c_f.o TEST_4_fortran+c_c.o
$ ./a.out
C function called by Fortran 
Values are xx = 2.00 and ii = 1
SUCCESS test 4 fortran calling c

# Test 5
$ csh ./TEST_csh.csh
SUCCESS csh test

# Test 6
$ ./TEST_perl.pl
SUCCESS perl test

# Test 7
$ ./TEST_sh.sh
SUCCESS sh test


# BUILDING LIBRARIES
$ cd $HOME/wrfpoc/zen3/Build_WRF
$ mkdir LIBRARIES

$ cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
wget http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/netcdf-4.1.3.tar.gz 
wget http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/jasper-1.900.1.tar.gz 
wget http://prdownloads.sourceforge.net/libpng/libpng-1.6.37.tar.gz?download
wget http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/zlib-1.2.7.tar.gz
wget https://github.com/westes/flex/archive/refs/tags/v2.6.4.tar.gz
wget https://support.hdfgroup.org/ftp/HDF/releases/HDF4.2.13/src/hdf-4.2.13.tar.gz
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.4/src/hdf5-1.10.4.tar.gz
wget https://www.ijg.org/files/jpegsrc.v9d.tar.gz
mv libpng-1.6.37.tar.gz\?download libpng-1.6.37.tar.gz

```
Create a setevn.sh file:
```
$ vi ~/setenv.sh

#!/bin/bash

ulimit -s unlimited

export OPENSSL=openssl
export YACC="yacc -d"
export J="-j 6"

export DIR=$HOME/wrfpoc/zen3/Build_WRF/LIBRARIES

# COMPILERS
export CC=gcc
export FC=gfortran
export SERIAL_FC=gfortran
export SERIAL_F77=gfortran
export SERIAL_CC=gcc
export SERIAL_CXX=g++
export MPI_FC=mpif90
export MPI_F77=mpif77
export MPI_CC=mpicc
export MPI_CXX=mpicxx

export PATH=$DIR/openmpi/bin:$PATH
export PATH=$DIR/netcdf/bin:$PATH
export NETCDF=$DIR/netcdf
export JASPERLIB=$DIR/grib2/lib
export JASPERINC=$DIR/grib2/include
export FLEX=$DIR/flex/bin/flex
export FLEX_LIB_DIR=$DIR/flex/lib
export HDF4=$DIR/hdf4
export HDF5=$DIR/hdf5
export OPENMPI=$DIR/openmpi
export LIBPNG=$DIR/libpng
export LIBPNGLIB=$DIR/libpng
export HDFEOS2=$DIR/hdfeos2
export NCARG=$DIR/ncl
export SZIP=$DIR/szip
export JPEG=$DIR/jpeg
export SQLITE3=$DIR/sqlite336
export GDAL=$DIR/gdal


# run-time linking   ${H5DIR}/lib
export LD_LIBRARY_PATH=${HDF5}/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${HDF4}/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${NETCDF}/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${JASPERLIB}:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${FLEX_LIB_DIR}:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${FLEX_LIB_DIR}:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${OPENMPI}/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${LIBPNG}/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${HDFEOS2}/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${SZIP}/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${SQLITE3}/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=${GDAL}/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/usr/lib64:$LD_LIBRARY_PATH

# LDOPE
export ANCPATH=$HOME/wrfpoc/zen3/Build_WRF/VPRM/ldope_32bit_i386_static_patched/ANCILLARY

# VPRMPreproc
export VPRM=$HOME/wrfpoc/zen3/Build_WRF/VPRM/VPRMpreproc_R99

# WRF
export WRFV3=$HOME/wrfpoc/zen3/Build_WRF/WRFV3
export WRFIO_NCD_LARGE_FILE_SUPPORT=1
export WRF_CHEM=1
export WRF_KPP=1

```

We will compile now the libraries needed.

```
$ source ~/setenv.sh
$ module avail
$ module load gcc-9.2.0
$ module load mpi/openmpi


# zlib
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
tar -zxvf zlib-1.2.7.tar.gz
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES/zlib-1.2.7
./configure --prefix=$DIR/zlib
make
make install
cd ..


# jpeg-9b
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
tar zxvf jpegsrc.v9d.tar.gz
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES/jpeg-9d
./configure --prefix=$DIR/jpeg --disable-dependency-tracking
make
make install
cd ..


# netcdf
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
tar zxvf netcdf-4.1.3.tar.gz
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES/netcdf-4.1.3
export LDFLAGS="-L$DIR/zlib/lib -L$DIR/jpeg/lib"
export CPFLAGS="-I$DIR/zlib/include -I$DIR/jpeg/include"
./configure --prefix=$DIR/netcdf --disable-dap --disable-netcdf-4 --disable-shared  # Check if this should be with shared libraries
make
make install


# hdf4 (Build netcdf first!!! and enable shared-libraries - Is Needed by OBSGRID later)
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
tar zxvf hdf-4.2.13.tar.gz
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES/hdf-4.2.13
# ./configure --prefix=$DIR/hdf4 --with-zlib=$DIR/zlib --enable-fortran --with-jpeg=$DIR/jpeg   # --> Use this with gcc-4.8.5 (DOES NOT WORK WITH OBSGRID)
# ./configure --prefix=$DIR/hdf4 --with-zlib=$DIR/zlib --enable-fortran --with-jpeg=$DIR/jpeg   --with-gnu-ld    # --> Use this with gcc-9.2.0 (DOES NOT WORK WITH OBSGRID)
./configure --prefix=$DIR/hdf4 --with-zlib=$DIR/zlib --disable-fortran --with-jpeg=$DIR/jpeg --enable-shared --with-gnu-ld    # --> Use this with gcc-9.2.0 (THIS ONE WORKS!)
make 
make install
cd ..

# hdf5
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
tar zxvf hdf5-1.10.4.tar.gz
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES/hdf5-1.10.4 
./configure --prefix=$DIR/hdf5 --enable-hl --enable-fortran --enable-unsupported --enable-cxx  --with-zlib=$DIR/zlib --with-pic
make 
make install
cd ..

# libpng
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
tar -zxvf libpng-1.6.37.tar.gz
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES/libpng-1.6.37
export LDFLAGS=-L$DIR/zlib/lib
export CPFLAGS=-I$DIR/zlib/include
./configure --prefix=$DIR/libpng
make
make install
cd ..


#JasPer
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
tar -zxvf jasper-1.900.1.tar.gz
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES/jasper-1.900.1
./configure --prefix=$DIR/jasper
make
make install
cd ..


#Flex
sudo yum install -y help2man
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
tar -zxvf v2.6.4.tar.gz
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES/flex-2.6.4
./autogen.sh
./configure --prefix=$DIR/flex
make
make install
cd ..


# H4toH5 (Just download the precompiled binaries for Linux )
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
wget https://support.hdfgroup.org/ftp/HDF5/releases/tools/h4toh5/h4toh5-2.2.5/bin/h4h5tools-1.10.6-2.2.5-centos7_64.tar.gz
tar -zxvf h4h5tools-1.10.6-2.2.5-centos7_64.tar.gz
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES/hdf
./H4H5-2.2.5-Linux.sh   --> Accept License and location
cd H4H5-2.2.5-Linux/HDF_Group/H4H5/2.2.5/bin
./h4toh5convert -h


# DELETE !!!!! 
./config --prefix=/usr --openssldir=/etc/ssl --libdir=lib no-shared zlib-dynamic
make -j8
make test -j8
sudo make install
export LD_LIBRARY_PATH=/usr/local/lib:/usr/local/lib64:$LD_LIBRARY_PATH


```
OPTIONAL
```
#
# Libraries compatibility tests  (OPTIONAL)
#

$ cd $HOME/wrfpoc/zen3/TESTS
$ wget http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/Fortran_C_NETCDF_MPI_tests.tar
$ tar -xvf Fortran_C_NETCDF_MPI_tests.tar

# Test 1: Fortran + C + NetCDF
$ cp ${NETCDF}/include/netcdf.inc .
$ gfortran -c 01_fortran+c+netcdf_f.f
$ gcc -c 01_fortran+c+netcdf_c.c
$ gfortran 01_fortran+c+netcdf_f.o 01_fortran+c+netcdf_c.o -L${NETCDF}/lib -lnetcdff -lnetcdf
$ ./a.out
C function called by Fortran
Values are xx = 2.00 and ii = 1
SUCCESS test 1 fortran + c + netcdf


# Test 2: Fortran + C + NetCDF + MPI
$ cp ${NETCDF}/include/netcdf.inc .
$ mpif90 -c 02_fortran+c+netcdf+mpi_f.f
$ mpicc -c 02_fortran+c+netcdf+mpi_c.c
$ mpif90 02_fortran+c+netcdf+mpi_f.o 02_fortran+c+netcdf+mpi_c.o -L${NETCDF}/lib -lnetcdff -lnetcdf
$ mpirun -np 2 ./a.out
   C function called by Fortran
   Values are xx =  2.00 and ii = 1
   C function called by Fortran
   Values are xx =  2.00 and ii = 1
 status =            2
 status =            2
 SUCCESS test 2 fortran + c + netcdf + mpi
 SUCCESS test 2 fortran + c + netcdf + mpi
 
 ```
``` 
# BUILDING WRF_V3.9.1  
# Upload Model PAckage.zip to the server and uncompres. Then copy file WRFV3.9.1.tar.gz
$ cd $HOME/wrfpoc/zen3/Build_WRF
$ cp {your_location}/WRFV3.9.1.tar.gz 
$ tar zxvf WRFV3.9.1.tar.gz
$ cd $HOME/wrfpoc/zen3/Build_WRF/WRFV3
$ ./clean -a
$ ./configure

```
Few options are presented 
```
checking for perl5... no
checking for perl... found /bin/perl (perl)
Will use NETCDF in dir: /shared/home/azureuser/wrfpoc/zen3/Build_WRF/LIBRARIES/netcdf
HDF5 not set in environment. Will configure WRF for use without.
PHDF5 not set in environment. Will configure WRF for use without.
Will use 'time' to report timing information


If you REALLY want Grib2 output from WRF, modify the arch/Config_new.pl script.
Right now you are not getting the Jasper lib, from the environment, compiled into WRF.

------------------------------------------------------------------------
Please select from among the following Linux x86_64 options:

  1. (serial)   2. (smpar)   3. (dmpar)   4. (dm+sm)   PGI (pgf90/gcc)
  5. (serial)   6. (smpar)   7. (dmpar)   8. (dm+sm)   PGI (pgf90/pgcc): SGI MPT
  9. (serial)  10. (smpar)  11. (dmpar)  12. (dm+sm)   PGI (pgf90/gcc): PGI accelerator
 13. (serial)  14. (smpar)  15. (dmpar)  16. (dm+sm)   INTEL (ifort/icc)
                                         17. (dm+sm)   INTEL (ifort/icc): Xeon Phi (MIC architecture)
 18. (serial)  19. (smpar)  20. (dmpar)  21. (dm+sm)   INTEL (ifort/icc): Xeon (SNB with AVX mods)
 22. (serial)  23. (smpar)  24. (dmpar)  25. (dm+sm)   INTEL (ifort/icc): SGI MPT
 26. (serial)  27. (smpar)  28. (dmpar)  29. (dm+sm)   INTEL (ifort/icc): IBM POE
 30. (serial)               31. (dmpar)                PATHSCALE (pathf90/pathcc)
 32. (serial)  33. (smpar)  34. (dmpar)  35. (dm+sm)   GNU (gfortran/gcc)
 36. (serial)  37. (smpar)  38. (dmpar)  39. (dm+sm)   IBM (xlf90_r/cc_r)
 40. (serial)  41. (smpar)  42. (dmpar)  43. (dm+sm)   PGI (ftn/gcc): Cray XC CLE
 44. (serial)  45. (smpar)  46. (dmpar)  47. (dm+sm)   CRAY CCE (ftn $(NOOMP)/cc): Cray XE and XC
 48. (serial)  49. (smpar)  50. (dmpar)  51. (dm+sm)   INTEL (ftn/icc): Cray XC
 52. (serial)  53. (smpar)  54. (dmpar)  55. (dm+sm)   PGI (pgf90/pgcc)
 56. (serial)  57. (smpar)  58. (dmpar)  59. (dm+sm)   PGI (pgf90/gcc): -f90=pgf90
 60. (serial)  61. (smpar)  62. (dmpar)  63. (dm+sm)   PGI (pgf90/pgcc): -f90=pgf90
 64. (serial)  65. (smpar)  66. (dmpar)  67. (dm+sm)   INTEL (ifort/icc): HSW/BDW
 68. (serial)  69. (smpar)  70. (dmpar)  71. (dm+sm)   INTEL (ifort/icc): KNL MIC
 72. (serial)  73. (smpar)  74. (dmpar)  75. (dm+sm)   FUJITSU (frtpx/fccpx): FX10/FX100 SPARC64 IXfx/Xlfx

```
Select `34` and nesting = `1. Default`

The configuration will create the `configure.wrf` file. Modify the value of DM_CC as indicated below:
```
DM_CC           =       mpicc -DMPI2_SUPPORT
```

Now we need to decide which type of case you wish to compile. Options are listed below.
```
em_real (3d real case)
em_quarter_ss (3d ideal case)
em_b_wave (3d ideal case)
em_les (3d ideal case)
em_heldsuarez (3d ideal case)
em_tropical_cyclone (3d ideal case)
em_hill2d_x (2d ideal case)
em_squall2d_x (2d ideal case)
em_squall2d_y (2d ideal case)
em_grav2d_x (2d ideal case)
em_seabreeze2d_x (2d ideal case)
em_scm_xy (1d ideal case)
```
For this purpose we are going to compile WRF for real cases. Compilation should take about 20-30 minutes. The ongoing compilation can be checked.
```
$ ./compile em_real >& compile.log &
$ tail -f compile.log
```

Once is completed the output should be like this:
```
==========================================================================
build started:   Thu Jul 22 16:58:45 UTC 2021
build completed: Thu Jul 22 17:32:15 UTC 2021

--->                  Executables successfully built                  <---

-rwxrwxr-x. 1 azureuser azureuser 63292080 Jul 22 17:32 main/ndown.exe
-rwxrwxr-x. 1 azureuser azureuser 63148792 Jul 22 17:32 main/real.exe
-rwxrwxr-x. 1 azureuser azureuser 62438144 Jul 22 17:32 main/tc.exe
-rwxrwxr-x. 1 azureuser azureuser 79232976 Jul 22 17:29 main/wrf.exe

==========================================================================

```

### 2. Pre-Processing Tools ####
#### 2.1 WPS #####

```
$ cd $HOME/wrfpoc/zen3/Build_WRF/WPS
$ ./configure
 1.  Linux x86_64, gfortran    (serial)
 2.  Linux x86_64, gfortran    (serial_NO_GRIB2)
 3.  Linux x86_64, gfortran    (dmpar)
 4.  Linux x86_64, gfortran    (dmpar_NO_GRIB2)
 5.  Linux x86_64, PGI compiler   (serial)
 6.  Linux x86_64, PGI compiler   (serial_NO_GRIB2)
 
 Select Option 3

$ vi configure.wps
```
Replace the file `configure.wps` with the following values:

```
DM_FC               = mpif90
DM_CC               = mpicc
CPP                 = cpp -P -traditional
```
Now compile:
```
$ ./compile 
$ find ./ -name "*.exe"
./util/src/rd_intermediate.exe
./util/src/mod_levs.exe
./util/src/int2nc.exe
./util/src/height_ukmo.exe
./util/src/calc_ecmwf_p.exe
./util/src/avg_tsfc.exe
./util/rd_intermediate.exe
./util/mod_levs.exe
./util/int2nc.exe
./util/height_ukmo.exe
./util/g1print.exe
./util/calc_ecmwf_p.exe
./util/avg_tsfc.exe
./ungrib/src/g1print.exe
./ungrib/g1print.exe
```

#### 2.2 OBSGRID #####

It requires cairo-devel libaries and  [NCL libraries ](https://www.ncl.ucar.edu/Download/build_from_src.shtml). 

NCL Libraries requires also JPEG, ZLIB,Cairo, NetCDF, HDF-4 plus some optional external packages. We will use conda for the installation of the libraries.

To install conda we will follow the instructions in [NCL]https://www.ncl.ucar.edu/Download/conda.shtml:


```
$ sudo yum install cairo-devel -y

$ cd /tmp
$ curl -O https://repo.anaconda.com/archive/Anaconda3-5.3.1-Linux-x86_64.sh
$ bash Anaconda3-5.3.1-Linux-x86_64.sh
$ source ~/.bashrc

$ source ~/anaconda3/bin/activate
$ conda create -n ncl_stable -c conda-forge ncl
$ conda activate ncl_stable

$ cd $HOME/wrfpoc/zen3/Build_WRF/OBSGRID
$ export FCFLAGS="-w -Wno-argument-mismatch -O2"
$ export FFLAGS="-w -Wno-argument-mismatch -O2"
```
Run the configuration to generate the configure.oa and we will amend the file before compiling. 
```
$ ./configure
```
Edit the configure.oa file and replace the following lines:
```
NETCDF_LIBS     =       -L${NETCDF}/lib -lnetcdf -lnetcdff
NETCDF_INC      =       -I${NETCDF}/include
NCARG_LIBS      =       -L${NCARG_ROOT}/lib -lncarg -lncarg_gks -lncarg_c -lX11 -lm -lcairo -lfreetype

FC              =       gfortran
FFLAGS          =       -ffree-form -O -fconvert=big-endian -frecord-marker=4
F77FLAGS        =       -ffixed-form -O -fconvert=big-endian -frecord-marker=4
FNGFLAGS        =       $(FFLAGS)
LDFLAGS         =
CC              =       gcc
CFLAGS          =
CPP             =       cpp -P -traditional
CPPFLAGS        =
```

Now compile:
```
$ ./compile
```
The exe files should have been generated now.
```
$ ls -l *.exe
lrwxrwxrwx 1 azureuser azureuser 15 Sep  6 15:43 obsgrid.exe -> src/obsgrid.exe
lrwxrwxrwx 1 azureuser azureuser 18 Sep  6 15:43 plot_level.exe -> src/plot_level.exe
lrwxrwxrwx 1 azureuser azureuser 22 Sep  6 15:43 plot_soundings.exe -> src/plot_soundings.exe
```

#### 2.3 VPRMpreproc_R99 #####

WRF-VRPM Preproc_R99 requires multiple pacakges:
- HDF5 (Installed Above)
- H4toH5 
- MODIS LDOPE Tool (provided in tar file)
- MODIS MRT (provided in tar file)
- NETCDF  (Installed Above)
- R  (sudo yum install R --> Takes a couple of minutes. Create Custom Image??)
- Rmap package for R [JGRI/rmap](https://github.com/JGCRI/rmap)
- HDF Package for R
- NETCDF package for R

```

# R, Rmap, HFD and NETCDF for R packages:

$ yum install libgit2-devel -y 
$ source ~/anaconda3/bin/activate
$ conda install -c r r-devtools    --> DELETE THIS LINE

# Dependencies needed by some R packages
$ sudo yum install libcurl-devel.x86_64 -y
$ sudo yum install ImageMagick-c++-devel.x86_64 -y 
$ yum install -y geos-devel.x86_64
$ yum install -y gdal-devel.x86_64
$ sudo yum install proj-devel.x86_64 -y
$ sudo yum install netcdf-devel.x86_64 -y


$ sudo yum -y install R 
$ export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/share/pkgconfig/
$ export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/lib/x86_64-linux-gnu/pkgconfig/
$ export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/lib/pkgconfig/
$ sudo -i R

> remotes::install_github("gaborcsardi/parsedate@f2da982")
> install.packages("rematch2")
> install.packages("testthat")
> install.packages("devtools")
> install.packages('rgdal', type = "source", configure.args=c('--with-proj-include=/usr/local/include','--with-proj-lib=/usr/local/lib'))
> devtools::install_github("JGCRI/rgcam")
> devtools::install_github("JGCRI/rmap")

# HFD for R
> install.packages("hdf5r")   --> Preferred
> install.packages("RNetCDF")  
> q()

```
##### 2.3.1 MODIS LDOPE Tool

```
# MODIS LDOPE Tool (provided in tar file)

# Pre-Requisites for LDOPE:
# 1. hdf4  (installed above)

# 2. HDF-EOS2
cd $HOME/wrfpoc/zen3/Build_WRF/VPRM
curl -o HDF-EOS2.20v1.00.tar.Z https://git.earthdata.nasa.gov/rest/git-lfs/storage/DAS/hdfeos/cb0f900d2732ab01e51284d6c9e90d0e852d61bba9bce3b43af0430ab5414903?response-content-disposition=attachment%3B%20filename%3D%22HDF-EOS2.20v1.00.tar.Z%22%3B%20filename*%3Dutf-8%27%27HDF-EOS2.20v1.00.tar.Z
tar zxvf HDF-EOS2.20v1.00.tar.Z
cd hdfeos
./configure --prefix=$DIR/hdfeos2 --with-hdf4=$HDF4 --enable-fortran --enable-install-include
make install

# 3. szip
cd $HOME/wrfpoc/zen3/Build_WRF/VPRM
wget https://support.hdfgroup.org/ftp/lib-external/szip/2.1.1/src/szip-2.1.1.tar.gz
tar zxvf szip-2.1.1.tar.gz
cd szip-2.1.1
./configure --prefix=$DIR/szip
make install


# Gctp
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
wget https://cpan.metacpan.org/authors/id/D/DS/DSTAHLKE/Cartography-Projection-GCTP-0.03.tar.gz
tar zxvf Cartography-Projection-GCTP-0.03.tar.gz
cd Cartography-Projection-GCTP-0.03
perl Makefile.PL
make
sudo make install

Installing /usr/local/share/man/man3/Cartography::Projection::GCTP.3pm
Appending installation info to /usr/lib64/perl5/perllocal.pod


# SQLITE (Required by PROJ 7, sqlite3 >= 3.11' but version of SQLite is 3.7.17)
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
wget https://www.sqlite.org/2021/sqlite-autoconf-3360000.tar.gz
tar zxvf sqlite-autoconf-3360000.tar.gz
cd sqlite-autoconf-3360000
./configure --prefix=$DIR/sqlite336
make
make install

# PROJ 7 (Required by GAL)
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
wget https://download.osgeo.org/proj/proj-6.3.2.tar.gz
tar zxvf proj-6.3.2.tar.gz
cd proj-6.3.2/
./configure --prefix=$DIR/proj SQLITE3_CFLAGS="-I$SQLITE3/include" SQLITE3_LIBS="-L$SQLITE3/lib -lsqlite3"
make
make install
 
# GDAL
cd $HOME/wrfpoc/zen3/Build_WRF/LIBRARIES
wget https://github.com/OSGeo/gdal/releases/download/v3.3.2/gdal-3.3.2.tar.gz
tar zxvf gdal-3.3.2.tar.gz
cd gdal-3.3.2
./configure --prefix=$DIR/gdal 
```

Finally LDOPE TOOL
```

# LDOPE TOOL
 sudo yum install -y multilib-rpm-config.noarch  --> Not really needed
cd $HOME/wrfpoc/zen3/Build_WRF/VPRM
wget https://www.bgc-jena.mpg.de/bgc-systems/uploads/Download/VPRMpreproc/ldope_32bit_i386_static_patched.tar.bz2
bunzip2 ldope_32bit_i386_static_patched.tar.bz2
tar xvf ldope_32bit_i386_static_patched.tar
cd ldope_32bit_i386_static_patched

```
Edit the file `/shared/home/azureuser/wrfpoc/zen3/Build_WRF/LIBRARIES/hdfeos2/include/gctp_prototypes.h` and rename the following functions:
```
int sinfor(double lon, double lat, double *x, double *y);
int sinforint(double r_maj, double r_min, double center_long,
              double false_east, double false_north);
int isinusfor(double lon, double lat, double *x, double *y);
int isinusforinit(double sphere, double lon_cen_mer, double false_east,
                   double false_north, double dzone, double djustify);
```
by the following: 
```
int sinfor_tmp(double lon, double lat, double *x, double *y);
int sinforint_tmp(double r_maj, double r_min, double center_long,
              double false_east, double false_north);
int isinusfor_tmp(double lon, double lat, double *x, double *y);
int isinusforinit_tmp(double sphere, double lon_cen_mer, double false_east,
                   double false_north, double dzone, double djustify);

```
Now make the following changes in ./src/Makefile
```
#LOC = /Net/Groups

HDF = $(DIR)/hdf4

BINDIR = ../bin
OBJDIR = ../obj

EXTRA = -O
CC1     = $(CC)

EOSINC = $(DIR)/hdfeos2/include
EOSLIBDIR =  $(DIR)/hdfeos2/lib

INCDIR  = -I$(HDF)/include
EOSINCDIR = -I$(EOSINC)
NCFLAGS = $(EXTRA) $(INCDIR) $(EOSINCDIR)

LIB   = -L/opt/gcc-9.2.0/lib64 -L/usr/lib64 -L/usr/local/lib -L$(HDF)/lib -L$(EOSLIBDIR) -L$(DIR/szip)/lib -L$(DIR/jpeg)/lib -lmfhdf -ljpeg -lGctp -lhdfeos -ldf -lz -lsz
EOSLIB = -L$(EOSLIBDIR) -lhdfeos -lGctp

```
Now run `./LDOPE_Linux_install.sh` and provide the path for HDF4: in this case `/shared/home/azureuser/wrfpoc/zen3/Build_WRF/LIBRARIES/hdf4`


#### 2.3.2 MODIS MRT (provided in tar file)

```
# MODIS MRT (provided in tar file)
cd $HOME/wrfpoc/zen3/Build_WRF/VPRM/MRT_32bit_i386_static_patched
sh install

1. The unzip executable and the MRTLinux.zip installation zip file
   must be present in the current directory.
2. You must know the directory path where the MRT is to be installed. --> /shared/home/azureuser/wrfpoc/zen3/Build_WRF/VPRM/MRT
3. You must know the path to the Java bin directory on your system. --> /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.302.b08-0.el7_9.x86_64/jre/bin

At the end:
The .profile and .bash_profile files have the following lines appended:

    PATH=$PATH:/shared/home/azureuser/wrfpoc/zen3/Build_WRF/VPRM/MRT/bin
    MRTDATADIR=/shared/home/azureuser/wrfpoc/zen3/Build_WRF/VPRM/MRT/data
    export MRTDATADIR


# VPRRM PreProc 
$ cd $HOME/wrfpoc/zen3/Build_WRF/VPRM
wget https://www.bgc-jena.mpg.de/bgc-systems/uploads/Download/VPRMpreproc/VPRMpreproc_LCC_R99.tar.bz2
cd VPRMpreproc_R99
vi config.r --> Replace the PATH to point to the right location 
cp ./RSources/gridEurope.r.BAK ./RSources/gridEurope.r 
vi ./RSources/gridEurope.r 
./get_synmap.sh
./compile.sh


```

#### 2.4 MOZBC #####
```
$ cd $HOME/wrfpoc/zen3/Build_WRF/mozbc
$ export NETCDF_DIR=$NETCDF
$ cp ${NETCDF}/include/netcdf.inc .
```
Edit the Makefile with the right $NETCDF
```
LIBS   = -L${NETCDF}/lib $(AR_FILES)
INCLUDE_MODULES = -I${NETCDF}/include

```

Now compile
```
$ ./make_mozbc
```

 



