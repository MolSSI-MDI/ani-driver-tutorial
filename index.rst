.. molssi_doc_theme documentation master file, created by
   sphinx-quickstart on Thu Mar 15 13:55:56 2018.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.


MolSSI Driver Interface TorchANI Tutorial
=========================================================

.. image:: _static/mdi-dark.png
    :width: 600px
    :align: center
    :class: only-dark

.. image:: _static/mdi-light.png
    :width: 600px
    :align: center
    :class: only-light

The MolSSI Driver Interface (MDI) project provides a standardized API for fast, on-the-fly communication between computational chemistry codes.
Traditionally, enabling communication between various computational chemistry codes has required a cumbersome process of writing and reading files to disk. 
The MolSSI Driver Interface (MDI) project offers a streamlined API that facilitates direct, on-the-fly communication between different programs. 
With MDI, researchers can now seamlessly link multiple computational tools using a Python or C++ script, bypassing the need for intermediate files. 

This tutorial will walk you through creating an MDI Driver that uses [TorchANI]() and
[LAMMPS]() to perform a molecular dynamics simulation.

.. toctree::
   :maxdepth: 2
   :hidden:
   :titlesonly:

   setup
   mdimechanic
   energy_calculations
   driver_tutorial


