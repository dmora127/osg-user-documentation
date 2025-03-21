---
ospool:
  path: software_examples/other_languages_tools/conda-tarball.md
---

# Conda with Tarballs

The Anaconda/Miniconda distribution of Python is a common tool for installing and managing Python-based software and other tools. 

There are two ways of using Conda on the OSPool: with a tarball as described in this guide, or by installing Conda inside a
[custom Apptainer/Singularity container](../conda-container/). Either works well, but the container
solution might be better if your Conda environment requires access to non-Python tools.

## Overview
When should you use Miniconda as an installation method in OSG?

 * Your software has specific conda-centric installation instructions.
 * The above is true and the software has a lot of dependencies.
 * You mainly use Python to do your work.

Notes on terminology:

- **conda** is a Python package manager and package ecosystem that exists in parallel with `pip` and [PyPI](https://pypi.org/).
- **Miniconda** is a slim Python distribution, containing the minimum amount of packages necessary for a Python installation that can use conda.
- **Anaconda** is a pre-built scientific Python distribution based on Miniconda that has many useful scientific packages pre-installed.

<p style="color:blue;">To create the smallest, most portable Python installation possible, we recommend starting with Miniconda and installing only the packages you actually require.</p>

To use a Miniconda installation for your jobs, create your installation environment on the access point and send a zipped version to your jobs.

## Install Miniconda and Package for Jobs
In this approach, we will create an entire software installation inside Miniconda and then use a tool called `conda pack` to package it up for running jobs.

### 1. Create a Miniconda Installation

After logging into your access point, download the latest Linux [miniconda installer](https://docs.conda.io/en/latest/miniconda.html) and run it. For example, 

      [alice@ap00]$ wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
      [alice@ap00]$ sh Miniconda3-latest-Linux-x86_64.sh
      
Accept the license agreement and default options. At the end, you can choose whether or not to “initialize Miniconda3 by running conda init?” 
- If you enter "no", you would then run the **eval** command listed by the installer to “activate” Miniconda. If you choose “no” you’ll want to save this command so that you can reactivate the Miniconda installation when needed in the future. 
- If you enter "yes", miniconda will edit your .bashrc file and PATH environment variable so that you do not need to define a path to Miniconda each time you log in. If you choose "yes", before proceeding, you must log off and close your terminal for these changes to go into effect. Once you close your terminal, you can reopen it, log in to your access point, and proceed with the rest of the instructions below.  

### 2. Create a conda "Environment" With Your Packages

(If you are using an `environment.yml` file as described [later](#specifying-exact-dependency-versions), you should instead create the environment from your `environment.yml` file. If you don’t have an `environment.yml` file to work with, follow the install instructions in this section. We recommend switching to the `environment.yml` method of creating environments once you understand the “manual” method presented here.)

Make sure that you’ve activated the base Miniconda environment if you haven’t already. Your prompt should look like this:

      (base)[alice@ap00]$ 
To create an environment, use the `conda create` command and then activate the environment:

      (base)[alice@ap00]$ conda create -n env-name
      (base)[alice@ap00]$ conda activate env-name
Then, run the `conda install` command to install the different packages and software you want to include in the installation. How this should look is often listed in the installation examples for software (e.g. [Qiime2](https://docs.qiime2.org/2023.2/install/), [Pytorch](https://pytorch.org/get-started/locally/)).

      (env-name)[alice@ap00]$ conda install pkg1 pkg2
Some Conda packages are only available via specific Conda channels which serve as repositories for hosting and managing packages. If Conda is unable to locate the requested packages using the example above, you may need to have Conda search other channels. More detail are available at [https://docs.conda.io/projects/conda/en/latest/user-guide/concepts/channels.html.](https://docs.conda.io/projects/conda/en/latest/user-guide/concepts/channels.html)

Packages may also be installed via `pip`, but you should only do this when there is no `conda` package available.

Once everything is installed, deactivate the environment to go back to the Miniconda “base” environment.

      (env-name)[alice@ap00]$ conda deactivate
For example, if you wanted to create an installation with `pandas` and `matplotlib` and call the environment `py-data-sci`, you would use this sequence of commands:

      (base)[alice@ap00]$ conda create -n py-data-sci
      (base)[alice@ap00]$ conda activate py-data-sci
      (py-data-sci)[alice@ap00]$ conda install pandas matplotlib
      (py-data-sci)[alice@ap00]$ conda deactivate
      (base)[alice@ap00]$ 

> ### More About Miniconda
> See the [official conda documentation](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html) for more information on creating and managing environments with **conda**.

### 3. Create Software Package
Make sure that your job’s Miniconda environment is created, but deactivated, so that you’re in the “base” Miniconda environment:

      (base)[alice@ap00]$ 
Then, run this command to install the `conda pack` tool:

      (base)[alice@ap00]$ conda install -c conda-forge conda-pack
Enter `y` when it asks you to install.

Finally, use `conda pack` to create a zipped tar.gz file of your environment (substitute the name of your conda environment where you see `env-name`), set the proper permissions for this file using `chmod`, and check the size of the final tarball:

      (base)[alice@ap00]$ conda pack -n env-name
      (base)[alice@ap00]$ chmod 644 env-name.tar.gz
      (base)[alice@ap00]$ ls -sh env-name.tar.gz
When this step finishes, you should see a file in your current directory named `env-name.tar.gz`.

### 4. Check Size of Conda Environment Tar Archive
The tar archive, `env-name.tar.gz`, created in the previous step will be used as input for subsequent job submission. As with all job input files, you should check the size of this Conda environment file. **If >1GB in size, you should move the file to either your `/public` or `/protected` folder, and transfer it to/from jobs using the `osdf:///` link, as described in [Overview: Data Staging and Transfer to Jobs](../../htc_workloads/managing_data/overview.md).** This is the most efficient way to transfer large files to/from jobs. 

### 5. Create a Job Executable
The job will need to go through a few steps to use this “packed” conda environment; first, setting the `PATH`, then unzipping the environment, then activating it, and finally running whatever program you like. The script below is an example of what is needed (customize as indicated to match your choices above). For future reference, let's call this executable `conda_science.sh`. 

      #!/bin/bash
      # File Name: science_with_conda.sh

      # have job exit if any command returns with non-zero exit status (aka failure)
      set -e

      # replace env-name on the right hand side of this line with the name of your conda environment
      ENVNAME=env-name

      # if you need the environment directory to be named something other than the environment name, change this line
      ENVDIR=$ENVNAME

      # these lines handle setting up the environment; you shouldn't have to modify them
      export PATH
      mkdir $ENVDIR
      tar -xzf $ENVNAME.tar.gz -C $ENVDIR
      . $ENVDIR/bin/activate

      # modify this line to run your desired Python script and any other work you need to do
      python3 hello.py

### 6. Submit Jobs
In your HTCondor submit file, make sure to have the following:

* Your executable should be the the bash script you created in [step 5](#5-create-a-job-executable).
* Remember to transfer your Python script and the environment `tar.gz` file to the job. If the `tar.gz` file is larger than 1GB, please move the file to either your `/protected` or `/public` directories and use the `osdf:///` file delivery mechanism as described above. 

An example submit file could look like: 

```
# File Name: conda_submission.sub

# Specify your executable (single binary or a script that runs several
#  commands) and arguments to be passed to jobs. 
#  $(Process) will be a integer number for each job, starting with "0"
#  and increasing for the relevant number of jobs.
executable = science_with_conda.sh
arguments = $(Process)

# Specify the name of the log, standard error, and standard output (or "screen output") files.

log = science_with_conda.log
error = science_with_conda.err
output = science_with_conda.out

# Transfer any file needed for our job to complete. 
transfer_input_files = osdf:///ospool/apXX/data/alice/env-name.tar.gz, hello.py

In the line above, the `XX` in `apXX` should be replaced with the numbers corresponding to your access point. 
# Specify Job duration category as "Medium" (expected runtime <10 hr) or "Long" (expected runtime <20 hr). 
+JobDurationCategory = “Medium”

# Tell HTCondor requirements (e.g., operating system) your job needs, 
# what amount of compute resources each job will need on the computer where it runs.
requirements = (OSGVO_OS_STRING == "RHEL 9")
request_cpus = 1
request_memory = 1GB
request_disk = 5GB

# Tell HTCondor to run 1 instance of our job:
queue 1
```

      
## Specifying Exact Dependency Versions

An important part of improving reproducibility and consistency between runs is to ensure that you use the correct/expected versions of your dependencies.

When you run a command like `conda install numpy` `conda` tries to install the most recent version of `numpy` For example, `numpy` version `1.22.3` was released on Mar 7, 2022. To install exactly this version of numpy, you would run `conda install numpy=1.22.3` (the same works for `pip` if you replace `=` with `==`). We recommend installing with an explicit version to make sure you have exactly the version of a package that you want. This is often called “pinning” or “locking” the version of the package.

If you want a record of what is installed in your environment, or want to reproduce your environment on another computer, conda can create a file, usually called `environment.yml`, that describes the exact versions of all of the packages you have installed in an environment. This file can be re-used by a different conda command to recreate that exact environment on another computer.

To create an `environment.yml` file from your currently-activated environment, run

      [alice@ap00]$ conda env export > environment.yml
This `environment.yml` will pin the exact version of every dependency in your environment. This can sometimes be problematic if you are moving between platforms because a package version may not be available on some other platform, causing an “unsatisfiable dependency” or “inconsistent environment” error. A much less strict pinning is

      [alice@ap00]$ conda env export --from-history > environment.yml
which only lists packages that you installed manually, and **does not pin their versions unless you yourself pinned them during installation**. If you need an intermediate solution, it is also possible to manually edit `environment.yml` files; see the [conda environment documentation](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#) for more details about the format and what is possible. In general, exact environment specifications are simply not guaranteed to be transferable between platforms (e.g., between Windows and Linux). **We strongly recommend using the strictest possible pinning available to you**.

To create an environment from an `environment.yml` file, run

      [alice@ap00]$ conda env create -f environment.yml
By default, the name of the environment will be whatever the name of the source environment was; you can change the name by adding a `-n \<name>` option to the `conda env create` command.

If you use a source control system like `git`, we recommend checking your `environment.yml` file into source control and making sure to recreate it when you make changes to your environment. Putting your environment under source control gives you a way to track how it changes along with your own code.

If you are developing software on your local computer for eventual use on the Open Science Pool, your workflow might look like this:

1. Set up a conda environment for local development and install packages as desired (e.g., `conda create -n science; conda activate science; conda install numpy`).
2. Once you are ready to run on the Open Science Pool, create an `environment.yml` file from your local environment (e.g., `conda env export > environment.yml`).
3. Move your `environment.yml` file from your local computer to the submit machine and create an environment from it (e.g., `conda env create -f environment.yml`), then pack it for use in your jobs, as per [Create Software Package](#3-create-software-package) above.


More information on conda environments can be found in [their documentation](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#).
