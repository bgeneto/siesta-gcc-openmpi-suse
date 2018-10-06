## 0. Purpose 

This document contains step-by-step instructions to proceed with a (hopefully) successful installation of the SIESTA (Spanish Initiative for Electronic Simulations with Thousands of Atoms) software on Linux (tested with OpenSUSE Tumbleweed) using the GCC and OpenMPI tools for parallelism. 

To achieve a parallel build of SIESTA you should ﬁrst determine which type of parallelism you need. It is advised to use MPI for calculations with a moderate number of cores. For hundreds of threads, hybrid parallelism using both MPI and OpenMP may be required.

## 1. Install prerequisite software

*Note: We assume you are running all the commands below as an ordinary (non-root) user, so we use `sudo` when required. 

```
sudo zypper install gcc-c++ gcc-fortran openmpi openmpi-devel readline-devel libxslt-tools
```

## 2. Create required installation folders

```
SIESTA_DIR=/opt/siesta
OPENBLAS_DIR=/opt/openblas
SCALAPACK_DIR=/opt/scalapack

sudo mkdir $SIESTA_DIR $OPENBLAS_DIR $SCALAPACK_DIR
# temporally loose permissions
sudo chmod -R 777 $SIESTA_DIR $OPENBLAS_DIR $SCALAPACK_DIR
```

## 3. Setup OpenMPI 

Unfortunetely, for SUSE, the openmpi installation does not work out of the box, you need to set PATH and LD_LIBRARY_PATH env vars. We recommend you add these lines to a file in your profile.d directory, say `/etc/profile.d/openmpi.sh` so that variables are set correctly for all sessions (see [Section 8](https://github.com/bgeneto/siesta-gcc-openmpi-suse/blob/master/README.md#8-make-siesta-available-for-all-users)). 

```
openmpipath=/usr/lib64/mpi/gcc/openmpi
export PATH=$PATH:$openmpipath/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$openmpipath/lib64:$OPENBLAS_DIR/lib
export CPATH=$CPATH:$openmpipath/include
```

You can check your current openmpi path on SUSE with the command below: 

```
echo $(mpicxx -showme:compile | awk '{ print $1}' | sed 's/-I//; s/\/include//')
```


## 3. Install prerequisite libraries 

In order to run siesta in parallel using MPI you need non-threaded blas and lapack libraries along with a standard scalapack library.

#### 3.1. Install single-threaded openblas library from source

*Note: apt installs a threaded version of openblas by default, I think this is not suitable for this MPI build of siesta.*

```
cd $OPENBLAS_DIR
wget -O OpenBLAS.tar.gz https://ufpr.dl.sourceforge.net/project/openblas/v0.3.3/OpenBLAS%200.3.3%20version.tar.gz
tar xzf OpenBLAS.tar.gz && rm OpenBLAS.tar.gz
cd "$(find . -name xianyi-OpenBLAS*)"
make DYNAMIC_ARCH=0 CC=gcc FC=gfortran HOSTCC=gcc BINARY=64 INTERFACE=64 \
  NO_AFFINITY=1 NO_WARMUP=1 USE_OPENMP=0 USE_THREAD=0 LIBNAMESUFFIX=nonthreaded
make PREFIX=$OPENBLAS_DIR LIBNAMESUFFIX=nonthreaded install
```

#### 3.2. Install scalapack from source

```
cd $SCALAPACK_DIR
wget http://www.netlib.org/scalapack/scalapack_installer.tgz
tar xzf ./scalapack_installer.tgz && cd ./scalapack_installer
./setup.py --prefix $SCALAPACK_DIR --blaslib=$OPENBLAS_DIR/lib/libopenblas_nonthreaded.a \
  --lapacklib=$OPENBLAS_DIR/lib/libopenblas_nonthreaded.a --mpibindir=$openmpipath/bin \
  --mpiincdir=$openmpipath/include
```

*Note: Answer 'b' if asked: 'Which BLAS library do you want to use ?'*

## 4. Install siesta from source

```
cd $SIESTA_DIR
wget https://launchpad.net/siesta/4.1/4.1-b3/+download/siesta-4.1-b3.tar.gz
tar xzf ./siesta-4.1-b3.tar.gz && rm ./siesta-4.1-b3.tar.gz
```

#### 4.1. Install siesta library dependencies from source

```
cd $SIESTA_DIR/siesta-4.1-b3/Docs
wget https://github.com/ElectronicStructureLibrary/flook/releases/download/v0.7.0/flook-0.7.0.tar.gz
(./install_flook.bash 2>&1) | tee install_flook.log
```

Install netcdf dependency (be patient):

```
cd $SIESTA_DIR/siesta-4.1-b3/Docs
wget https://zlib.net/zlib-1.2.11.tar.gz
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.8/hdf5-1.8.18/src/hdf5-1.8.18.tar.bz2
wget -O netcdf-c-4.4.1.1.tar.gz https://github.com/Unidata/netcdf-c/archive/v4.4.1.1.tar.gz
wget -O netcdf-fortran-4.4.4.tar.gz https://github.com/Unidata/netcdf-fortran/archive/v4.4.4.tar.gz
(./install_netcdf4.bash 2>&1) | tee install_netcdf4.log
cd $SIESTA_DIR/siesta-4.1-b3/Docs/build/netcdf/4.4.1.1 && ln -s lib64 lib
cd $SIESTA_DIR/siesta-4.1-b3/Docs/build/hdf5/1.8.18 && ln -s lib64 lib
```

If anything goes wrong in this step you can check the `install_netcdf4.log` log file.

#### 4.2. Download our custom 'arch.make' file for GCC + OpenMPI build 

```
cd $SIESTA_DIR/siesta-4.1-b3/Obj
wget -O arch.make https://raw.githubusercontent.com/bgeneto/siesta-gcc-openmpi/master/gnu-openmpi-arch.make
```

#### 4.3. Build siesta executable 

```
cd $SIESTA_DIR/siesta-4.1-b3/Obj
sh ../Src/obj_setup.sh
sudo ln -s $SIESTA_DIR/siesta-4.1-b3/Docs/build/netcdf/4.4.1.1/lib64 $SIESTA_DIR/siesta-4.1-b3/Docs/build/netcdf/4.4.1.1/lib
make OBJDIR=Obj
```

## 5. Revert permissions

```
sudo chown -R root:root $SIESTA_DIR $OPENBLAS_DIR $SCALAPACK_DIR
sudo chmod -R 755 $SIESTA_DIR $OPENBLAS_DIR $SCALAPACK_DIR
```

## 6. Test siesta

Let's copy siesta `Test` directory to our home (where we have all necessary permissions).

```
SIESTA_DIR=/opt/siesta
mkdir $HOME/siesta
rsync -a $SIESTA_DIR/siesta-4.1-b3/Tests/ $HOME/siesta/Tests/
```

Now create a symbolic link to siesta executable 

```
cd $HOME/siesta
ln -s $SIESTA_DIR/siesta-4.1-b3/Obj/siesta
```

Finally run some test job:

```
cd $HOME/siesta/Tests/h2o_dos/
make
```

We should see the following message at the end:

```
===> SIESTA finished successfully
```

## 8. Make siesta available for all users

If you want to make siesta available to all users you can move the required directories to another location, say `/opt`, and set the required environmental variables system wide:

```
echo "export PATH=\$PATH:$openmpi/bin" | sudo tee --append /etc/profile.d/openmpi.sh > /dev/null
echo "export export LD_LIBRARY_PATH=\$LD_LIBRARY_PATH:$openmpipath/lib64:$OPENBLAS_DIR/lib" | sudo tee --append /etc/profile.d/openmpi.sh > /dev/null
echo "export CPATH=\$CPATH:$openmpipath/include" | sudo tee --append /etc/profile.d/openmpi.sh > /dev/null
```

## 9. Learning to use siesta

Read the [manual](https://siesta-project.github.io/bsc2017/siesta-4.1.pdf).
