# nzesm_apptainer
Definition files for creating an NZESM Apptainer environment

## Introduction

Containerisation allows you to compile code that can be easily ported from one platform to another (with some restrictions). As the name suggests, all the dependencies are contained -- there are no undefined references.

The container `nzesmenv` described below comes with an operating system, compilers, and libraries, including MPI. It can be thought as a replacement for the 
modules used on many high performance computers to load in dependencies (toolchain, NetCDF, etc). 

## How to build a container

First you need to have Apptainer installed. If you're running Linux, you're in luck, just follow these [instructions](https://apptainer.org/docs/user/latest/). On Windows, you can use Windows Linux Subsystem (WSL) and on Mac you might have to install Apptainer within a Docker environment. 

A container needs a definition file, e.g. `conf/nzesmenv.def`. This file lists the operating system, the compilers and the steps to build the libraries. It is a recipe for building the container.

To build the container, type
```
apptainer build nzesmenv.sif conf/nzesmenv.def
```
Now take a cup of coffee. 

### Building the container on NeSI

If you don't have access to a Linux system with Apptainer installed, you can also [build the container on Mahuika](https://support.nesi.org.nz/hc/en-gb/articles/6008779241999-Build-an-Apptainer-container-on-a-Milan-compute-node) by submitting the follwowing SLURM job
```
#!/bin/bash -e
#SBATCH --job-name=apptainer_build
#SBATCH --partition=milan
#SBATCH --time=0-04:00:00
#SBATCH --mem=30GB
#SBATCH --cpus-per-task=4

# load environment module
module purge
module load Apptainer

# recent Apptainer modules set APPTAINER_BIND, which typically breaks
# container builds, so unset it here
unset APPTAINER_BIND

# create a build and cache directory on nobackup storage
export APPTAINER_CACHEDIR="/nesi/nobackup/$SLURM_JOB_ACCOUNT/$USER/apptainer_cache"
export APPTAINER_TMPDIR="/nesi/nobackup/$SLURM_JOB_ACCOUNT/$USER/apptainer_tmpdir"
mkdir -p $APPTAINER_CACHEDIR $APPTAINER_TMPDIR
setfacl -b $APPTAINER_TMPDIR

apptainer build --force --fakeroot nzesmenv.sif conf/nzesmenv.def
```

Once the build completes you will end up with a file `nzesmenv.sif`, which you can copy across platforms.

## How to run a shell within a container

Assuming you have loaded the `Apptainer` module on Mahuika (or have the command `apptainer` available on your system),
```
apptainer shell nzesmenv.sif
```
will land you in an environment with compilers
```
Apptainer> which mpiifort
/opt/intel/oneapi/compiler/2023.0.0/linux/bin/intel64/ifort
```
and other tools installed. Installed dependencies live under `/software/{include,lib,bin}`. FCM has has been installed and is accessible,
```
Apptainer> fcm
FCM 2021.05.0 (/software/fcm)
Usage:
        fcm [APPLICATION] [OPTIONS] [ARGUMENTS]

Arguments:
    fcm provides the following applications:

        branch-create, bcreate, bc
        branch-delete, bdelete, bdel, brm
        branch-diff, bdiff, bdi
        branch-info, binfo
        branch-list, bls
        browse, trac, www
        build
        cfg-print, cfg
        cmp-ext-cfg
        conflicts, cf
        export-items
        extract
        gui
        keyword-print, kp
        loc-layout
        make
        mkpatch
        test-battery

    fcm overrides the following svn applications:

        add
        commit, ci
        delete, del, remove, rm
        diff, di
        merge
        switch, sw
        update, up

    fcm explicitly doesn't support the following svn applications:

        changelist

    Type "fcm help APPLICATION" for help on individual application.

    Type "svn help APPLICATION" for help on other svn application.
```

## How to run a command inside the container

If you want to run the `nc-config` command, for example,
```
apptainer exec nzesmenv.sif nc-config
``` 

## How to compile an application using the containerised environment

When running `apptainer shell` on Mahuika, directories (`$HOME`, `/nesi/project`, `/nesi/nobackup`) are mounted by default. Additional directories can be mounted with the `-B <dir>` option. 

You can use the compilers inside the container to compile your application. In the example below we compile a simple MPI C application
```
cat > myapp.c << EOF
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    // Initialize the MPI environment
    MPI_Init(NULL, NULL);

    // Get the number of processes
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    // Get the rank of the process
    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

    // Get the name of the processor
    char processor_name[MPI_MAX_PROCESSOR_NAME];
    int name_len;
    MPI_Get_processor_name(processor_name, &name_len);

    // Print off a hello world message
    printf("Hello world from processor %s, rank %d out of %d processors\n",
           processor_name, world_rank, world_size);

    // Finalize the MPI environment.
    MPI_Finalize();
}
EOF
apptainer exec nzesmenv.sif mpicc myapp.c -o myapp
```

## Running a containerised MPI application

You can either leverage the MPI inside the application
```
apptainer exec nzesmenv.sif mpiexec -n 4 ./myapp
```
or use the hosts's MPI. In the latter case, the MPI versions inside and on the host must be compatible. On NeSI's Mahuika we recommend to use the Intel MPI. In this case your SLURM script would look like:
```
cat > myapp.sl << EOF
#!/bin/bash -e
#SBATCH --job-name=runCase14       # job name (shows up in the queue)
#SBATCH --time=01:00:00       # Walltime (HH:MM:SS)
#SBATCH --hint=nomultithread
#SBATCH --mem-per-cpu=2g             # memory (in MB)
#SBATCH --ntasks=10        # number of tasks (e.g. MPI)
#SBATCH --cpus-per-task=1     # number of cores per task (e.g. OpenMP)
#SBATCH --output=%x-%j.out    # %x and %j are replaced by job name and ID
#SBATCH --error=%x-%j.err
#SBATCH --partition=milan

ml purge
ml Apptainer

module load intel        # load the Intel MPI
export I_MPI_FABRICS=ofi # turn off shm to run on multiple nodes

srun apptainer exec -B /opt/slurm/lib64/ nzesmenv.sif ./myapp
EOF
sbatch myapp.sl
```

## Caching your password when using rose

Container `nzesmenv.sif` comes with fcm, cylc and rose installed. To access the remote Met Office repos in password-less fashion, you will need to have an account on [MOSRS](https://code.metoffice.gov.uk/trac/home). Rose tasks may require you to have your password cached. To cache your password when using `rose` in the container, do
```bash
Apptainer> source /software/rose/bin/mosrs-setup-gpg-agent
```
You may need to enter your password twice. Check that your password has been cached with
```bash
Apptainer> rosie hello
```

## Building GCOM

GCOM is the communication library required by UM to build. At the moment of writing, GCOM requires FCM, Cylc 7 and Rose 1 to build . We recommend to use Cylc 7.9.9 on `w-clim01.maui.niwa.co.nz` to compile GCOM. 

### Getting GCOM

Start by checking out the code
```
fcm co file:///opt/niwa/um_sys/metoffice-science-repos/gcom/main/trunk gcom
cd gcom
```

### Configuring GCOM

Next, you'll need to copy the files in `<this_repo>/packages/gcom/rose-stem/site/niwa/suite.rc` to `rose-stem/site/niwa/` and `<this_repo>/packages/gcom/fc-make/machines/niwa_apptainer*.cfg` to `fc-make/machines/` under the `gcom` repo, respectively. You will need to edit the files `<this_repo>/packages/gcom/fc-make/machines/niwa_apptainer*.cfg` as these refer to the location of the `nzesmenv.sif` file. You may also have to change the bindings of the apptainer/singularity command if compiling on another platform.

Note: the provided configuration files use the older `singularity` command in place of `apptainer`. The `singularity` command also works in apptainer. 

### Compiling GCOM

Type
```
rose stem -v -v -v --group=apptainer_build
rose stem -v -v -v --group=apptainer_test
```
in the gcom directory.






