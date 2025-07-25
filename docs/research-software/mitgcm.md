# MITgcm

The Massachusetts Institute of Technology General Circulation Model
(MITgcm) is a numerical model designed for study of the atmosphere,
ocean, and climate. MITgcm's flexible non-hydrostatic formulation
enables it to simulate fluid phenomena over a wide range of scales; its
adjoint capabilities enable it to be applied to sensitivity questions
and to parameter and state estimation problems. By employing fluid
equation isomorphisms, a single dynamical kernel can be used to simulate
flow of both the atmosphere and ocean.

## Useful Links

  - [MITgcm home page](http://mitgcm.org)
  - [MITgcm documentation](https://mitgcm.readthedocs.io/en/latest/)

## Building MITgcm on ARCHER2

MITgcm is not available via a module on ARCHER2 as users will build
their own executables specific to the problem they are working on.
However, we do provide an optfile which will allow `genmake2` to create
Makefiles which will work on ARCHER2.

!!! note
    The processes to build MITgcm on the ARCHER2 4-cabinet system and full
    system are slightly different. Please make sure you use the commands
    for the correct system below.

You can obtain the MITgcm source code from the developers by cloning
from the GitHub repository with the command

    git clone https://github.com/MITgcm/MITgcm.git

You should then copy the ARCHER2 optfile into the MITgcm directories. You may use the files at the locations below for the 4-cabinet and full systems.

=== "Full system"
    ```bash
    cp /work/n02/shared/mjmn02/ECCOv4/cases/cce/cce1/scripts/dev_linux_amd64_cray_archer2 MITgcm/tools/build_options/
    ```
  
You should also set the following environment variables.
`MITGCM_ROOTDIR` is used to locate the source code and should point to
the top MITgcm directory. Optionally, adding the MITgcm tools directory
to your `PATH` environment variable makes it easier to use tools such as
`genmake2`, and the `MITGCM_OPT` environment variable makes it easier to
refer to pass the optfile to `genmake2`.

=== "Full system"
    ```
    export MITGCM_ROOTDIR=/path/to/MITgcm
    export PATH=$MITGCM_ROOTDIR/tools:$PATH
    export MITGCM_OPT=$MITGCM_ROOTDIR/tools/build_options/dev_linux_amd64_cray_archer2
    ```

When using `genmake2` to create the Makefile, you will need to specify the
optfile to use. Other commonly used options might be to use extra source
code with the `-mods` option, to enable MPI with `-mpi`, and to enable OpenMP with `-omp`. You might
then run a command that resembles the following:

    genmake2 -mods /path/to/additional/source -mpi -optfile $MITGCM_OPT

You can read about the full set of options available to `genmake2` by
running

    genmake2 -help

Finally, you may then build your executable by running 

    make depend
    make

## Running MITgcm on ARCHER2

### Pure MPI
Once you have built your executable you can write a script like the
following which will allow it to run on the ARCHER2 compute nodes. This
example would run a pure MPI MITgcm simulation over 2 nodes of 128 cores
each for up to one hour.

=== "Full system"
    ```
    #!/bin/bash

    # Slurm job options (job-name, compute nodes, job time)
    #SBATCH --job-name=MITgcm-simulation
    #SBATCH --time=1:0:0
    #SBATCH --nodes=2
    #SBATCH --ntasks-per-node=128
    #SBATCH --cpus-per-task=1

    # Replace [budget code] below with your project code (e.g. t01)
    #SBATCH --account=[budget code] 
    #SBATCH --partition=standard
    #SBATCH --qos=standard

    # Set the number of threads to 1
    #   This prevents any threaded system libraries from automatically
    #   using threading.
    export OMP_NUM_THREADS=1

    # Launch the parallel job
    #   Using 256 MPI processes and 128 MPI processes per node
    #   srun picks up the distribution from the sbatch options
    srun --distribution=block:block --hint=nomultithread ./mitgcmuv
    
    ```

### Hybrid OpenMP & MPI
!!! warning
    Running the model in hybrid mode may lead to performance decreases as well
    as increases. You should be sure to profile your code both as a pure MPI
    application and as a hybrid OpenMP-MPI application to ensure you are
    making efficient use of resources. Be sure to read both the Archer2
    [advice on OpenMP](../user-guide/tuning.md#hybrid-mpi-and-openmp)  and the
    [MITgcm documentation](https://mitgcm.readthedocs.io/en/latest/getting_started/getting_started.html#building-with-openmp)
    first.

!!! note
    Early versions of the ARCHER2 MITgcm optfile do not contain an `OMPFLAG`.
    Please ensure you have an up to date copy of the optfile before attempting
    to compile OpenMP enabled codes.

Depending upon your model setup, you may wish to run the MITgcm code as a
hybrid OpenMP-MPI application. In terms of compiling the model, this is as
simple as using the flag `-omp` when calling `genmake2`, and updating your
`SIZE.h` file to have multiple tiles per process.

The model can be run using a slurm job submission script similar to that shown
below. This example will run MITgcm across 2 nodes, with each node using 16 MPI
processes, and each process using 4 threads. Note that this would underpopulate
the nodes — i.e. we will only be using 128 of the 256 cores available to us.
This can also sometimes lead to performance increases.

=== "Full system"
    ```
    #!/bin/bash

    # Slurm job options (job-name, compute nodes, job time)
    #SBATCH --job-name=MITgcm-hybrid-simulation
    #SBATCH --time=1:0:0
    #SBATCH --nodes=2
    #SBATCH --ntasks-per-node=16
    #SBATCH --cpus-per-task=4

    # Replace [budget code] below with your project code (e.g. t01)
    #SBATCH --account=[budget code] 
    #SBATCH --partition=standard
    #SBATCH --qos=standard

    # Set the number of threads to 1
    #   This prevents any threaded system libraries from automatically
    #   using threading.
    export OMP_NUM_THREADS=4  # Set to number of threads per process
    export OMP_PLACES="cores(128)"  # Set to total number of threads
    export OMP_PROC_BIND=true  # Required if we want to underpopulate nodes

    # Launch the parallel job
    #   Using 256 MPI processes and 128 MPI processes per node
    #   srun picks up the distribution from the sbatch options
    srun --distribution=block:block --hint=nomultithread ./mitgcmuv

    ```

One final note, is that you should remember to update the `eedata` file in the
model's run directory to ensure the number of threads requested there match
those requested in the job submission script.

## Reproducing the ECCO version 4 (release 4) state estimate on ARCHER2

The ECCO version 4 state estimate (ECCOv4-r4) is an observationally-constrained numerical solution produced by the ECCO group at JPL. If you would like to reproduce the state estimate on ARCHER2 in order to create customised runs and experiments, follow the instructions below. They have been slightly modified from the JPL instructions for ARCHER2. 

For more information, see the ECCOv4-r4 website <https://ecco-group.org/products-ECCO-V4r4.htm>

### Get the ECCOv4-r4 source code

First, navigate to your directory on the ``/work`` filesystem in order to get access to the compute nodes. Next, create a working directory, perhaps MYECCO, and navigate into this working directory:

    mkdir MYECCO
    cd MYECCO
    
In order to reproduce ECCOv4-r4, we need a specific checkpoint of the MITgcm source code. 

    git clone https://github.com/MITgcm/MITgcm.git -b checkpoint66g
    
Next, get the ECCOv4-r4 specific code from GitHub:

    cd MITgcm
    mkdir -p ECCOV4/release4
    cd ECCOV4/release4
    git clone https://github.com/ECCO-GROUP/ECCO-v4-Configurations.git
    mv ECCO-v4-Configurations/ECCOv4\ Release\ 4/code .
    rm -rf ECCO-v4-Configurations
    
### Get the ECCOv4-r4 forcing files

The surface forcing and other input files that are too large to be stored on GitHub are available via NASA data servers. In total, these files are about 200 GB in size. You must register for an Earthdata account and connect to a WebDAV server in order to access these files. For more detailed instructions, read the help page <https://ecco.jpl.nasa.gov/drive/help>.

First, apply for an Earthdata account: <https://urs.earthdata.nasa.gov/users/new>

Next, acquire your WebDAV credentials: <https://ecco.jpl.nasa.gov/drive> (second box from the top)

Now, you can use wget to download the required forcing and input files:

    wget -r --no-parent --user YOURUSERNAME --ask-password https://ecco.jpl.nasa.gov/drive/files/Version4/Release4/input_forcing
    wget -r --no-parent --user YOURUSERNAME --ask-password https://ecco.jpl.nasa.gov/drive/files/Version4/Release4/input_init 
    wget -r --no-parent --user YOURUSERNAME --ask-password https://ecco.jpl.nasa.gov/drive/files/Version4/Release4/input_ecco
      
After using `wget`, you will notice that the `input*` directories are, by default, several levels deep in the directory structure. Use the `mv` command to move the `input*` directories to the directory where you executed the `wget` command. Specifically,

```
mv ecco.jpl.nasa.gov/drive/files/Version4/Release4/input_forcing/ .
mv ecco.jpl.nasa.gov/drive/files/Version4/Release4/input_init/ .
mv ecco.jpl.nasa.gov/drive/files/Version4/Release4/input_ecco/ .
rm -rf ecco.jpl.nasa.gov
```

### Compiling and running ECCOv4-r4

The steps for building the ECCOv4-r4 instance of MITgcm are very similar to those for other build cases. First, wou will need to create a build directory:

    cd MITgcm/ECCOV4/release4
    mkdir build
    cd build

Load the NetCDF modules:

    module load cray-hdf5
    module load cray-netcdf

If you haven't already, set your environment variables:

    export MITGCM_ROOTDIR=../../../../MITgcm
    export PATH=$MITGCM_ROOTDIR/tools:$PATH
    export MITGCM_OPT=$MITGCM_ROOTDIR/tools/build_options/dev_linux_amd64_cray_archer2
    
Next, compile the executable:

    genmake2 -mods ../code -mpi -optfile $MITGCM_OPT
    make depend
    make
    
Once you have compiled the model, you will have the mitgcmuv executable for ECCOv4-r4. 

#### Create run directory and link files

In order to run the model, you need to create a run directory and link/copy the appropriate files. First, navigate to your directory on the ``work`` filesystem. From the ``MITgcm/ECCOV4/release4`` directory:

    mkdir run
    cd run
    
    # link the data files
    ln -s ../input_init/NAMELIST/* .
    ln -s ../input_init/error_weight/ctrl_weight/* .
    ln -s ../input_init/error_weight/data_error/* .
    ln -s ../input_init/* .
    ln -s ../input_init/tools/* .
    ln -s ../input_ecco/*/* .
    ln -s ../input_forcing/eccov4r4* .

    python mkdir_subdir_diags.py
    
    # manually copy the mitgcmuv executable
    cp -p ../build/mitgcmuv .

For a short test run, edit the ``nTimeSteps`` variable in the file ``data``. Comment out the default value and uncomment the line reading ``nTimeSteps=8``. This is a useful test to make sure that the model can at least start up. 

To run on ARCHER2, submit a batch script to the Slurm scheduler. Here is an example submission script:

    #!/bin/bash
    
    # Slurm job options (job-name, compute nodes, job time)
    #SBATCH --job-name=ECCOv4r4-test
    #SBATCH --time=1:0:0
    #SBATCH --nodes=8
    #SBATCH --ntasks-per-node=12
    #SBATCH --cpus-per-task=1
    
    # Replace [budget code] below with your project code (e.g. t01)
    #SBATCH --account=[budget code] 
    #SBATCH --partition=standard
    #SBATCH --qos=standard
    
    # Set the number of threads to 1
    #   This prevents any threaded system libraries from automatically
    #   using threading.
    export OMP_NUM_THREADS=1
    
    # Launch the parallel job
    #   Using 256 MPI processes and 128 MPI processes per node
    #   srun picks up the distribution from the sbatch options
    srun --distribution=block:block --hint=nomultithread ./mitgcmuv

This configuration uses 96 MPI processes at 12 MPI processes per node. Once the run has finished, in order to check that the run has successfully completed, check the end of one of the standard output files. 

    tail STDOUT.0000
    
It should read 

    PROGRAM MAIN: Execution ended Normally
    
The files named `STDOUT.*` contain diagnostic information that you can use to check your results. As a first pass, check the printed statistics for any clear signs of trouble (e.g. NaN values, extremely large values). 

#### ECCOv4-r4 in adjoint mode

If you have access to the commercial TAF software produced by <http://FastOpt.de>, then you can compile and run the ECCOv4-r4 instance of MITgcm in adjoint mode. This mode is useful for comprehensive sensitivity studies and for constructing state estimates. From the ``MITgcm/ECCOV4/release4`` directory, create a new code directory and a new build directory:

    mkdir code_ad
    cd code_ad
    ln -s ../code/* .
    cd ..
    mkdir build_ad
    cd build_ad
    
In this instance, the ``code_ad`` and ``code`` directories are identical, although this does not have to be the case. Make sure that you have the ``staf`` script in your path or in the ``build_ad`` directory itself. To make sure that you have the most up-to-date script, run:

    ./staf -get staf
    
To test your connection to the FastOpt servers, try:

    ./staf -test
    
You should receive the following message:

    Your access to the TAF server is enabled.
    
The compilation commands are similar to those used to build the forward case.

    # load relevant modules
    module load cray-netcdf-hdf5parallel
    module load cray-hdf5-parallel

    # compile adjoint model
    ../../../MITgcm/tools/genmake2 -ieee -mpi -mods=../code_ad -of=(PATH_TO_OPTFILE)
    make depend
    make adtaf
    make adall
    
The source code will be packaged and forwarded to the FastOpt servers, where it will undergo source-to-source translation via the TAF algorithmic differentiation software. If the compilation is successful, you will have an executable named ``mitgcmuv_ad``. This will run the ECCOv4-r4 configuration of MITgcm in adjoint mode. As before, create a run directory and copy in the relevant files. The procedure is the same as for the forward model, with the following modifications:

    cd ..
    mkdir run_ad
    cd run_ad
    # manually copy the mitgcmuv executable
    cp -p ../build_ad/mitgcmuv_ad .
    
To run the model, change the name of the executable in the Slurm submission script; everything else should be the same as in the forward case. As above, at the end of the run you should have a set of `STDOUT.*` files that you can examine for any obvious problems. 


##### Compile time errors

If TAF compilation fails with an error like `failed to convert GOTPCREL relocation;
relink with --no-relax` then add the following line to the FFLAGS options: `-Wl,--no-relax`.

##### Checkpointing for adjoint runs

In an adjoint run, there is a balance between storage (i.e. saving the model state to disk) and recomputation (i.e. integrating the model forward from a stored state). Changing the `nchklev` parameters in the `tamc.h` file at compile time is how you control the relative balance between storage and recomputation. 

A suggested strategy that has been used on a variety of HPC platforms is as follows:
1. Set `nchklev_1` as large as possible, up to the size allowed by memory on your machine. (Use the `size` command to estimate the memory per process. This should be just a little bit less than the maximum allowed on the machine. On ARCHER2 this is 2 GB (standard) and 4 GB (high memory)).
2. Next, set `nchklev_2` and `nchklev_3` to be large enough to accomodate the entire run. A common strategy is to set `nchklev_2 = nchklev_3 = sqrt(numsteps/nchklev_1) + 1`. 
3. If the `nchklev_2` files get too big, then you may have to add a fourth level (i.e. `nchklev_4`), but this is unlikely. 

This strategy allows you to keep as much in memory as possible, minimising the I/O requirements for the disk. This is useful, as I/O is often the bottleneck for MITgcm runs on HPC. 

Another way to adjust performance is to adjust how tapelevel I/O is handled. This strategy performs well for most configurations:
```
C o tape settings
#define ALLOW_AUTODIFF_WHTAPEIO
#define AUTODIFF_USE_OLDSTORE_2D
#define AUTODIFF_USE_OLDSTORE_3D
#define EXCLUDE_WHIO_GLOBUFF_2D
#define ALLOW_INIT_WHTAPEIO
```
