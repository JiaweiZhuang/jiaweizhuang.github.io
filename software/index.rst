.. title: Software
.. slug: software
.. date: 2019-02-17 22:40:46 UTC-05:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text

I am actively involved in open-source (scientific) software development. Most projects can be found on `my GitHub page <https://github.com/JiaweiZhuang>`_.

Software in Production
----------------------

Here are software applications used by at least hundreds of users for their research.

| **xESMF: Universal Regridder for Geospatial Data** (`link <https://github.com/JiaweiZhuang/xESMF>`__)
| `Regridding <https://climatedataguide.ucar.edu/climate-data-tools-and-analysis/regridding-overview>`_ is `one of the most important needs <http://www.ncl.ucar.edu/Document/Pivot_to_Python/NCL_and_Python_Survey_Report_2018.pdf>`_ in Atmospheric and Climate Data Analysis. xESMF combines the battle-tested `ESMF <https://www.earthsystemcog.org/projects/esmf/>`_ and modern scientific Python stacks like `Xarray <http://xarray.pydata.org/en/stable/>`_ and `Dask <https://dask.org/>`_ to provide fast and easy-to-use regridding functionalities. I am also interested in how it can be integrated with the `Pangeo <https://pangeo.io/>`_ big-data stack.
|

| **GEOS-Chem: Global Model of Atmospheric Chemistry and Composition** (`link <https://github.com/geoschem>`_)
| GEOS-Chem is a numerical model for atmospheric composition, `supported by NASA <https://map.nasa.gov/models/GEOS-Chem.php>`_ and `managed at Harvard <http://acmg.seas.harvard.edu/geos/>`_. It is used by `hundreds of groups <http://acmg.seas.harvard.edu/geos/geos_people.html>`_ worldwide and has been developed over 20+ years. My major contributions include: (1) Porting the original OpenMP-only code to MPI for multi-node parallelization. Unlike writing MPI code from scratch, refactoring an existing giant code base (~1 million lines of source code) imposes unique software engineering challenges. The new code uses the ESMF_ software framework on top of the original code, to minimize the changes to the original code base. (2) Bringing the model to the Amazon Web Services (AWS) cloud, including a cloud-HPC environment with ~1000 CPU cores.
|

Software in Prototype
---------------------

They are typically for class projects or just for fun. They can be useful prototypes for building more serious software applications.

| **Cubed-Sphere Grid Visualization** (`web <http://acmg.seas.harvard.edu/geos/cubed_sphere.html>`__, `code <https://github.com/JiaweiZhuang/Plotly_CubedSphere>`__)
| The `Cubed-Sphere grid <https://www.weather.gov/media/sti/nggps/Putman_Lin_Finite_Volume_Cubed_Sphere_Grid_JCompPhys_2007.pdf>`_ is getting increasingly popular in atmospheric modeling, in part due to its excellent scalability on massively parallel architecture. For example, it is used by `NOAA's next generation global weather prediction system <https://www.weather.gov/news/fv3>`_. Interestingly, such grid is also used for `extending convolutional neural networks to spherical geometries <https://papers.nips.cc/paper/6935-spherical-convolutions-and-their-application-in-molecular-modelling>`_. Unlike the well-known latitude-longitude grid, Cubed-Sphere can be a bit unintuitive at first glance (although it is actually super intuitive!). A good visualization can let people understand the grid geometry quickly without dividing into the math. The code uses `Plotly Python API <https://plot.ly/python/>`_ to generate JavaScript that runs in user's browser.
|

| **Cubed-Sphere Data Processing** (`link <https://github.com/JiaweiZhuang/cubedsphere>`__)
| A tiny package to facilitate the processing of cubed-sphere data with Xarray. It implements `a fast weighted-binning algorithm <https://gist.github.com/JiaweiZhuang/798a05b7c0bdc6ea81017a53cb76ac18>`_ for taking zonal mean over curvilinear grids, 10x faster than the reference implementation. It is used in `one of my papers <https://www.atmos-chem-phys.net/18/6039/2018/acp-18-6039-2018.pdf>`_.
|

| **2D High-order Advection Solver in TensorFlow** (`link <https://github.com/JiaweiZhuang/advection_solver/tree/vectorization>`__)
| No one prevents you from using neural network libraries like TensorFlow & PyTorch for general-purpose scientific computing :) This can be a quick way to make code running on GPU/TPU. The code implements `a second-order flux-limited scheme <https://journals.ametsoc.org/doi/abs/10.1175/1520-0493%281994%29122%3C1575%3AACOTVL%3E2.0.CO%3B2>`_ using `TensorFlow Eager Execution <https://www.tensorflow.org/guide/eager>`_.
|

| **Neural Network ODE Solver** (`link <https://github.com/JiaweiZhuang/AM205_final>`__)
| The code implements a very old and simple idea that the solution to an Ordinary Differential Equation (ODE) can be parameterized by a neural network and "solved" by minimization a loss function. It uses `HIPS-autograd <https://github.com/HIPS/autograd>`_ to perform automatic differentiation on NumPy functions. (It is NOT about the fancy new paper on `Neural ODEs <https://arxiv.org/abs/1806.07366>`_!)
|

| **Parallel K-means** (`link <https://github.com/JiaweiZhuang/CS205_final_project>`__)
| K-means is a simple and highly-parallelizable unsupervised clustering algorithm. The code parallelizes K-means by OpenMP, MPI, hybrid OpenMP-MPI, and CUDA C.
|
