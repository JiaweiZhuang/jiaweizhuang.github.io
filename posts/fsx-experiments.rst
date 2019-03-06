.. title: Experiments with AWS FSx for Lustre: I/O benchmark and Dask cluster deployment
.. slug: fsx-experiments
.. date: 2019-03-05 14:53:14 UTC-05:00
.. tags: AWS, Cloud, HPC, MPI, Spack, I/O, Dask, Xarray
.. category: 
.. link: 
.. description: 
.. type: text

AWS has recently launched an extremely interesting new service called `FSx for Lustre <https://aws.amazon.com/fsx/lustre/>`_. Since last week, FSx has also been integrated with the `AWS ParallelCluster <https://aws-parallelcluster.readthedocs.io>`_ framework [#pclus-fsx]_, so you can spin-up a Lustre-enabled HPC cluster with one-click. As a reminder for non-HPC people, `Lustre <http://lustre.org/>`_ is a well-known high-performance parallel file system, deployed on many world-class supercomputers [#nas-lustre]_. Previously, ParallelCluster (and its predecessor `CfnCluster <https://cfncluster.readthedocs.io>`_) only provided a `Network File System (NFS) <https://en.wikipedia.org/wiki/Network_File_System>`_, running on the master node and exported to all compute nodes. This often causes a serious bottleneck for I/O-intensive applications. A fully-managed Lustre service should largely solve such I/O problem. Furthermore, FSx is "deeply integrated with Amazon S3"[#fsx-s3]_. Considering the incredible amount of public datasets living in S3 [#s3-opendata]_, such integration can lead to endless interesting use cases.

As a new user of FSx, my natural questions are:

- Does it deliver the claimed performance?
- How easy can it be used for real-world applications?

To answer these questions, I did some initial experiments:

- Performed I/O benchmark with `IOR <https://github.com/hpc/ior>`_, the de-facto HPC I/O benchmark framework.
- Deployed a `Dask <https://github.com/dask/dask>`_ cluster on top of AWS HPC environment, and used it to process public Earth science data in S3 buckets.

The two experiments are independent so feel free to jump to either one.

**Disclaimer**: This is just a preliminary result and is by no means a rigorous evaluation. In particular, I have made no effort on fine-turning the performance. I am not affiliated with AWS (although I do have their generous funding), so please treat this as a third-party report.

.. contents::
.. section-numbering::

Cluster infrastructure
======================

The first step is to spin up an AWS ParallelCluster. See my `previous post <link://slug/aws-hpc-guide>`_ for a detailed walk-through.

AWS ParallelCluster configuration
---------------------------------

The :code:`~/.parallelcluster/config` file is almost the same as in previous post. The only thing new is the :code:`fsx` section. At the time of writing, only CentOS 7 is supported [#pcluster-ubuntu]_. 

.. code:: bash

    [cluster your-cluster-section-name]
    base_os = centos7
    ...
    fsx_settings = fs
    
    [fsx fs]
    shared_dir = /fsx
    storage_capacity = 14400

According to AWS [#fsx-performance]_, a 14,400 GB file system would deliver 2,880 MB/s throughput. You may use a much bigger size for production, but this size should be good for initial testing.

The above config will create an empty file system. A more interesting usage is to pre-import S3 buckets:

.. code:: bash

    [fsx fs]
    shared_dir = /fsx
    storage_capacity = 14400
    imported_file_chunk_size = 1024
    import_path = s3://era5-pds

Here I am using the ECMWF ERA5 Reanalysis data [#era5]_. The data format is NetCDF4. Other datasets would work similarly.

:code:`pcluster create` can take several minutes longer than without FSx, because provisioning a Lustre server probably involves heavy-lifting on the AWS side. Go to the "Amazon FSx" console to check the creation status.

After a successful launch, log in and run :code:`df -h` to make sure that Lustre is properly mounted to :code:`/fsx`. 

Testing integration with existing S3 bucket
-------------------------------------------

Objects in the bucket will appear as normal on-disk files.

.. code:: bash

    $ cd /fsx
    $ ls # displays objects in the S3 bucket
    ...
    
However, FSx only stores metadata:

.. code:: bash
    
    $ cd /fsx/2008/01/data # specific to ERA5 data
    $ ls -lh *   # the data appear to be big (~1 GB)
    -rwxr-xr-x 1 root root  988M Jul  4  2018 air_pressure_at_mean_sea_level.nc
    ...
    $ du -sh *  # but the actual content is super small.
    512	air_pressure_at_mean_sea_level.nc
    ...

The actual data will be pulled from S3 when accessed. For a NetCDF4 file, either :code:`ncdump -h` or :code:`h5ls` will display its basic contents and cause the entire file to be pulled from S3.

.. code:: bash

    $ ncdump -h air_pressure_at_mean_sea_level.nc  # `ncdump` is installable from `sudo yum install netcdf`, or from Spack, or from Conda
    ...
    $ du -sh *  # now much bigger
    962M	air_pressure_at_mean_sea_level.nc
    ...

.. note::
  
    If you get HDF5 error on Lustre, set :code:`export HDF5_USE_FILE_LOCKING=FALSE` [#hdf5-error]_.

I/O benchmark by IOR
====================

For general reference, see IOR's documentation: https://ior.readthedocs.io

Install IOR by Spack
--------------------

Configure Spack as in the previous post. Then, getting IOR is simply:

.. code:: bash

    $ spack install ior ^openmpi+pmi schedulers=slurm

IOR is also quite easy to install from source, outside of Spack.

Discover the :code:`ior` executable by:

.. code:: bash

    $ export PATH=$(spack location -i ior)/bin:$PATH

I/O benchmark result
--------------------

With two ``c5n.18xlarge`` compute nodes running, a multi-node, parallel write-read test can be done by:

.. code:: bash

    $ mkdir /fsx/ior_tempdir
    $ cd /fsx/ior_tempdir
    $ srun -N 2 --ntasks-per-node 36 ior -t 1m -b 16m -s 4 -F -C -e
    ...
    Max Write: 1632.01 MiB/sec (1711.28 MB/sec)
    Max Read:  1654.59 MiB/sec (1734.96 MB/sec)
    ...

Conducting a proper I/O benchmark is not straightforward, due to various caching effects. IOR implements several tricks (reflected in command line parameters) to get around those effects [#ior-tutorial]_.

I can get maximum throughput with 8 client nodes:

.. code:: bash

    $ srun -N 8 --ntasks-per-node 36 ior -t 1m -b 16m -s 4 -F -C -e
    ...
    Max Write: 2905.59 MiB/sec (3046.73 MB/sec)
    Max Read:  2879.96 MiB/sec (3019.85 MB/sec)
    ...

This matches the 2,880 MB/s claimed by AWS! Using more nodes shows marginal improvement, since the bandwidth should already be saturated.

The logical next step is to test IO-heavy HPC applications and conduct a detailed I/O-profiling. In this post, however, I decide to try a more interesting use case -- big data analytics.

Dask cluster on top of AWS HPC stack
====================================

The entire idea comes from the Pangeo project (http://pangeo.io) that aims to develop a big-data geoscience platform on HPC and cloud. At its core, Pangeo relies on two excellent Python libraries:

- Xarray (http://xarray.pydata.org), which is probably the best way to handle NetCDF files and many other data formats in geoscience. It is also used as a general-purpose "multi-dimensional Pandas" outside of geoscience.
- Dask (https://dask.org), a parallel computing library that can scale NumPy, Pandas, Xarray, and Scikit-Learn to parallel and distributed environments. In particular, `Dask-distributed <https://distributed.dask.org>`_ handles distributed computing.

The normal way to deploy Pangeo on cloud is via `Dask-Kubernetes <http://kubernetes.dask.org>`_, leveraging fully-managed Kubernetes services like:

- `Google Kubernetes Engine <https://cloud.google.com/kubernetes-engine/>`_
- `Amazon Elastic Container Service for Kubernetes (EKS) <https://aws.amazon.com/eks/>`_
- `Azure Kubernetes Service <https://azure.microsoft.com/en-us/services/kubernetes-service/>`_

On the other hand, the deployment of Pangeo on local HPC clusters is through `Dask-Jobqueue <https://jobqueue.dask.org>`_ [#pangeo-hpc]_.

Since we already have a fully-fledged HPC cluster (contains Slurm + MPI + Lustre), there is no reason not to test the second approach. Is AWS now a cloud platform or an HPC cluster? The boundary seems to be blurred.

Cluster deployment with Dask-jobqueue
-------------------------------------

The deployment turns out to be extremely easy. I am still in the learning curve of Kubernetes, and this alternative HPC approach feels much more straightforward for an HPC person like me.

First, get Miniconda:

.. code:: bash

    $ cd /shared
    $ wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh
    $ bash miniconda.sh -b -p miniconda
    $ echo ". /shared/miniconda/etc/profile.d/conda.sh" >> ~/.bashrc
    $ source ~/.bashrc 
    $ conda create -n py37 python=3.7
    $ conda activate py37  # replaces `source activate` for conda>=4.4
    $ conda install -c conda-forge xarray netCDF4 cartopy dask-jobqueue jupyter

Optionally, install additional visualization libraries that I will use later:

.. code:: bash

    $ pip install geoviews hvplot datashader
  
.. note::
  
    It turns out that we don't need to install MPI4Py! Dask-jobqueue only needs a scheduler (here we have Slurm) to launch processes, and uses its own communication mechanism (defaults to TCP) [#dask-hpc]_.

With two idle :code:`c5n.18xlarge` nodes, use the following code in :code:`ipython` to initialize a distributed cluster:

.. code:: python

    from dask_jobqueue import SLURMCluster
    cluster = SLURMCluster(cores=72, processes=36, memory='150GB')  # Slurm thinks there are 72 cores per node due to EC2 hyperthreading
    cluster.scale(2*36)

    from distributed import Client
    client = Client(cluster)

In a separate shell, use :code:`sinfo` to check the node status -- they should be fully allocated.

To enable Dask's dashboard [#dask-dashboard]_, add an additional SSH connection in a new shell:

.. code::

    $ pcluster ssh your-cluster-name -N -L 8787:localhost:8787

Visit :code:`localhost:8787` in the web browser (NOT something like :code:`http://172.31.5.224:8787` shown in Python) .

Alternatively, everything can be put together, including Jupyter notebook's port-forwarding:

.. code:: bash

    $ pcluster ssh your-cluster-name -L 8889:localhost:8889 -L 8787:localhost:8787
    $ conda activate py37
    $ jupyter notebook --NotebookApp.token='' --no-browser --port=8889

Visit :code:`localhost:8889` to use the notebook.

That's all about the deployment! This Dask cluster is able to perform parallel read/write with the Lustre file system.

Computation with Dask-distributed
---------------------------------

As an example, I compute the average Sea Surface Temperature (SST) [#sst]_ over near 300 GBs of ERA5 data. It gets done in 15 seconds with 8 compute nodes, which would have taken > 20 minutes with a single small node. Here's the screen recording of Dask dashboard during computation.

.. vimeo:: 321645143
   :height: 400
   :width: 800

The full code is available in the `next notebook <link://slug/dask-hpc-fsx>`_, with some technical comments. At the end of the notebook also shows a sign of climate change (computed from the SST data), so at least we get a bit scientific insight from this toy problem. Hopefully such great computing power can be used to solve some big science.

Final thoughts
==============

Back to my initial questions:

- Does it deliver the claimed performance? Yes, and very accurately, at least for the moderate size I tried. A larger-scale benchmark is to TBD though.
- How easy can it be used for real-world applications? It turns out to be quite easy. All building blocks are already there, and I just need to put them together. It took me one day to get such initial tests done.

This HPC approach might be an alternative way of deploying the Pangeo big data stack on AWS. Some differences from the Kubernetes + pure S3 way are:

- No need to worry about the HDF + Cloud problem [#hdf-cloud]_. People can now access data in S3 through a POSIX-compliant, high-performance file system interface. This seems a big deal because huge amounts of data are already in HDF & NetCDF formats, and converting them to more a cloud-friendly format like Zarr might take some effort.
- It is probably easier for existing cloud-HPC users to adopt. Numerical simulations and post-processing can be done in exactly the same environment.
- It is likely to cost more (haven't rigorously calculated), due to heavier resource provisioning. Lustre essentially acts as a huge cache for S3. In the long-term, this kind of data analytics workflow should probably be handled in a more cloud-native way, using Lambda-like serverless computing, to maximize resource utilization and minimize computational cost. But it is nice to have something that "just works" right now.

Some possible further steps:
 
- The performance can be fine-tuned indefinitely. There is an extremely-large parameter space: Lustre stripe size, HDF5 chunk size, Dask chunk size, Dask processes vs threads, client instance counts and types... But unless there are important scientific/business needs, fine-tuning it doesn't seem super interesting.
- For me personally, this provides a very convenient test environment for scaling-out xESMF [#xesmf-pangeo]_, the regridding package I wrote. Because the entire pipeline is clearly I/O-limited, what I really need is just a fast file system.
- The most promising use case is probably some deep-learning-like climate analytics [#climate-net]_. DL algorithms are generally data hungry, and the best place to put massive datasets is, with not doubt, the cloud. How Dask + Xarray + Pangeo fit into DL workflow seems to be in active discussion [#xarray-dl]_ .

References
==========
.. [#pclus-fsx] Added in ParallelCluster v2.2.1 https://github.com/aws/aws-parallelcluster/releases/tag/v2.2.1. See FSx section in the docs: https://aws-parallelcluster.readthedocs.io/en/latest/configuration.html#fsx
.. [#nas-lustre] For example, NASA's supercomputing facility provides a nice user guide on Lustre: https://www.nas.nasa.gov/hecc/support/kb/102/
.. [#fsx-s3] See "Using S3 Data Repositories" in FSx guide: https://docs.aws.amazon.com/fsx/latest/LustreGuide/fsx-data-repositories.html
.. [#s3-opendata] See the Registry of Open Data on AWS https://registry.opendata.aws/. A large fraction of them are Earth data: https://aws.amazon.com/earth/.
.. [#pcluster-ubuntu] See this issue: https://github.com/aws/aws-parallelcluster/issues/896
.. [#fsx-performance] See Amazon FSx for Lustre Performance at https://docs.aws.amazon.com/fsx/latest/LustreGuide/performance.html
.. [#era5] Search for "ECMWF ERA5 Reanalysis" in the Registry of Open Data on AWS: https://registry.opendata.aws/ecmwf-era5. As a reminder for non-atmospheric people, a reanalysis is like the best guess of past atmospheric states, obtained from observations and simulations. For a more detailed but non-technical introduction, Read *Reanalyses and Observations: What’s the Difference?* at https://journals.ametsoc.org/doi/full/10.1175/BAMS-D-14-00226.1
.. [#hdf5-error] https://stackoverflow.com/questions/49317927/errno-101-netcdf-hdf-error-when-opening-netcdf-file
.. [#ior-tutorial] See "First Steps with IOR" at: https://ior.readthedocs.io/en/latest/userDoc/tutorial.html
.. [#pangeo-hpc] See "Getting Started with Pangeo on HPC": https://pangeo.readthedocs.io/en/latest/setup_guides/hpc.html
.. [#dask-hpc] See the "High Performance Computers" section in Dask docs: http://docs.dask.org/en/latest/setup/hpc.html
.. [#dask-dashboard] See "Viewing the Dask Dashboard" in Dask-Jobqueue docs: https://jobqueue.dask.org/en/latest/interactive.html#viewing-the-dask-dashboard
.. [#sst] SST is an important climate change indicator: https://www.epa.gov/climate-indicators/climate-change-indicators-sea-surface-temperature
.. [#hdf-cloud] HDF in the Cloud: challenges and solutions for scientific data: http://matthewrocklin.com/blog/work/2018/02/06/hdf-in-the-cloud
.. [#xesmf-pangeo] Initial tests regarding distributed regridding with xESMF on Pangeo: https://github.com/pangeo-data/pangeo/issues/334
.. [#climate-net] For example, see Berkeley Lab's ClimateNet: https://cs.lbl.gov/news-media/news/2019/climatenet-aims-to-improve-machine-learning-applications-in-climate-science-on-a-global-scale/
.. [#xarray-dl] See the discusson in this issue: https://github.com/pangeo-data/pangeo/issues/567