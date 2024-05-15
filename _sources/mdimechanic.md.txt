# Using MDIMechanic

:::{admonition} Prerequisites
:class: note

Before completing this section, make sure you have completed the setup instructions in the [previous section](setup.md).
:::

In this tutorial, we will use the MDIMechanic package to run our code.
MDIMechanic is a Python package that provides an interface for running environments and MDI Engines. 
It orchestrates Docker containers using a streamlined set up process. 

The file `mdimechanic.yml` contains the configuration for the MDIMechanic environment.


To get started, execute the following command in your terminal:

:::{tab-set}

````{tab-item} bash

```bash
mdimechanic build
```
````
:::

While your image is building, we can examine the `mdimechanic.yaml` file to see what is happening. 
Looking at the file, you will see the following:

```yaml
code_name: 'mdi-ani-tutorial'
docker:
  image_name: 'mdi-ani-tutorial'

  build_image:
    - apt-get update && apt-get install -y curl
    - curl "http://vergil.chemistry.gatech.edu/psicode-download/Psi4conda-1.9.1-py311-Linux-x86_64.sh" -o Psi4conda-1.9.1-py311-Linux-x86_64.sh --keepalive-time 2
    - bash Psi4conda-1.9.1-py311-Linux-x86_64.sh -b -p $HOME/psi4conda
    - echo $'. $HOME/psi4conda/etc/profile.d/conda.sh\nconda activate' >> ~/.bashrc
    - source ~/.bashrc
    - /root/psi4conda/bin/pip install pymdi
    - /root/psi4conda/bin/pip install numpy
    - /root/psi4conda/bin/pip install torch --index-url https://download.pytorch.org/whl/cpu
    - /root/psi4conda/bin/pip install torchani
    - /root/psi4conda/bin/pip install matplotlib
    - /root/psi4conda/bin/pip install tabulate
    - /root/psi4conda/bin/pip cache purge
```

This file specifies the name of the code, the Docker image to use, and the commands to run to build the image.
The Docker image is built from a base image for MDI that has compilers and other necessary software installed.
The `build_image` section specifies the commands to run to install the necessary software for the tutorial.
We are installing Psi4, pymdi, numpy, torch, torchani, and matplotlib.
Notice that this container is configured to install the CPU version of PyTorch (to save space and improve build time).

## Using the Image

MDIMechanic has a command called `mdimechanic interactive` that will mount your local directory as a volume in the container and open a shell in the container.
Execute MDIMechanic interactive with the following command:

:::{tab-set}

````{tab-item} bash

```bash
mdimechanic interactive
```
````
:::

You are now in the container and can use the software installed. 
Check the directory contents using `ls` and confirm that it is the same as your local directory.

Open a Python interpreter and confirm that you can import torch and Psi4:

:::{tab-set}

````{tab-item} python

```python
import torch
import psi4
```
````
:::

If you can import these packages, you have successfully set up your environment and are ready to run the tutorial. 

The other important section in `mdimechanic.yml` is the `run_scripts` section.
In that section, you will see commands for launching LAMMPS as well as the MDI Driver we will be writing. 
If you are familiar with Docker, you may be interested to know that this section is used to specify a configuration for Docker Compose. 
It will be discussed further in the [Driver Tutorial](driver_tutorial.md).

In the next section, we will use MDI Mechanic and the `mdimechanic interactive` mode to run Psi4 and ANI for energy calculations.