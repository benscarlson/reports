---
title: "Storrs HPC setup"
format: 
  html:
    self-contained: true
---

This tutorial shows you how to install common R packages and call simple parallel R scripts using `doMPI`.

First, make sure you have an account on Storres HPC. Then, authenticate to UConn's VPN. Once you have done this, you can SSH to to the Storrs HPC.

```bash
ssh <yourid>@login.storrs.hpc.uconn.edu
```

::: {.callout-note}
You can configure this command so that it is much simpler, e.g.

```bash
ssh storres
```

See the advanced section for more information.
:::

# Install packages

The HPC support site has a great resource for [installing R packages](https://kb.uconn.edu/space/SH/26033587623/R+Guide).

Most packages install with no problem, but others require you to point to libraries that are on the HPC. If a package install fails, contact HPC support staff, and they will help you. The two packages that I had a problem with were `tidyverse` and `terra`.

::: {.callout-note}
Below, I show how to install `tidyverse`, `terra` and `Rmpi`. In order to test MPI, you only need to install `Rmpi`. If that is your only goal, you can skip `tidyverse` and `terra`. If you don't load `gdal` and `cuda`, make sure you load `openmpi`.

```bash
openmpi/5.0.2
```
:::

Load R and other modules

```bash
module unload gcc
module load gdal/3.8.4 cuda/11.6 r/4.4.0
```

The modules `gdal` and `cuda` are required for `terra`. The `tidyverse` package installed only after I loaded the `gdal` and `cuda` modules, so I'm not sure if `tidyverse` depends on one, the other, or both.

Next, make a personal directory to store R packages. Start R.

```bash
mkdir ~/rlibs

R
```

Within R, you need to tell R about the personal directory for R packages you created.

``` r
.libPaths("~/rlibs")
```

::: {.callout-note}
If you want to avoid setting `.libPaths()` every time you start R, you can put this command into your `.Rprofile`
```bash
shell echo '.libPaths("\~/rlibs")' > ~/.Rprofile
```
:::

Set a convenience variable for the cran mirror

``` r
cran <- "https://cloud.r-project.org/"
```

#### Install `tidyverse`

This install takes a long time! It took 1-2 hours for me. It also initially failed (eventually) when installing the `ragg` package. So try installing that first.

``` r
install.packages("ragg", lib = "~/rlibs", repo = cran)
install.packages("tidyverse", lib = "~/rlibs", repo = cran) #Veery long install
```

#### Install `terra`

You need to point to the `proj` libraries and ensure you've loaded the `gdal` and `cuda` modules (see above).

``` r
install.packages("terra", lib = "~/rlibs", type = "source",   repo = cran
  configure.args = c("--with-sqlite3-lib=/gpfs/sharedfs1/admin/hpc2.0/apps/sqlite/3.45.2/lib", 
    "--with-proj-lib=/gpfs/sharedfs1/admin/hpc2.0/apps/proj/9.4.0/lib64"))
```

#### Install `Rmpi` and `doMPI`

This install command matches what is in HPC's install documentation, edited to match the specific version of `MPI`. Make sure the `mpi` module is loaded before installing.

``` r

install.packages("Rmpi", lib = "~/rlibs", repo = cran, 
  configure.args = c("--with-Rmpi-include=/gpfs/sharedfs1/admin/hpc2.0/apps/openmpi/5.0.2/include", 
    "--with-Rmpi-libpath=/gpfs/sharedfs1/admin/hpc2.0/apps/openmpi/5.0.2/lib", 
    "--with-Rmpi-type=OPENMPI", 
    "--with-mpi=/gpfs/sharedfs1/admin/hpc2.0/apps/openmpi/5.0.2"))
```

The `doMPI` package should install with no problems.

``` r
install.packages("doMPI", lib = "~/rlibs", repo = cran)
```

#### Install other packages

All other packages I tested installed without any issues. Make sure you install using the pattern

``` r
install.packages(<my package>,lib = "~/rlibs", repos=cran)
```

#### Install package builds (.tar.gz)

Assumes you have the file `CVmaxnet_0.0.0.9000.tar.gz` in the directory `r_pkg_builds`

This follows [Karl Broman] (https://kbroman.org/pkg_primer/pages/build.html)
You may not have to create the R_LIBS variable

```bash
R_LIBS=~/rlibs

module unload gcc
module load r/4.4.0

R CMD INSTALL --library=$R_LIBS ~/r_pkg_builds/CVmaxnet_0.0.0.9000.tar.gz
```

## Test MPI

There are many ways to use the HPC. I use MPI following Steve Weston, the author of the `doMPI` package, the O'Reilly book [Parallel R](https://www.oreilly.com/library/view/parallel-r/9781449317850/), a founder of Revolution Analytics, and an administrator of Yale's HPC. However, you don't need to use MPI to use the HPC; many other approaches are perfectly reasonable for the typical scenarios in R scripts in ecology.

You will normally submit batch jobs using SLURM, but you should first run scripts interactively to test them. To test interactively, first create a small script to test MPI. Then, request interactive nodes to run the script.

Put this R script somewhere that you can execute it. If you didn't set `.libPaths()` in `.Rprofile`, you need to add it to the script. Note I'm calling `Rscript` using the `#!` operator.

This script uses three tasks to compute the square root of the task number. Call the script `test_dompi.r`

`test_dompi.r`

``` r
#!/usr/bin/env Rscript

library(doMPI)

cl <- startMPIcluster(verbose=TRUE)
registerDoMPI(cl)

foreach(i=1:3) %dopar% {
  message(i)
  sqrt(i)
}

closeCluster(cl)
mpi.quit()
```

### Test interactively

Request four interactive tasks in the debug queue. The script uses three tasks, but you need to request four because the master process requires one task.

```bash
srun -n 4 -p debug --pty bash
```

Once you've been deposited into the interactive nodes, load the modules and request mpi to run the script. Again, you have to tell mpi to use four tasks.

```bash
module unload gcc
module load gdal/3.8.4 cuda/11.6 r/4.4.0

mpirun -n 4 R --slave -f test_dompi.r
```

After a slight pause, you should see package loading messages and then the output of a list that has the square root of the task number.

```         
Loading required package: foreach
Loading required package: foreach
Loading required package: foreach
Loading required package: foreach
Loading required package: iterators
Loading required package: iterators
Loading required package: iterators
Loading required package: iterators
Loading required package: Rmpi
Loading required package: Rmpi
Loading required package: Rmpi
Loading required package: Rmpi
[[1]]
[1] 1

[[2]]
[1] 1.414214

[[3]]
[1] 1.732051
```

Output from within the loop is saved to a log file for each task.

```bash
ls MPI*.log
```

```         
MPI_1_bsc23001_1038499.log  MPI_3_bsc23001_1038501.log
MPI_2_bsc23001_1038500.log
```

These files are where the output from the `message(i)` function is sent.

```bash
cat MPI_1_bsc23001_1038499.log
```

```         
starting worker loop: cores = 1
waiting for a taskchunk...
job environment of length 926 will be broadcast
initializing for new job 515511
executing taskchunk 1 containing 1 tasks
1 <---- This is the output from message(i)
returning results for taskchunk 1
waiting for a taskchunk...
cleaning up after job 515511 because job complete
waiting for a taskchunk...
shutting down
```

Once that runs, `exit` back to the login node.

### Test using SLURM

To run an mpi script using slurm, you create a small script that sets parameters then calls your parallel script. You can call slurm directly from the command line (see advanced section) but this is a bit more advanced so calling mpi through a slurm script is usually how tutorials introduce slurm. Call the script `test_dompi_slurm.sh`

`test_dompi_slurm.sh`

```bash
#!/bin/bash
#SBATCH -p debug
#SBATCH -n 4

module unload gcc
module load gdal/3.8.4 cuda/11.6 r/4.4.0

mpirun test_dompi.r
```

This script is essentially the same as the `mpirun` call above, except you use `#SBATCH` commands to set `-p` and `-n` parameters.

Call the script using `sbatch`

```bash
sbatch test_dompi_slurm.sh
```

The script should run after a short pause. You can check your place in the queue using `squeue --me`

This time, the script-level output is directed to `slurm-*.log`. The task-level output is again sent to individual `MPI*.log` files. 

## Advanced configuration

### Make an hostname alias

It becomes tedious to type `ssh <netid>@login.storrs.hpc.uconn.edu` every time you want 
to connect with the hpc. You can set up ssh so you can instead type `ssh storrs` to log in.

Just edit your `config` file and add the following. Replace <netid> with your netid.

```shell
vim ~/.ssh/config

Host storrs
    HostName login.storrs.hpc.uconn.edu
    User <netid>
    UseKeychain yes
    AddKeystoAgent yes
```

Test to make sure it works

```shell
ssh storrs
```

### Avoid typing your password every time

Typing your password every time to log in also gets annoying. You can avoid this 
by installing the public key on your local machine on to storrs.

Assuming you set up the `storrs` host alias above, just add the key to your keychain
and copy it to the hpc. It will ask you for your password but this should be for the last time.

```shell
ssh-add --apple-use-keychain ~/.ssh/id_rsa
ssh-copy-id storrs
```

Now when you connect to the hpc you won't need to supply your password

```shell
ssh storrs
```

TODO
* Calling slurm directly from the command line and passing script parameters


## Tips and tricks

### Run simple commands directly from a local shell

You don't need to connect to the ssh just to execute a simple command like creating a directory.
Intead, you can use the `ssh` command to excecute commands on the hpc without being
deposited into a remote shell. 

```bash
ssh storrs "mkdir -p ~/test"
```

You will receive the following message but I have confirmed with hpc support that 
you can disregard it.

`/etc/profile.d/slurm.sh: line 28: sjobs: command not found`

The error is related to a script that usually runs when you log into the hpc.
Since you are not logging into an interactive shell, the script fails. The important
point is that your command will have executed successfully.

TODO
* Code sharing/github setup
