# Profiling

There are a number of different ways to access profiling data on 
ARCHER2. In this section, we discuss the HPE Cray profiling tools,
CrayPat-lite and CrayPat. We also show how to get usage data
on currently running jobs from Slurm batch system.

You can also use [the Arm Forge tool](../data-tools/arm-forge.md)
to profile applications on ARCHER2

## CrayPat-lite

CrayPat-lite is a simplified and easy-to-use version of the Cray
Performance Measurement and Analysis Tool (CrayPat). CrayPat-lite
provides basic performance analysis information automatically, with a
minimum of user interaction, and yet offers information useful to users
wishing to explore a program's behaviour further using the full CrayPat
suite.

### How to use CrayPat-lite

1.  Ensure the `perftools-base` module is loaded.

    `module list`

2.  Load the `perftools-lite` module.

    `module load perftools-lite`

3.  Compile your application normally. An informational message from
    CrayPat-lite will appear indicating that the executable has been
    instrumented.

    ```
    cc -h std=c99  -o myapplication.x myapplication.c
    ```
    
    ```
    INFO: creating the CrayPat-instrumented executable 'myapplication.x' (lite-samples) ...OK  
    ```

4.  Run the generated executable normally by submitting a job.

    ```
    #!/bin/bash

    #SBATCH --job-name=CrayPat_test
    #SBATCH --nodes=4
    #SBATCH --ntasks-per-node=128
    #SBATCH --cpus-per-task=1
    #SBATCH --time=00:20:00

    #SBATCH --account=[budget code]
    #SBATCH --partition=standard
    #SBATCH --qos=standard

    # Launch the parallel program
    srun --hint=nomultithread --distribution=block:block mpi_test.x
    ```

5.  Analyse the data.

    After the job finishes executing, CrayPat-lite output should be printed
    to stdout (i.e. at the end of the job's output file). A new directory will
    also be created containing `.rpt` and `.ap2` files. The `.rpt` files are text
    files that contain the same information printed in the job's output file and
    the `.ap2` files can be used to obtain more detailed information, which can be
    visualized using the [Cray Apprentice2 tool](#cray-apprentice2).

### Further help

  - [CrayPat-lite User Guide](https://support.hpe.com/hpesc/public/docDisplay?docId=a00114942en_us&page=index.html)

## CrayPat

The Cray Performance Analysis Tool (CrayPat) is a powerful framework for
analysing a parallel application’s performance on Cray supercomputers.
It can provide very detailed information about the timing and performance
of individual application procedures.

CrayPat can perform two types of performance analysis, *sampling*
experiments and *tracing* experiments. A sampling experiment probes the
code at a predefined interval and produces a report based on the data
collected. A tracing experiment explicitly monitors the code
performance within named routines. Typically, the overhead associated
with a tracing experiment is higher than that associated with a sampling
experiment but provides much more detailed information. The key to
getting useful data out of a sampling experiment is to run your
profiling for a representative length of time.

### Sampling analysis

1.  Ensure the `perftools-base` module is loaded.

    `module list`

2.  Load `perftools` module.

    `module load perftools`

3.  Compile your code in the standard way always using the Cray compiler
    wrappers (ftn, cc and CC). Object files need to be made available to
    CrayPat to correctly build an instrumented executable for profiling
    or tracing, this means that the compile and link stage should be
    separated by using the `-c` compile flag.

    ```
    auser@ln01:/work/t01/t01/auser> cc -h std=c99 -c jacobi.c
    auser@ln01:/work/t01/t01/auser> cc jacobi.o -o jacobi
    ```

4.  To instrument the binary, run the `pat_build` command. This will 
    generate a new binary with `+pat` appended to the end (e.g. `jacobi+pat`).

    `auser@ln:/work/t01/t01/auser> pat_build jacobi`

5.  Run the new executable with `+pat` appended as you would with the
    regular executable. Each run will produce its own 'experiment
    directory' containing the performance data as `.xf` files
    inside a subdirectory called `xf-files` (e.g. running the 
    `jacobi+pat` instrumented executable might produce
    `jacobi+pat+12265-1573s/xf-files`).
6.  Generate report data with `pat_report`.

The `.xf` files contain the raw sampling data from the run and need to
be post-processed to produce useful results. This is done using the
`pat_report` tool which converts all the raw data into a summarised and
readable form. You should provide the name of the experiment directory as
the argument to `pat_report`.

    auser@ln:/work/t01/t01/auser> pat_report jacobi+pat+12265-1573s

    Table 1:  Profile by Function (limited entries shown)

    Samp% |  Samp |  Imb. |  Imb. | Group
            |       |  Samp | Samp% |  Function
            |       |       |       |   PE=HIDE
    100.0% | 849.5 |    -- |    -- | Total
    |--------------------------------------------------
    |  56.7% | 481.4 |    -- |    -- | MPI
    ||-------------------------------------------------
    ||  48.7% | 414.1 |  50.9 | 11.0% | MPI_Allreduce
    ||   4.4% |  37.5 | 118.5 | 76.6% | MPI_Waitall
    ||   3.0% |  25.2 |  44.8 | 64.5% | MPI_Isend
    ||=================================================
    |  29.9% | 253.9 |  55.1 | 18.0% | USER
    ||-------------------------------------------------
    ||  29.9% | 253.9 |  55.1 | 18.0% | main
    ||=================================================
    |  13.4% | 114.1 |    -- |    -- | ETC
    ||-------------------------------------------------
    ||  13.4% | 113.9 |  26.1 | 18.8% | __cray_memcpy_SNB
    |==================================================

This report will generate more files with the extension `.ap2` in the
experiment directory. These hold the same data as the `.xf` files but
in the post-processed form. Another file produced has an `.apa` extension
and is a text file with a suggested configuration for generating a
traced experiment. 

The `.ap2` files generated are used to view performance
data graphically with the [Cray Apprentice2 tool](#cray-apprentice2).

The `pat_report` command is able to produce many different profile
reports from the profiling data. You can select a predefined report with
the `-O` flag to `pat_report`. A selection of the most generally useful
predefined report types are:= listed below.

  - **ca+src** - Show the callers (bottom-up view) leading to the
    routines that have a high use in the report and include source code
    line numbers for the calls and time-consuming statements.
  - **load\_balance** - Show load-balance statistics for the high-use
    routines in the program. Parallel processes with minimum, maximum
    and median times for routines will be displayed. Only available with
    tracing experiments.
  - **mpi\_callers** - Show MPI message statistics. Only available with
    tracing experiments.

Example output:
    
    auser@ln01:/work/t01/t01/auser> pat_report -O ca+src,load_balance  jacobi+pat+12265-1573s

    Table 1:  Profile by Function and Callers, with Line Numbers (limited entries shown)

    Samp% |  Samp |  Imb. |  Imb. | Group
            |       |  Samp | Samp% |  Function
            |       |       |       |   PE=HIDE
    100.0% | 849.5 |    -- |    -- | Total
    |--------------------------------------------------
    |--------------------------------------
    |  56.7% | 481.4 | MPI
    ||-------------------------------------
    ||  48.7% | 414.1 | MPI_Allreduce
    3|        |       |  main:jacobi.c:line.80
    ||   4.4% |  37.5 | MPI_Waitall
    3|        |       |  main:jacobi.c:line.73
    ||   3.0% |  25.2 | MPI_Isend
    |||------------------------------------
    3||   1.6% |  13.2 | main:jacobi.c:line.65
    3||   1.4% |  12.0 | main:jacobi.c:line.69
    ||=====================================
    |  29.9% | 253.9 | USER
    ||-------------------------------------
    ||  29.9% | 253.9 | main
    |||------------------------------------
    3||  18.7% | 159.0 | main:jacobi.c:line.76
    3||   9.1% |  76.9 | main:jacobi.c:line.84
    |||====================================
    ||=====================================
    |  13.4% | 114.1 | ETC
    ||-------------------------------------
    ||  13.4% | 113.9 | __cray_memcpy_SNB
    3|        |       |  __cray_memcpy_SNB
    |======================================
    

### Tracing analysis

#### Automatic Program Analysis (APA)

We can produce a focused tracing experiment based on the results from
the *sampling* experiment using `pat_build` with the `.apa` file
produced during the sampling.

    auser@ln01:/work/t01/t01/auser> pat_build -O jacobi+pat+12265-1573s/build-options.apa

This will produce a third binary with extension `+apa`. This binary
should once again be run on the compute nodes and the name of the
executable changed to `jacobi+apa`. As with the sampling analysis, a
report can be produced using `pat_report`. For example:

    
    auser@ln01:/work/t01/t01/auser> pat_report jacobi+apa+13955-1573t

    Table 1:  Profile by Function Group and Function (limited entries shown)

    Time% |      Time |     Imb. |  Imb. |       Calls | Group
            |           |     Time | Time% |             |  Function
            |           |          |       |             |   PE=HIDE

    100.0% | 12.987762 |       -- |    -- | 1,387,544.9 | Total
    |-------------------------------------------------------------------------
    |  44.9% |  5.831320 |       -- |    -- |         2.0 | USER
    ||------------------------------------------------------------------------
    ||  44.9% |  5.831229 | 0.398671 |  6.4% |         1.0 | main
    ||========================================================================
    |  29.2% |  3.789904 |       -- |    -- |   199,111.0 | MPI_SYNC
    ||------------------------------------------------------------------------
    ||  29.2% |  3.789115 | 1.792050 | 47.3% |   199,109.0 | MPI_Allreduce(sync)
    ||========================================================================
    |  25.9% |  3.366537 |       -- |    -- | 1,188,431.9 | MPI
    ||------------------------------------------------------------------------
    ||  18.0% |  2.334765 | 0.164646 |  6.6% |   199,109.0 | MPI_Allreduce
    ||   3.7% |  0.486714 | 0.882654 | 65.0% |   199,108.0 | MPI_Waitall
    ||   3.3% |  0.428731 | 0.557342 | 57.0% |   395,104.9 | MPI_Isend
    |=========================================================================
    

#### Manual Program Analysis

CrayPat allows you to manually choose your profiling preference. This is
particularly useful if the APA mode does not meet your tracing analysis
requirements.

The entire program can be traced as a whole using `-w`:

    
    auser@ln01:/work/t01/t01/auser> pat_build -w jacobi
    

Using `-g`, a program can be instrumented to trace all function entry
point references belonging to the trace function group (mpi, libsci,
lapack, scalapack, heap, etc):

    auser@ln01:/work/t01/t01/auser> pat_build -w -g mpi jacobi

### Dynamically-linked binaries

CrayPat allows you to profile un-instrumented, dynamically linked
binaries with the `pat_run` utility. `pat_run` delivers profiling
information for codes that cannot easily be rebuilt. To use `pat_run`:

1.  Load the `perftools-base` module if it is not already loaded.

    `module load perftools-base`
    

2.  Run your application normally including the `pat_run` command right
    after your `srun` options.

    `srun [srun-options] pat_run [pat_run-options] program [program-options]`

3.  Use `pat_report` to examine any data collected during the execution
    of your application.

    `auser@ln01:/work/t01/t01/auser> pat_report jacobi+pat+12265-1573s`

Some useful `pat_run` options are as follows.

  - `-w`  Collect data by tracing.
  - `-g`  Trace functions belonging to group names. See the -g option in
    pat\_build(1) for a list of valid tracegroup values.
  - `-r`  Generate a text report upon successful execution.

### Further help

  - [CrayPat User Guide](https://support.hpe.com/hpesc/public/docDisplay?docId=a00114942en_us&page=index.html)

## Cray Apprentice2

Cray Apprentice2 is an optional GUI tool that is used to visualize and
manipulate the performance analysis data captured during program
execution. Cray Apprentice2 can display a wide variety of reports and graphs,
depending on the type of program being analyzed, the way in which the program
was instrumented for data capture, and the data that was collected during program
execution.

You will need to use CrayPat to first instrument your program and
capture performance analysis data, and then `pat_report` to generate
the `.ap2` files from the results. You may then use Cray Apprentice2 to
visualize and explore those files.

The number and appearance of the reports that can be generated using
Cray Apprentice2 is determined by the kind and quantity of data captured
during program execution, which in turn is determined by the way in
which the program was instrumented and the environment variables in
effect at the time of program execution. For example, changing the
PAT\_RT\_SUMMARY environment variable to 0 before executing the
instrumented program nearly doubles the number of reports available when
analyzing the resulting data in Cray Apprentice2.

    
    export PAT_RT_SUMMARY=0
    

To use Cray Apprentice2 (`app2`), load `perftools-base` module if it is
not already loaded.

    
    module load perftools-base
    

Next, open the experiment directory generated during the instrumentation
phase with Apprentice2.


    auser@ln01:/work/t01/t01/auser> app2 jacobi+pat+12265-1573s
    

## Hardware Performance Counters

!!! note
    A limited number of hardware performance counters are currently available
    on the ARCHER2 compute nodes. We are working to make more available.

Hardware performance counters can be used to monitor CPU and power events on
ARCHER2 compute nodes. The monitoring and reporting of hardware counter events is 
integrated with *CrayPat* - users should use CrayPat as described earlier in this
section to run profiling experiments to gather data from hardware counter events
and to analyse the data.

### Counters and counter groups available

You can explore which event counters are available on compute nodes by running the following
commands (replace `t01` with a valid budget code for your account):

```bash
module load perftools
srun --ntasks=1 --partition=standard --qos=short --account=t01 papi_avail
```

For convenience, the CrayPat tool provides predetermined groups of hardware event
counters. You can get more information on the hardware event counters available
through CrayPat with the following commands (on a login or compute node):

```bash
module load perftools
pat_help counters rome groups
```

If you want information on which hardware event counters are included in a group
you can type the group name at the prompt you get after running the command above.
Once you have finished browsing the help, type `.` to quit back to the command line.

#### Power/energy counters available

You can also access counters on power/energy consumption. To list the counters available
to monitor power/energy use you can use the command (replace `t01` with a valid budget
code for your account):

```bash
module load perftools
srun --ntasks=1 --partition=standard --qos=short --account=t01 papi_native_avail -i cray_pm
```

### Enabling hardware counter data collection

You enable the collection of hardware event counter data as part of a CrayPat 
experiment by setting the environment variable `PAT_RT_PERFCTR` to a comma separated
list of the groups/counters that you wish to measure.

For example, you could set (usually in your job submission script):

```bash
export PAT_RT_PERFCTR=1
```

to use the `1` counter group (summary with branch activity).

### Analysing hardware counter data

If you enabled collection of hardware event counters when running your profiling
experiment, you will automatically get a report on the data when you use the 
`pat_report` command to analyse the profile experiment data file.

You will see information similar to the following in the output from CrayPat for
different sections of your code (this example if for the case where 
`export PAT_RT_PERFCTR=1`, counter group: summary with branch activity, was 
set in the job submission script):

```
==============================================================================
  USER / main
------------------------------------------------------------------------------
  Time%                                                   88.3% 
  Time                                               446.113787 secs
  Imb. Time                                           33.094417 secs
  Imb. Time%                                               6.9% 
  Calls                       0.002 /sec                    1.0 calls
  PAPI_BR_TKN                 0.240G/sec    106,855,535,005.863 branch
  PAPI_TOT_INS                5.679G/sec  2,533,386,435,314.367 instr
  PAPI_BR_INS                 0.509G/sec    227,125,246,394.008 branch
  PAPI_TOT_CYC                            1,243,344,265,012.828 cycles
  Instr per cycle                                          2.04 inst/cycle
  MIPS                 1,453,770.20M/sec                        
  Average Time per Call                              446.113787 secs
  CrayPat Overhead : Time      0.2%           
```

### More information on hardware counters

More information on using hardware counters can be found in the appropriate section
of the HPE documentation:

* [HPE Performance Analysis Tools User Guide](https://support.hpe.com/hpesc/public/docDisplay?docLocale=en_US&docId=a00123563en_us)

## Performance and profiling data in Slurm

Slurm commands on the login nodes can be used to quickly and simply retrieve
information about memory usage for currently running and completed jobs.

There are three commands you can use on ARCHER2 to query job data from 
Slurm, two are standard Slurm commands and one is a script that provides
information on running jobs:

- The `sstat` command is used to display status information of a running
  job or job step
- The `sacct` command is used to display accounting data for all finished
  jobs and job steps within the Slurm job database.
- The `archer2jobload` command is used to show CPU and memory usage information
  for running jobs. (This script is based on one originally written for the
  [COSMA HPC facility](https://www.dur.ac.uk/icc/cosma/) at the University of
  Durham.)

We provide examples of the use of these three commands below.

For the `sacct` and `sstat` command, the memory properties we print out below are:

 - `AveRSS` - The mean memory use per node over the length of the job
 - `MaxRSS` - The maximum memory use per node measured during the job
 - `MaxRSSTask` - The maximum memory use from any process in the job

### Example 1: `sstat` for running jobs

To display the current memory use of a running job with the ID 123456:

```
sstat --format=JobID,AveCPU,AveRSS,MaxRSS,MaxRSSTask -j 123456
```

### Example 2: `sacct` for finished jobs

To display the memory use of a completed job with the ID 123456:

```
sacct --format=JobID,JobName,AveRSS,MaxRSS,MaxRSSTask -j 123456
```    

Another usage of `sacct` is to display when a job was submitted, started running and ended for a particular user:

```
sacct --format=JobID,Submit,Start,End -u auser
```

### Example 3: `archer2jobload` for running jobs

Using the `archer2jobload` command on its own with no options will show the current
CPU and memory use across compute nodes for all running jobs.

More usefully, you can provide a job ID to `archer2jobload` and it will show a summary
of the CPU and memory use for a specific job. For example, to get the usage data for job
123456, you would use:

```
auser@ln01:~> archer2jobload 123456
# JOB: 123456
CPU_LOAD            MEMORY              ALLOCMEM            FREE_MEM            TMP_DISK            NODELIST            
127.35-127.86       256000              239872              169686-208172       0                   nid[001481,001638-00
```

This shows the minimum CPU load on a compute node is 126.04 (close to the limit of 128 cores) with the maximum load 127.41 (indicating all the nodes are being used evenly). The minimum free memory is 171893 MB and the maximum free memory is 177224 MB.

If you add the `-l` option, you will see a breakdown per node:

```
auser@ln01:~> archer2jobload -l 276236
# JOB: 123456
NODELIST            CPU_LOAD            MEMORY              ALLOCMEM            FREE_MEM            TMP_DISK            
nid001481           127.86              256000              239872              169686              0                   
nid001638           127.60              256000              239872              171060              0                   
nid001639           127.64              256000              239872              171253              0                   
nid001677           127.85              256000              239872              173820              0                   
nid001678           127.75              256000              239872              173170              0                   
nid001891           127.63              256000              239872              173316              0                   
nid001921           127.65              256000              239872              207562              0                   
nid001922           127.35              256000              239872              208172              0 
```

### Further help with Slurm

The definitions of any variables discussed here and more usage information can be found in the man pages of `sstat` and `sacct`.

## Arm Forge

The Arm Forge tool also provides profiling capabilities. See:

- [ARCHER2 Arm Forge documentation](../data-tools/arm-forge.md)
