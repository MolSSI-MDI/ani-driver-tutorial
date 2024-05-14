# MDI Driver Tutorial

In this setion, we will use the MolSSI Driver Interface to perform molecular dynamics simulations with TorchANI.

:::{admonition} Prerequisites
:class: note

Before starting this tutorial, you should have already completed the [Computer Set Up](setup).
It is also suggested that you have completed the [TorchANI Tutorial](torchani_tutorial).

:::

To complete this tutorial, clone the [starting repository](https://github.com/MolSSI-MDI/mdi-ani-workshop.git).

````{tab-set} 
```{tab-item} shell

:::{code-block} shell  
git clone https://github.com/MolSSI-MDI/mdi-ani-workshop.git
:::

```
````

## Examining Initial Files

The starting repository contains a set of files we will need to complete this tutorial.
We will be using TorchANI to compute forces on atoms given a particular molecular configuration,
passing those forces to LAMMPS using MDI, then allowing LAMMPs to perform the dynamics steps.

When you clone the repository, you will see the following directory structure:

```
├── README.md
├── docker
│   └── Dockerfile
├── lammps
│   └── water
│       ├── lammps.data
│       ├── lammps.in
│       └── outfiles
├── mdi-ani-tutorial
│   ├── mdi-ani-tutorial.py
│   └── util.py
└── mdimechanic.yml
```

This is our starting set of files. 
This is roughly the set of files you would obtain from running the MDI CookieCutter. 
In this case, we have added some starting files to perform a simulation using LAMMPS.
We've also added several utility functions to `util.py` that will help use complete this tutorial more easily.

All of the files associated with the LAMMPS simulation are in the `lammps` directory.
If you view these files, you will see a folder called `water` that contains the LAMMPS input file `lammps.in` and the LAMMPS data file `lammps.data`.
This will be our starting configuration for our simulation using TorchANI.

### Python Driver

Next, open the file `mdi-ani-tutorial.py` in the `mdi-ani-tutorial` directory.
This file contains the Python code that will be used to run the simulation.

To start, you will see the following code:

````{tab-set} 

```{tab-item} mdi-ani-tutorial.py

:::{code-block} python
import sys
import warnings

# Import the MDI Library
import mdi

# Import MPI Library
try:
    from mpi4py import MPI

    use_mpi4py = True
    mpi_comm_world = MPI.COMM_WORLD
except ImportError:
    use_mpi4py = False
    mpi_comm_world = None

# Import parser
from util import create_parser, connect_to_engines

if __name__ == "__main__":

    # Read in the command-line options
    args = create_parser().parse_args()

    mdi_options = args.mdi

    if mdi_options is None:
        mdi_options = (
            "-role DRIVER -name driver -method TCP -port 8021 -hostname localhost"
        )
        warnings.warn(f"Warning: -mdi not provided. Using default value: {mdi_options}")

    # Initialize the MDI Library
    mdi.MDI_Init(mdi_options)

    # Get the correct MPI intra-communicator for this code
    mpi_comm_world = mdi.MDI_MPI_get_world_comm()

    engines = connect_to_engines(1)

    ###########################
    # Perform the simulation
    ###########################

    # Send the "EXIT" command to each of the engines
    for comm in engines.values():
        mdi.MDI_Send_Command("EXIT", comm)
:::

```
````

The code that is present in the file to start is boilerplate code that initializes the MDI Library and connects to the engines (in this case, LAMMPS).
We only need to worry about adding code in the section that says

```
###########################
# Perform the simulation
###########################
```

### MDI Mechanic Configuration
The second important file is the `mdimechanic.yml` file.
This file contains the configuration for the MDI Mechanic software.
MDI Mechanic is a utility tool that can perform many functions related to MDI, including launching engines and running simulations.
Under the hood, MDI Mechanic uses a software called Docker to launch engines and drivers. 

We've already configured the `mdimechanic.yml` file to install necessary dependencies for this tutorial and launch the LAMMPS engine.
At the end of the file, you will see 

```
lmp -mdi "-role ENGINE -name LAMMPS -method TCP -port 8021 -hostname ani-tutorial" -l outfiles/log.lammps -in lammps.in > outfiles/water_tutorial.out
```

This is a launch command for LAMMPS.
If you've used LAMMPS before, you will recognize this as the command to run a LAMMPS simulation with an additional flag `-mdi` that tells LAMMPS to connect to the MDI driver.

The first time you use this repository, you will need to run `mdimechanic build`

````{tab-set} 

```{tab-item} shell 

:::{code-block} shell 
mdimechanic build
:::
```

````

To confirm that your set up is correct, you can run the following command:

````{tab-set} 

```{tab-item} shell 

:::{code-block} shell 
mdimechanic run --name ani-tutorial
:::
```

````
This command will launch the LAMMPS engine and the MDI driver.
Upon successful completion of this command, you should see the following output:

```
Running a custom calculation with MDI Mechanic.
====================================================
================ Output from Docker ================
====================================================
Attaching to temp-ani-tutorial-1, temp-lammps-1
temp-ani-tutorial-1 exited with code 0
temp-lammps-1 exited with code 0

====================================================
============== End Output from Docker ==============
====================================================
Success: The driver ran to completion.
```

You will also see several files have appeared in the `lammps/water/outfiles` directory.
You will see `log.lammps` and `water_tutorial.out`.
Both of these will show that LAMMPS started.
However, you will not see any time step information because we have not yet instructed LAMMPS to run any steps through the input file or MDI Driver.

## Getting Started with MDI

Now that we have confirmed that the MDI driver and LAMMPS engine are running correctly, we can write the code that will perform the simulation.
We will build this section up slowly. 
A key concept that we will need to understand for this section is the concept of an "engine" in our driver. 
If you look at the starting driver code, you will see the following line:

``` 
engines = connect_to_engines(1)
```

The `connect_to_engines` function is a helper function that will connect to the engines that are running.
In our case, we have only one engine running, which is LAMMPS.
The `engines` variable is a dictionary that contains the MDI communicator for each engine.

We can examine the engines that are running by adding the following code to the `# Perform the simulation` section:

````{tab-set} 

```{tab-item} python

:::{code-block} python
# Print the engines that are running
for name, comm in engines.items():
    print(f"Engine {name} is running.")
:::
```
````

If you rerun the driver code using `mdimechanic run --name ani-tutorial`, you will see the following output:

```
Running a custom calculation with MDI Mechanic.
====================================================
================ Output from Docker ================
====================================================
Attaching to temp-ani-tutorial-1, temp-lammps-1
temp-ani-tutorial-1  | Engine LAMMPS is running.
temp-ani-tutorial-1 exited with code 0
temp-lammps-1 exited with code 0

====================================================
============== End Output from Docker ==============
====================================================
Success: The driver ran to completion.
```

For convenience, we will create a variable called `lammps` to use for the rest of the tutorial. 

Add the following after the loop in your code:

```python
lammps = engines["LAMMPS"]
```

At the end of this section, your code should look like the following:

````{tab-set} 

```{tab-item} mdi-ani-tutorial.py

:::{code-block} python
import sys
import warnings

# Import the MDI Library
import mdi

# Import MPI Library
try:
    from mpi4py import MPI

    use_mpi4py = True
    mpi_comm_world = MPI.COMM_WORLD
except ImportError:
    use_mpi4py = False
    mpi_comm_world = None

# Import parser
from util import create_parser, connect_to_engines

if __name__ == "__main__":

    # Read in the command-line options
    args = create_parser().parse_args()

    mdi_options = args.mdi

    if mdi_options is None:
        mdi_options = (
            "-role DRIVER -name driver -method TCP -port 8021 -hostname localhost"
        )
        warnings.warn(f"Warning: -mdi not provided. Using default value: {mdi_options}")

    # Initialize the MDI Library
    mdi.MDI_Init(mdi_options)

    # Get the correct MPI intra-communicator for this code
    mpi_comm_world = mdi.MDI_MPI_get_world_comm()

    engines = connect_to_engines(1)

    ###########################
    # Perform the simulation
    ###########################

    for name, comm in engines.items():
        print(f"Engine {name} is running.")

    lammps = engines["LAMMPS"]

    # Send the "EXIT" command to each of the engines
    for comm in engines.values():
        mdi.MDI_Send_Command("EXIT", comm)
:::
```

````

## Sending and Receiving Data
When using MDI, we retrieve data from the engine by sending a command to the engine and then receiving the data.
The first command we use is the `mdi.MDI_Send_Command` function. 
The argument to this function is the command to send and the engine we would like to send the command to.
When using `MDI_Send_Command`, we use `<` to tell the engine to send data to the driver, and `>` to tell the driver to send data to the engine.
For example, to get the number of atoms in the system, we first call `mdi.MDI_Send_Command("<NATOMS", lammps)` to tell LAMMPS to send the number of atoms.

After we have sent the command to the engine, we have to retrieve the data.
This is done with the `mdi.MDI_Recv` function.
When using `mdi.MDI_Recv`, we have to tell MDI how many values we expect to get back and the data type for the value or values we expect to receive.
They syntax is

```python
mdi.MDI_Recv(EXPECTED_NUMBER_OF_VALUES, DATA_TYPE, ENGINE_COMMUNICATOR)
```

The expected data types are defined in the MDI library.
For example, `mdi.MDI_INT` is the data type for an integer, and `mdi.MDI_DOUBLE` is the data type for a double (decimal) value.

Add the following code to the `# Perform the simulation` section to get the number of atoms in the system:

````{tab-set} 

```{tab-item} python

:::{code-block} python
# Get the number of atoms in the system
mdi.MDI_Send_Command("<NATOMS", lammps)
natoms = mdi.MDI_Recv(1, mdi.MDI_INT, lammps)

print(f"The number of atoms in the system is {natoms}.")
:::
```
````

These set of commands first tells LAMMPS to send the number of atoms in the system (`mdi.MDI_Send_Command("<NATOMS", lammps)`). 
After that, we receive the number of atoms from LAMMPS. Using `mdi.MDI_Recv`, we tell lammps we are expecting `1` value for `natoms` and that its data type is an integer.

````{admonition} Check Your Understanding 
:class: exercise

Retrieve the box vectors from LAMMPS and print it to the screen.

The command to get the box vectors is `<CELL`. You expect this to contain 9 values (3 for each dimension).
The data type we expect is an mdi.MDI_DOUBLE.

```{admonition} Solution
:class: solution dropdown

:::{code-block} python
# Get the box size from LAMMPS  
mdi.MDI_Send_Command("<CELL", lammps)   
box_vects = mdi.MDI_Recv(9, mdi.MDI_DOUBLE, lammps)

print(f"The box size is {box_vects}.")
:::

```     

````

With the above two commands, we have retrieved much of the information we know that we need when using TorchANI to compute forces.
Other information we will need are the atomic identities and atomic positions. 

We will now add code to get the atomic identities.
LAMMPS does not store atomic identities directly, but we can use the `MASSES` command to get the masses of the atoms.

Add the following code to the `# Perform the simulation` section to get the atomic masses:

````{tab-set} 

```{tab-item} python

:::{code-block} python
# Get the atomic masses
mdi.MDI_Send_Command("<MASSES", lammps)
masses = mdi.MDI_Recv(natoms, mdi.MDI_DOUBLE, lammps)
:::
```
````

To retrieve the coordinates of the atoms, we use the `<COORDS` command.
We expect there to be `3 * natoms` values returned, where each set of three values corresponds to the x, y, and z coordinates of an atom.

Add the following code to the `# Perform the simulation` section to get the atomic coordinates:

````{tab-set} 

```{tab-item} python

:::{code-block} python 
# Get the atomic coordinates
mdi.MDI_Send_Command("<COORDS", lammps)
coords = mdi.MDI_Recv(3 * natoms, mdi.MDI_DOUBLE, lammps)
print(f"There are {len(coords)} coordinates in the system.")
:::

```
````

:::{admonition} Exercise - Units
:class: exercise

Check the values printed when you print your box vectors and atomic coordinates.
Are they the same as was specified in the LAMMPS input file? 
What could be happening?

```{admonition} Solution
:class: solution dropdown 

When you print the box vectors, you will see that the length of each side is 11.735. 
This is not the same as our input value from the LAMMPS input file (6.21).

This is because LAMMPS uses a different unit system than the one we are using in our driver code.
In LAMMPS, the units are in Angstroms, while in our driver code, we are using atomic units (Bohr radii).

```   
:::

### Fixing Units

:::{admonition} MDI Units
:class: caution

When using MDI, it is important to remember that the units used by the engine may not be the same as the units you are using in your driver code.
MDI always expects to send and receive data in atomic units.
This means that if you are using a different unit system in your engine, you will need to convert the units to atomic units before sending or receiving data.
:::

Let's convert the units of the box vectors and atomic coordinates to atomic units.
We will first retrieve a conversion factor from MDI, then convert the units of the box vectors and atomic coordinates.  
It will be simplest to do this if we are using NumPy arrays, so we will convert the coordinates to a NumPy array.

````{tab-set} 

```{tab-item} python

:::{code-block} python
bohr_to_angstrom = mdi.MDI_Conversion_Factor("bohr","angstrom")

box_vects = np.array(box_vects) * bohr_to_angstrom
coords = np.array(coords) * bohr_to_angstrom

print(f"The box size is {box_vects}.")
:::
```
````

## Driving a Simulation using MDI

Now that we've seen how to retrieve data from LAMMPS, we can start to drive a simulation using MDI.
MDI is capable of "driving" a simulation code, which means we can send it to various points in the simulation code to perform tasks that you choose.
This is accomplished through the concept of "nodes" in MDI.
In an MDI engine, the `@INIT_MD` node is the point where a molecular dynamics simulation is initialized.
At this point, system properties like the number of atoms, the box size, and the atomic positions are set.
Note that this initialization is dependent on the input file that you prepare for LAMMPS.

Whenever we want to send a command to an engine, we use the `MDI_Send_Command` function.
This function takes two arguments: the command to send and the communicator for the engine.


Add the following code to the `# Perform the simulation` section to tell LAMMPS to initialize molecular dynamics:

````{tab-set} 

```{tab-item} python

:::{code-block} python
# Send the INIT_MD command to LAMMPS
mdi.MDI_Send_Command("@INIT_MD", lammps)
:::
```
````    

Just running this code won't look like it did anything. 
However, internally, LAMMPS has initialized the molecular dynamics simulation and moved to the specified node.

To perform dynamics, we will want to send MDI to the `@FORCES` node for our desired number of steps.
We will do this by first setting a variable for the number of steps we want to run and then sending the `@FORCES` command to LAMMPS for each step.

````{tab-set} 

```{tab-item} python

:::{code-block} python  
# Set the number of steps to run    
nsteps = 10

# Run the dynamics  
for i in range(nsteps):    
    mdi.MDI_Send_Command("@FORCES", lammps)
:::
```
````

If you check the output of the LAMMPS engine, you will see that it has run the dynamics steps.
If you check `log.lammps` or `water_tutorial.out` in your `lammps/water/outfiles` directory, you will see output similar to the following:

```
Per MPI rank memory allocation (min/avg/max) = 7.902 | 7.902 | 7.902 Mbytes
------------ Step              0 ----- CPU =            0 (sec) -------------
TotEng   =      -122.2700 KinEng   =         0.0000 Temp     =         0.0000 
PotEng   =      -122.2700 E_bond   =         0.0000 E_angle  =         0.0000 
E_dihed  =         0.0000 E_impro  =         0.0000 E_vdwl   =        32.8080 
E_coul   =       285.5475 E_long   =      -440.6255 Press    =      5318.3164
------------ Step              1 ----- CPU =   0.00097576 (sec) -------------
TotEng   =       -41.4920 KinEng   =         0.0432 Temp     =         0.9656 
PotEng   =       -41.5352 E_bond   =         0.0000 E_angle  =         0.0000 
E_dihed  =         0.0000 E_impro  =         0.0000 E_vdwl   =        27.3276 
E_coul   =       371.6457 E_long   =      -440.5085 Press    =     20062.1465
------------ Step              2 ----- CPU =  0.001521911 (sec) -------------
TotEng   =       -41.4923 KinEng   =         0.1493 Temp     =         3.3400 
PotEng   =       -41.6417 E_bond   =         0.0000 E_angle  =         0.0000 
E_dihed  =         0.0000 E_impro  =         0.0000 E_vdwl   =        27.2650 
E_coul   =       371.6024 E_long   =      -440.5091 Press    =     19377.1620
```

This output shows the energy of the system at each step.
Right now, this energy is the energy that is calculated by LAMMPS for the forcefield we've specified in our input files. 
Our task in the next section will be to replace the forces LAMMPS calculates with those calculated by TorchANI.

## Calculating Forces using TorchANI

We now have all of the information we need to compute forces using TorchANI.
If you're interested in understanding more about how to use TorchANI, please see the [TorchANI Tutorial](torchani_tutorial).

When performing our MD, we know that we will need the updated coordinates at each step. 
Modify your for loop to retrieve the coordinates using MDI. 
We won't do anything with them yet, but we will need them in the next section.

````{tab-set} 

```{tab-item} python

:::{code-block} python
# Run the dynamics
for i in range(nsteps):
    
    # Send to @FORCES node
    mdi.MDI_Send_Command("@FORCES", lammps) 

    # Get the atomic coordinates
    mdi.MDI_Send_Command("<COORDS", lammps)
    coords = mdi.MDI_Recv(3 * natoms, mdi.MDI_DOUBLE, lammps)
    coords = np.array(coords) * bohr_to_angstrom
:::
```
````

For the purposes of this tutorial, we have included a utility function called `calculate_ANI_forces` that uses the information we have retrieved already and returns the forces calculated by TorchANI.
Let's take a look at the docstring for this function. 
Open `util.py` and look at the `calculate_ANI_force` function.

```
def calculate_ANI_force(
    masses: Union[np.ndarray, list[float]], coordinates: np.ndarray, cell: np.ndarray
):
    """
    Calculate the ANI2x force for system of atoms.

    Parameters
    ----------
    masses :
        An array or list containing the masses of the atoms.
    coordinates :
        An array containing the 3D coordinates of the atoms.
    cell :
        A 3x3 array representing the periodic boundary conditions of the simulation cell.

    Returns
    -------
    np.ndarray
        A flattened array of forces acting on each atom, reshaped to be 1-dimensional.
    """
```

This function takes the masses, coordinates, and cell vectors as input and returns the forces calculated by TorchANI. 

Here, we are passing in the coordinates, box vectors, and masses that we have retrieved from LAMMPS.

The forces returned by TorchANI are Hartree/Angstrom.
MDI requires atomic units (Hartree/Bohr), so we divide by the conversion factor `bohr_to_angstrom` to convert the forces.

````{admonition} Exercise - Forces
:class: exercise

Modify your loop so that you calculate the forces using TorchANI at each step.
Then, use MDI to send the forces to LAMMPS.
You can read about the `>FORCES` command using the new [API Documentation](https://janash.github.io/mdi-docs/api/mdi_standard/commands/data-exchange/FORCES.html).
The command to send the forces to LAMMPS is `>FORCES`.
**Be sure to check the units of the forces you are sending to LAMMPS.**
MDI expects atomic units (Ha/Bohr)

```{admonition} Solution
:class: solution dropdown

:::{code-block} python  

# Run the dynamics
for i in range(nsteps):
    
    # Send to @FORCES node
    mdi.MDI_Send_Command("@FORCES", lammps) 

    # Get the atomic coordinates
    mdi.MDI_Send_Command("<COORDS", lammps)
    coords = mdi.MDI_Recv(3 * natoms, mdi.MDI_DOUBLE, lammps)

    # Calculate the forces
    ani_forces = calculate_ANI_force(masses, coords, box_vects) / bohr_to_angstrom

    # Send the forces to LAMMPS
    mdi.MDI_Send_Command(">FORCES", lammps)
    mdi.MDI_Send(ani_forces, lammps)
:::
    
```     
    
````    

After you have completed this exercise, you have successfully driven a simulation using MDI and TorchANI.
You can increase your number of steps to run longer dynamics or modify the input file to change the simulation conditions.













