# Set Up

## GitHub Codespaces (Recommended)
We recommend using the [GitHub Codespaces](https://codespaces.new/MolSSI-MDI/mdi-ani-workshop) we have prepared for this workshop.
GitHub Codespaces are cloud-based development environments that are created directly from a GitHub repository.
This Codespace will have all of the necessary software installed and configured for you to complete the tutorial.

To use GitHub Codespaces, you must first [sign up for a GitHub account](https://github.com/signup) if you do not have one. 
Then you can click the button below to open the code space.

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/MolSSI-MDI/mdi-ani-workshop)

## Installing the Software Locally
You may also choose to install software and run the tutorial locally. 
In order to do so, you will need to have [Docker](https://docs.docker.com/get-docker/) and Python installed on your machine.
We recommend using the [conda package manager](https://docs.conda.io/en/latest/miniconda.html) for Python installation and environment management. 
If you'd like to know more detailed instructions as well as MolSSI's recommendations for computer set-up, see [these instructions](https://education.molssi.org/python-package-best-practices/setup.html) from our Best Practices workshop.

After ensuring you have the necessary software installed, create an environment for this project then install mdimechanic using pip:

:::{tab-set}

````{tab-item} bash

```bash
conda create -n mdi-ani-workshop python=3.11
pip install mdimechanic
```

````
:::

To complete this tutorial, clone the [starting repository](https://github.com/MolSSI-MDI/mdi-ani-workshop.git).

````{tab-set} 
```{tab-item} shell

:::{code-block} shell  
git clone https://github.com/MolSSI-MDI/mdi-ani-workshop.git
:::

```
````

