# Tutorial

This is a tutorial on how to use the `mdi_rexmd` package.

This is a tutorial for users on Infer - the Virgina Tech compute cluster.

## Installation

Step 0: Download the containers

The containers are available on MolSSI's large file server.
You should create a directory to store the containers in your home directory.

```bash
mkdir -p apptainer_images
cd apptainer_images
wget https://pneuma.chem.vt.edu/apptainer/mdi_tinker9_cuda121.sif
wget https://pneuma.chem.vt.edu/apptainer/repex_latest.sif
``` 
Check to make sure you have the containers. When you type `ls`, you should see two files named `mdi_tinker9_cuda121.sif` and `repex_latest.sif`.

Step 1: Load required modules

```bash
module load shared
module load mpich/ge/gcc/64/3.3.2
module load containers/apptainer/1.3.3
```

Step 2: Clone the repository and install the package.

It is preferable for you to do this in a conda or virtual environment.
Make sure you have Python 3.10 or higher installed.

```bash
git clone https://github.com/janash/mdi_rexmd.git
cd mdi_rexmd
pip install -e .
```

## Running Replica Exchange
Navigate back to your home directory. Create a new directory called `repex_tutorial` and navigate into it.

```bash
mkdir repex_tutorial
cd repex_tutorial
```

Create a text file called `config.txt` with the following contents:

```bash
# Configuration for Replica Exchange Molecular Dynamics Setup
# Temperatures should be a comma-separated list of values in Kelvin.
temperatures=300,305,308,312

# Equilibration (optional)
# this will run a simulation for the specified number of steps without using mdi
# for each replica
equil=1000

# The number of simulation steps for each replica.
n_steps=10000

# The interval for output, in steps.
interval=100

# The key file for each replica
tinker_key=tinker.key

# The starting configuration for each replica
tinker_xyz=tinker.xyz

# The prm file to use
tinker_prm=amoebabio18.prm

# Path to the Apptainer image for the engine.
engine_image=~/apptainer_containers/mdi_tinker9_cuda121.sif

# Path to the Apptainer image for the driver.
driver_image=~/apptainer_containers/repex_latest.sif
```

You can generate the scripts to run replica exchange using the `mdi_rexmd` package's command line utility `setup_repex`. Note that `mdi_rexmd` must be installed in your environment (as directed in the instructions above).

```bash
setup_repex config.txt
```
The config file included in this tutorial specifies that each replica should be run for 1000 steps. 
After this, replica exchange will be run for 10000 steps with exchanges every 100 steps.

You can see this in the config file:
```
# Equilibration (optional)
# this will run a simulation for the specified number of steps without using mdi
# for each replica
equil=1000

# The number of simulation steps for each replica.
n_steps=10000

# The interval between replica exchange attempts.
interval=100
```

Running the previous bash command will generate a number of scripts in the current directory.

The most important script is called `run_repex.sh`. This is the "entry point" script that you will use to start the replica exchange simulation.

If you are running on a local cluster without a queueing system, you can run the script directly.

```bash
bash run_repex.sh
```

However, if you are running on HPC, like Infer, you should insert this script into a job submission script. 

Note that you should fill in the appropriate account and partition information for your job.
The compilation of Tinker 9 should work on V100 GPUs and T4 GPUs, but not on P100 GPUs.

```bash
#!/bin/bash

#SBATCH --job-name=rex_tutorial     
#SBATCH --partition=v100_normal_q
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --gres=gpu:1
#SBATCH --time=24:00:00
#SBATCH --output=mdi-repex_%j.out
#SBATCH --account=YOUR_ACCOUNT_HERE

# Load the Apptainer module
module load apps
module load containers/apptainer/1.3.3

# If you're using a conda environment, activate it here
# source activate mdi-test

# Submit the replica exchange script
bash run_repex.sh
```
This will submit the job to the queueing system. 

When your job is finished you should see something similar to the following in the output file `mdi-repex_*.out`:

```
Pair: 300.0 and 305.0   2       50      0.04
Pair: 308.0 and 312.0   3       50      0.06
Pair: 305.0 and 308.0   4       50      0.08
Acceptance rate: 0.06
```

This output shows the acceptance rate of exchanges between replicas. This will depend on the system and the temperatures you have chosen.
There will also be a folder called `remd_analysis` with detailed output of the exchange attempts.

Trajectories for each temperature will be stored in the `tinker_TEMP` directories, where `TEMP` is the temperature of the replica.