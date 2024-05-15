# Calculating Molecular Energies

Before we start to write a molecular dynamics driver for ANI, we might want to calculate the energies of a few molecules to make sure we get reasonable energy estimates and get a feel for using TorchANI.

In this tutorial, we will calculate the energy of water clusters using both ANI and Psi4. We will compare our energy results and compare the amount of time it takes to calculate the energy of a water cluster using ANI and Psi4.
The benchmarks we will do as part of this section are basic and are far from thorough.
However, they are a good starting point for understanding the performance of ANI and Psi4.  

## Water Cluster Configurations
We have provided a number of XYZ files for water clusters in the `xyz` folder. 
We will first work with files in the `xyz/scaling` folder to examine calculated energies and scaling behavior. 
These XYZ files come from a [database of water cluster minima](https://sites.uw.edu/wdbase/database-of-water-clusters/).


## Energy Calculation and Benchmarking Script

Open the script `xyz/energy_calculations.py` to see the starting code for this section.
The starting code includes functions for caluculating the energy of a system using ANI and Psi4.

:::{tab-set}

````{tab-item} xyz/energy_calculations.py
```python
"""
Retrieve xyz from a URL and calculate the energy using ANI
"""

import glob
import os 
import time

import psi4
import torch
import torchani

import numpy as np
import matplotlib.pyplot as plt

import tabulate


device = torch.device("cpu")
model = torchani.models.ANI2x(periodic_table_index=True).to(device)

def elements_to_atomic_numbers(elements):
    """
    Convert element symbols to atomic numbers using a predefined dictionary.

    Parameters
    ----------
    elements : list of str
        A list of element symbols.

    Returns
    -------
    list of int
        A list of atomic numbers corresponding to the element symbols.
    """

    conversion_dict =  {
    "H": 1,     # Hydrogen
    "C": 6,    # Carbon
    "N": 7,    # Nitrogen
    "O": 8,    # Oxygen
    "F": 9,    # Fluorine
    "Cl": 17,   # Chlorine
    "S": 16    # Sulfur
    }

    return [ conversion_dict[element] for element in elements ]

def calculate_ANI_energy(xyz_file):
    """
    Calculate the ANI2x energy for system of atoms.

    Parameters
    ----------
    elements:
        An array or list containing the elements of the atoms in the system.
    coordinates:
        An array containing the 3D coordinates of the atoms.
    
    Returns
    -------
    tuple(float, float)
        The energy of the system and the time taken to calculate the energy.
    """

    coordinates = np.loadtxt(xyz_file, skiprows=2, usecols=(1, 2, 3))
    elements = np.loadtxt(xyz_file, skiprows=2, usecols=0, dtype=str)

    coords_reshape = coordinates.reshape(1, -1, 3)
    atomic_numbers = elements_to_atomic_numbers(elements)

    torch_coords = torch.tensor(coords_reshape, requires_grad=True, device=device).float()
    atomic_numbers_torch = torch.tensor([atomic_numbers], device=device)
   
    # Calculate the energy
    start = time.time() 
    energy = model((atomic_numbers_torch, torch_coords)).energies
    end = time.time()   
    
    # Return the energy
    return energy.item(), end - start   

def calculate_Psi4_energy(xyz_file, name=None):
    """
    Calculate the energy of a system using Psi4 and the wb97x/6-31g* level of theory.

    Parameters
    ----------
    xyz_file:
        The path to the xyz file.
    name:   
        The name of the output file. If not provided, the name of the xyz file will be used.

    Returns
    -------
    tuple(float, float)
        The energy of the system as calculated by Psi4 using the wb97x/6-31g* level of theory.    
    """

    with open(xyz_file, 'r') as file:
        xyz_data = file.read()
    
    if not name:
        name = os.path.basename(xyz_file).split(".")[0]

    # set the geometry for Psi4
    psi4.geometry(xyz_data)
    
    # set output file for Psi4
    psi4.set_output_file(F'{name}.dat', False)
    
    # calculate the energy
    start = time.time()
    psi4_energy = psi4.energy("wb97x/6-31g*")
    end = time.time()   

    return psi4_energy, end - start


if __name__ == "__main__":

    xyz_files = glob.glob("scaling/*.xyz")

    energies = []
    times = []
    differences = []   

    for xyz in xyz_files:
        ani_energy, ani_time = calculate_ANI_energy(xyz)
        psi4_energy, psi4_time = calculate_Psi4_energy(xyz)

        n_water = int(os.path.basename(xyz).split(".")[0].split("_")[1])

        energies.append((n_water, ani_energy, psi4_energy))
        times.append((n_water, ani_time, psi4_time))
        differences.append((n_water, ani_energy - psi4_energy))

    # Sort the results
    energies = sorted(energies)
    times = sorted(times)
    differences = sorted(differences)

    # Print energies
    energy_headers = ["Number of Water Molecules", "ANI Energy (Ha)", "Psi4 Energy (Ha)"]
    print(tabulate.tabulate(energies, headers=energy_headers, tablefmt="grid"))

    # Print energy differences
    energy_diff_headers = ["Number of Water Molecules", "Energy Difference (Ha)"]
    print(tabulate.tabulate(differences, headers=energy_diff_headers, tablefmt="grid"))

    # Print times
    time_headers = ["Number of Water Molecules", "ANI Time (s)", "Psi4 Time (s)"]
    print(tabulate.tabulate(times, headers=time_headers, tablefmt="grid"))

    # Write the energies to a file
    with open("benchmark_energies.txt", "w") as f:
        f.write("Number of water molecules, ANI Energy (Ha), Psi4 Energy (Ha)\n")
        for energy in energies:
            f.write(f"{energy[0]}, {energy[1]}, {energy[2]}\n")
    
    # Write the timing information to a file
    with open("benchmark_times.txt", "w") as f:
        f.write("Number of water molecules, ANI Time (s), Psi4 Time (s)\n")
        for time in times:
            f.write(f"{time[0]}, {time[1]}, {time[2]}\n")

    # Create a plot of the energy differences
    plt.figure()
    plt.plot([x[0] for x in differences], [x[1] for x in differences], 'o-', label='Energy Difference (ANI - Psi4)')
    plt.xlabel("Number of Water Molecules")
    plt.ylabel("Energy Difference (Ha)")
    plt.legend()

    plt.savefig("energy_differences.png")

    # Create a plot of the timings
    plt.figure()
    plt.plot([x[0] for x in times], [x[1] for x in times], 'o-', label='ANI Time')
    plt.plot([x[0] for x in times], [x[2] for x in times], 's-', label='Psi4 Time')
    plt.xlabel("Number of Water Molecules")
    plt.ylabel("Time (s)")
    plt.legend()

    plt.savefig("timing.png") 
```
````
:::

Run this script by opening the MDI Mechanic container in interactive mode.

:::{tab-set}

````{tab-item} bash
```bash
mdi interactive
```
````
:::

Next, `cd` to the `xyz` directory and run the script.

:::{tab-set}

````{tab-item} bash
```bash
cd xyz
python energy_calculations.py
```
````
:::

While the script is running, you can examine the code to see how the energy calculations are done using ANI and Psi4.

## Exercises

### Exercise 1: Quantifying Scaling Behavior
Using the timings we obtained from the energy calculation (written to `benchmark_times.txt`), calculate the scaling behavior of ANI and Psi4.
The scaling behavior, also called the computational complexity, of an algorithm is the relationship between the size of the input and the time it takes to run the algorithm. 
This is also sometimes called "big O" notation, with the "O" standing for "order of magnitude".

To calculate the computational complexity, we can fit a curve to the timings and determine the order of the curve.

```{math}
T(N) = aN^b
```

Where {math}`T(N)` is the time taken to calculate the energy of a system with {math}`N` water molecules, {math}`a` is a constant, and {math}`b` is the order of the curve. The {math}`b` value is the scaling behavior of the algorithm.

When performing this fit, consider that you can linearize the equation by taking the logarithm of both sides:

```{math}
\log(T(N)) = \log(a) + b \log(N)
```

For performing this task, you might consider the [Numpy Polyfit function](https://numpy.org/doc/stable/reference/generated/numpy.polyfit.html), the [Scipy Curve Fit function](https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.curve_fit.html), or another fitting method of your choice.

**Hint** - You should not be re-running the energy calculations to complete this exercise. You can use the timings written to `benchmark_times.txt`.

**Hint 2** - Depending on you you choose to complete the problem, you may need to install additional Python packages. You can do this in `mdimechanic interactive` mode using `pip install`, or by modifying `mdimechanic.yaml` to include the packages you need.

### Exercise 2: Water Potential Energy Surface
Using the starting configuration for the water dimer (`xyz/water_dimer.xyz`), calculate the potential energy surface of the water dimer using ANI and Psi4. You should move one of the water molecules closer to or further from the other water molecule and calculate the energy at each step. Use the oxygen-oxygen distance as the reaction coordinate.

You can use this z-matrix template for the water dimer:

```
water_dimer = """
O1
H2 1 1.0
H3 1 1.0 2 104.52
x4 2 1.0 1 90.0 3 180.0
--
O5 2 **R** 4 90.0 1 180.0
H6 5 1.0 2 120.0 4 90.0
H7 5 1.0 2 120.0 4 -90.0
"""
```

Where `**R**` is the oxygen-oxygen distance.
This can be made into a Psi4 geometry object by replacing `**R**` with the desired separation distance.

You can use this as starting code:

```python
rvals = [1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1, 2.2, 2.3, 2.4, 2.5, 3.0, 3.5]
energies = []
for r in rvals:
    # Build a new molecule at each separation
    mol = psi4.geometry(water_dimer.replace('**R**', str(r)))
```

A Psi4 molecule can be saved to an xyz file using `mol.save_xyz_file("water_dimer_2.xyz", True)`. 
You can then use this xyz file to calculate the energy using ANI.

## Exercise 3: Evaluating the Accuracy of ANI for Benzene
The directory `xyz/benzene` contains a number of XYZ files with different configurations of benzene.
Write a script to calculate the energy of each benzene configuration using ANI and Psi4. 
Does ANI provide the correct relative energies for the different benzene configurations?





