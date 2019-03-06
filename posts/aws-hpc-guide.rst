.. title: A scientist's guide to cloud-HPC: example with AWS ParallelCluster, Slurm, Spack, and WRF
.. slug: aws-hpc-guide
.. date: 2019-03-01 14:35:25 UTC-05:00
.. tags: AWS, Cloud, HPC, MPI, Spack, WRF
.. category: 
.. link: 
.. description: 
.. type: text

.. contents::
.. section-numbering::

Motivation and principle of this guide
======================================

Cloud-HPC is growing rapidly [#cloud-hpc-growth]_, and the growth can only be faster with AWS's recent HPC-oriented services such as `EC2 c5n <https://aws.amazon.com/about-aws/whats-new/2018/11/introducing-amazon-ec2-c5n-instances/>`_, `FSx for Lustre <https://aws.amazon.com/fsx/lustre/>`_, and the soon-coming `EFA <https://aws.amazon.com/about-aws/whats-new/2018/11/introducing-elastic-fabric-adapter/>`_. However, orchestrating a cloud-HPC cluster is by no means easy, especially considering that many HPC users are from science and engineering and are not trained with IT and system administration skills. There are very few documentations for this niche field, and users could face a pretty steep learning curve. To make people's lives a bit easier (and to provide a reference for the future me), I wrote this guide to show an easy-to-follow workflow of building a fully-fledged HPC cluster environment on AWS. 

My basic principles are:

- Minimize the learning curve for non-IT/non-CS people. That being said, it can still take a while for new users to learn. But you should be able to use a cloud-HPC cluster with confidence after going through this guide.
- Focus on common, general, transferrable cases. I would avoid diving into a particular scientific field, or into a niche AWS utility with no counterparts on other cloud platforms -- those can be left to other posts, but do not belong to this guide.

This guide will go through:

- Spin-up an HPC cluster with `AWS ParallelCluster <https://github.com/aws/aws-parallelcluster>`_, AWS's official HPC framework. If you prefer a multi-platform, general-purpose tool, consider `ElastiCluster <https://github.com/gc3-uzh-ch/elasticluster>`_, but expect a steeper learning curve and less up-to-date AWS features. If you feel that all those frameworks are too black-boxy, try building a cluster manually [#manual-cluster]_ to understand how multiple nodes are glued together. The manual approach becomes quite inconvenient at production, so you will be much better off by using a higher-level framework.
- Basic cluster operations with `Slurm <https://github.com/SchedMD/slurm>`_, an open-source, modern job scheduler deployed on many HPC centers. ParallelCluster can also use `AWS Batch <https://aws.amazon.com/batch/>`_ instead of Slurm as the scheduler; it is a very interesting feature but I will not cover it here.
- Common cluster management tricks such as changing the node number and type on the fly.
- Install HPC software stack by `Spack <https://github.com/spack/spack>`_, an open-source, modern HPC package manager used in production at many HPC centers. This part should also work for other cloud platforms, on your own workstation, or in a container.
- Build real-world HPC code. As an example I will use the `Weather Research and Forecasting (WRF) Model <https://github.com/wrf-model/WRF>`_, an open-source, well-known atmospheric model. This is just to demonstrate that getting real applications running is relatively straightforward. Adapt it for your own use cases.

Prerequisites
=============

This guide only assumes:

- **Basic EC2 knowledge.** Knowing how to use a single instance is good enough. Thanks to the wide-spreading ML/DL hype, this seems to become a common skill for science & engineering students -- most people in my department (non-CS) know how to use `AWS DL AMI <https://aws.amazon.com/machine-learning/amis/>`_, `Google DL VM <https://cloud.google.com/deep-learning-vm/>`_ or `Azure DS VM <https://azure.microsoft.com/en-us/services/virtual-machines/data-science-virtual-machines/>`_. If not, the easiest way to learn it is probably through online DL courses [#dl-course]_.
- **Basic S3 knowledge.** Knowing how to ``aws s3 cp`` is good enough. If not, check out `AWS's 10-min tutorial <https://aws.amazon.com/getting-started/tutorials/backup-to-s3-cli/>`_.
- **Entry-level HPC user knowledge.** Knowing how to submit MPI jobs is good enough. If not, checkout HPC carpentry [#hpc-carpentry]_.

It does NOT require the knowledge of:

- `CloudFormation <https://aws.amazon.com/cloudformation/>`_. It is the underlying framework for AWS ParallelCluster (and many third-party tools), but can take quite a while to learn.
- Cloud networking. You can use the cluster smoothly even without knowing what TCP is.
- How to build complicated libraries from source -- this will be handled by Spack.

Cluster deployment
==================

This section uses ParallelCluster version 2.2.1 as of Mar 2019. Future versions shouldn't be vastly different.

First, check out ParallelCluster's official doc: https://aws-parallelcluster.readthedocs.io. It guides you through some toy examples, but not production-ready applications. Play with the toy examples a bit and get familiar with those basic commands:

- ``pcluster configure``
- ``pcluster create``
- ``pcluster list``
- ``pcluster ssh``
- ``pcluster delete``

A minimum config file for production HPC
----------------------------------------

The cluster infrastructure is fully specified by :code:`~/.parallelcluster/config`. A minimum, recommended config file would look like:

.. code:: bash

    [aws]
    aws_region_name = xxx

    [cluster your-cluster-section-name]
    key_name = xxx
    base_os = centos7
    master_instance_type = c5n.large
    compute_instance_type = c5n.18xlarge
    cluster_type = spot
    initial_queue_size = 2
    scheduler = slurm
    placement_group = DYNAMIC
    vpc_settings = your-vpc-section-name
    ebs_settings = your-ebs-section-name

    [vpc your-vpc-section-name]
    vpc_id = vpc-xxxxxxxx
    master_subnet_id = subnet-xxxxxxxx

    [ebs your-ebs-section-name]
    shared_dir = shared
    volume_type = st1
    volume_size = 500

    [global]
    cluster_template = your-cluster-section-name
    update_check = true
    sanity_check = true

A brief comment on what are set:

- :code:`aws_region_name` should be set at initial :code:`pcluster configure`. I use :code:`us-east-1`.
- :code:`key_name` is your EC2 key-pair name, for :code:`ssh` to master instance.
- :code:`base_os = centos7` should be a good choice for HPC, because CentOS is particularly tolerant of legacy HPC code. Some code that doesn't build on Ubuntu can actually pass on CentOS. Without build problems, any OS choice should be fine -- you shouldn't observe visible performance difference across different OS, as long as the compilers are the same.
- Use the biggest compute node :code:`c5n.18xlarge` to minimize communication. Master node is less critical for performance and is totally up to you.
- :code:`cluster_type = spot` will save you a lot of money by using spot instances for compute nodes.
- :code:`initial_queue_size = 2` spins up two compute nodes at initial launch. This is default but worth emphasizing. Sometimes there is not enough compute capacity in a zone, and with :code:`initial_queue_size = 0` you won't be able to detect that at cluster creation.
- Set :code:`scheduler = slurm` as we are going to use it in later sections.
- :code:`placement_group = DYNAMIC` creates a placement group [#placement-group]_ on the fly so you don't need to create one yourself. Simply put, a cluster placement group improves inter-node connection.
- :code:`vpc_id` and :code:`master_subnet_id` should be set at initial :code:`pcluster configure`. Because a subnet id is tied to an avail zone [#avail-zone]_, the subnet option implicitly specifies which avail zone your cluster will be launched into. You may want to change it because the spot pricing and capacity vary across avail zones.
- :code:`volume_type = st1` specifies throughput-optimized HDD [#ebs-type]_ as shared disk. The minimum size is 500 GB. It will be mounted to a directory :code:`/shared` (which is also default) and will be visible to all nodes.
- :code:`cluster_template` allows you to put multiple cluster configurations in a single config file and easily switch between them.

Credential information like :code:`aws_access_key_id` can be omitted, as it will default to awscli credentials stored in :code:`~/.aws/credentials`.

The full list of parameters are available in the official docs [#pcluster-config]_. Other useful parameters you may consider changing are:

- Set :code:`placement = cluster` to also put your master node in the placement group.
- Specify :code:`s3_read_write_resource` so you can access that S3 bucket without configuring AWS credentials on the cluster. Useful for archiving data.
- Increase :code:`master_root_volume_size` and :code:`compute_root_volume_size`, if your code involves heavy local disk I/O.
- :code:`max_queue_size` and :code:`maintain_initial_size` are less critical as they can be easily changed later.

I have omitted the FSx section, which is left to the `next post <link://slug/fsx-experiments>`_.

One last thing: Many HPC code runs faster with hyperthreading disabled [#hyper]_. To achieve this at launch, you can write a custom script and execute it via the ``post_install`` option in pcluster's config file. This is a bit involved though. Hopefully there can be a simple option in future versions of pcluster.

With the config file in place, run :code:`pcluster create your-cluster-name` to launch a cluster.

What are actually deployed
--------------------------

(This part is not required for first-time users. It just helps understanding.)

AWS ParallelCluster (or other third-party cluster tools) glues many AWS services together. While not required, a bit more understanding of the underlying components would be helpful -- especially when debugging and customizing things.

The official doc provides a conceptual overview [#pcluster-components]_. Here I give a more hands-on introduction by actually walking through the AWS console. When a cluster is running, you will see the following components in the console:

- **CloudFormation Stack.** Displayed under "Services" - "CloudFormation". This is the top-level framework that controls the rest. You shouldn't need to touch it, but its output can be useful for debugging.

.. image:: /images/pcluster_components/cloudformation.png
   :align: center
   :height: 150 pt

The rest of services are all displayed under the main EC2 console.

- **EC2 Placement Group.** It is created automatically because of the line :code:`placement_group = DYNAMIC` in the :code:`config` file.
 
.. image:: /images/pcluster_components/placement_group.png
   :align: center
   :height: 120 pt

- **EC2 Instances.** Here, there are one master node and two compute nodes running, as specified by the :code:`config` file. You can directly :code:`ssh` to the master node, but the compute nodes are only accessible from the master node, not from the Internet.

.. image:: /images/pcluster_components/ec2_instance.png
   :align: center
   :height: 150 pt

- **EC2 Auto Scaling Group.** Your compute instances belong to an Auto Scaling group [#autoscaling]_, which can quickly adjust the number of instances with minimum human operation. The number under the "Instances" column shows the current number of compute nodes; the "Desired" column shows the target number of nodes, and this number can be adjusted automatically by the Slurm scheduler; the "Min" column specifies the lower bound of nodes, which cannot be changed by the scheduler; the "Max" column corresponds to :code:`max_queue_size` in the config file. You can manually change the number of compute nodes here (more on this later).

.. image:: /images/pcluster_components/autoscaling.png
   :align: center
   :height: 120 pt

The launch event is recored in the "Activity History"; if a node fails to launch, the error message will go here.

.. image:: /images/pcluster_components/activity_history.png
   :align: center
   :height: 150 pt

- **EC2 Launch Template.** It specifies the EC2 instance configuration (like instance type and AMI) for the above Auto Scaling Group.

.. image:: /images/pcluster_components/launch_template.png
   :align: center
   :height: 120 pt

- **EC2 Spot Request.** With :code:`cluster_type = spot`, each compute node is associated with a spot request.

.. image:: /images/pcluster_components/spot_requests.png
   :align: center
   :height: 120 pt

- **EBS Volume.** You will see 3 kinds of volumes. A standalone volume specified in the :code:`ebs` section, a volume for master node, and a few volumes for compute nodes.

.. image:: /images/pcluster_components/ebs_volume.png
   :align: center
   :height: 120 pt

- **Auxiliary Services.** They are not directly related to the computation, but help gluing the major computing services together. For example, the cluster uses DynamoDB (Amazon's noSQL database) for storing some metadata. The cluster also relies on Amazon SNS and SQS for interaction between the Slurm scheduler and the AutoScaling group. We will see this in action later.

.. image:: /images/pcluster_components/dynamo_db.png
   :align: center
   :height: 120 pt

Imagine the workload involved if you launch all the above resources by hand and glue them together. Fortunately, as a user, there is no need to implement those from scratch. But it is good to know a bit about the underlying components.

In most cases, you should not manually modify those individual resources. For example, if you terminate a compute instance, a new one will be automatically launched to match the current autoscaling requirement. Let the high-level :code:`pcluster` command handle the cluster operation. Some exceptions will be mentioned in the "tricks" section later.


ParallelCluster basic operation
===============================

Using Slurm
-----------

After login to the master node with :code:`pcluster ssh`, you will use Slurm to interact with compute nodes. Here I summarize commonly-used commands. For general reference, see Slurm's documentation: https://www.schedmd.com/. 

Slurm is pre-installed at :code:`/opt/slurm/` :

.. code:: bash

    $ which sinfo
    /opt/slurm/bin/sinfo

Check compute node status:

.. code:: bash

    $ sinfo
    PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
    compute*     up   infinite      2   idle ip-172-31-3-187,ip-172-31-7-245

The :code:`172-31-xxx-xxx` is the Private IP [#private-ip]_ of the compute instances. The address range falls in your AWS VPC subnet. On EC2, :code:`hostname` prints the private IP:

.. code:: bash

    $ hostname  # private ip of master node
    ip-172-31-7-214

To execute commands on compute nodes, use :code:`srun`:

.. code:: bash

    $ srun -N 2 -n 2 hostname  # private ip of compute nodes
    ip-172-31-3-187
    ip-172-31-7-245

The printed IP should match the output of :code:`sinfo`.

You can go to a compute node with the standard Slurm command:

.. code:: bash

    $ srun -N 1 -n 72 --pty bash  # Slurm thinks a c5n.18xlarge node has 72 cores due to hyperthreading
    $ sinfo  # one node is fully allocated
    PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
    compute*     up   infinite      1  alloc ip-172-31-3-187
    compute*     up   infinite      1   idle ip-172-31-7-245

Or simply via :code:`ssh`:

.. code:: bash

    $ ssh ip-172-31-3-187
    $ sinfo  # still idle
    PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
    compute*     up   infinite      2   idle ip-172-31-3-187,ip-172-31-7-245

In this case, the scheduler is not aware of such activity.

The :code:`$HOME` directory is exported to all nodes via NFS by default, so you can still see the same files from compute nodes. However, system directories like :code:`/usr` are specific to each node. Software libraries should generally be installed to a shared disk, otherwise they will not be accessible from compute nodes.

Check system & hardware
-----------------------

A natural thing is to check CPU info with :code:`lscpu` and file system structure with :code:`df -h`. Do this on both master and compute nodes to see the differences.

A serious HPC user should also check the network interface:

.. code:: bash

    $ ifconfig  # display network interface names and details
    ens5: ...

    lo: ...

Here, the :code:`ens5` section is the network interface for inter-node commnunication. Its driver should be :code:`ena`:

.. code:: bash

    $ ethtool -i ens5
    driver: ena
    version: 1.5.0K

This means that "Enhanced Networking" is enabled [#ena]_. This should be the default on most modern AMIs, so you shouldn't need to change anything.

Cluster management tricks
-------------------------

AWS ParallelCluster is able to auto-scale [#pcluster-autoscaling]_, meaning that new compute nodes will be launched automatically when there are pending jobs in Slurm's queue, and idle nodes will be terminated automatically.

While this generally works fine, such automatic update takes a while and feels a bit black-boxy. A more straightforward & transparent way is to modify the autoscaling group directly in the console. Right-click on your AutoScaling Group, and select "Edit":

.. image:: /images/pcluster_components/edit_autoscaling.png
   :align: center
   :height: 80 pt

- Modifying "Desired Capacity" will immediately cause the cluster to adjust to that size. Either to request more nodes or to kill redundant nodes.
- Increase "Min" to match "Desired Capacity" if you want the compute nodes to keep running even if they are idle. Or keep "Min" as zero, so idle nodes will be killed after some time period (a few minutes, roughly match the "Default Cooldown" section in the Auto Scaling Group).
- "Max" must be at least the same as "Desired Capacity". This is the hard-limit that the scheduler cannot violate.

After compute nodes are launched or killed, Slurm should be aware of such change in ~1 minute. Check it with :code:`sinfo`.

To further change the type (not just the number) of the compute nodes, you can modify the :code:`config` file, and run :code:`pcluster update your-cluster-name` [#pcluster-update]_.

Install HPC software stack with Spack
=====================================

While you can get pre-built MPI binaries with :code:`sudo yum install -y openmpi-devel` on CentOS or :code:`sudo apt install -y libopenmpi-dev` on Ubuntu, they are generally not the specific version you want. On the other hand, building custom versions of libraries from source is too laborious and error-prone [#build-netcdf]_. Spack achieves a great balance between the ease-of-use and customizability. It has an excellent documentation which I strongly recommend reading: https://spack.readthedocs.io/.

Here I provide the minimum required steps to build a production-ready HPC environment.

Getting Spack is super easy:

.. code:: bash

    cd /shared  # install to shared disk
    git clone https://github.com/spack/spack.git
    echo 'export PATH=/shared/spack/bin:$PATH' >> ~/.bashrc  # to discover spack executable
    source ~/.bashrc

At the time of writing, I am using:

.. code:: bash

    $ spack --version
    0.12.1

The first thing is to check what compilers are available. Most OS should already have a GNU compiler installed, and Spack can discover it:

.. code:: bash

    $ spack compilers
    ==> Available compilers
    -- gcc centos7-x86_64 -------------------------------------------
    gcc@4.8.5

.. note::

    If not installed, just :code:`sudo yum install gcc gcc-gfortran gcc-c++` on CentOS or :code:`sudo apt install gcc gfortran g++` on Ubuntu.

You might want to get a newer version of the compiler:

.. code:: bash

    $ spack install gcc@8.2.0  # can take 30 min!
    $ spack compiler add $(spack location -i gcc@8.2.0)
    $ spack compilers
    ==> Available compilers
    -- gcc centos7-x86_64 -------------------------------------------
    gcc@8.2.0  gcc@4.8.5

.. note::
    
    Spack builds software from source, which can take a while. To persist the build you can run it inside :code:`tmux` sessions. If not installed, simply run :code:`sudo yum install tmux` or :code:`sudo apt install tmux`.

.. note::

    Always use :code:`spack spec` to check versions and dependencies before running :code:`spack install`!

MPI libraries (OpenMPI with Slurm support)
------------------------------------------

Spack can install many MPI implementations, for example:

.. code:: bash

    $ spack info mpich
    $ spack info mvapich2
    $ spack info openmpi

In this example I will use OpenMPI. It has a super-informative documentation at https://www.open-mpi.org/faq/

Installing OpenMPI
^^^^^^^^^^^^^^^^^^

In principle, the installation is as simple as:

.. code:: bash

    $ spack install openmpi  # not what we will use here
    
Or a specific version:

.. code:: bash

    $ spack install openmpi@3.1.3  # not what we will use here

However, we want OpenMPI to be built with Slurm [#ompi-slurm]_, so the launch of MPI processes can be handled by Slurm's scheduler.

Because Slurm is pre-installed, you will add it as an external package to Spack [#slurm-spack]_. 

.. code:: bash

    $ which sinfo  # comes with AWS ParallelCluster
    /opt/slurm/bin/sinfo
    $ sinfo -V
    slurm 16.05.3

Add the following section to :code:`~/.spack/packages.yaml`:

.. code:: bash

    packages:
      slurm:
        paths:
          slurm@16.05.3: /opt/slurm/
        buildable: False

**This step is extremely important. Without modifying packages.yaml, Spack will install Slurm for you, but the newly-installed Slurm is not configured with the AWS cluster.**

Then install OpenMPI wih:

.. code:: bash

    $ spack install openmpi+pmi schedulers=slurm  # use this

After installation, locate its directory:

.. code:: bash

    $ spack find -p openmpi

Modify :code:`$PATH` to discover executables like :code:`mpicc`:

.. code:: bash

    $ export PATH=$(spack location -i openmpi)/bin:$PATH

.. note::
    Spack removes the :code:`mpirun` executable by default if built with Slurm, to encourage the use of :code:`srun` for better process management [#remove-mpirun]_. I need :code:`mpirun` for illustration purpose in this guide, so recover it by :code:`ln -s orterun mpirun` in the directory :code:`$(spack location -i openmpi)/bin/`.

A serious HPC user should also check the available Byte Transfer Layer (BTL) in OpenMPI:

.. code:: bash

    $ ompi_info --param btl all
      MCA btl: self (MCA v2.1.0, API v3.0.0, Component v3.1.3)
      MCA btl: tcp (MCA v2.1.0, API v3.0.0, Component v3.1.3)
      MCA btl: vader (MCA v2.1.0, API v3.0.0, Component v3.1.3)
      ...

- :code:`self`, as its name suggests, is for a process to talk to itself [#ompi-self]_.
- :code:`tcp` is the default inter-node communication mechanism on EC2 [#ompi-tcp]_. It is not ideal for HPC, but this should be changed with the coming EFA_.
- :code:`vader` is a high-performance intra-node communication mechanism [#ompi-vader]_.

Using OpenMPI with Slurm
^^^^^^^^^^^^^^^^^^^^^^^^

Let's use this boring but useful "MPI hello world" example:

.. code:: C

    #include <mpi.h>
    #include <stdio.h>
    #include <unistd.h>

    int main(int argc, char *argv[])
    {
        int rank, size;
        char hostname[32];
        MPI_Init(&argc, &argv);

        MPI_Comm_rank(MPI_COMM_WORLD, &rank);
        MPI_Comm_size(MPI_COMM_WORLD, &size);
        gethostname(hostname, 31);

        printf("I am %d of %d, on host %s\n", rank, size, hostname);
        
        MPI_Finalize();
        return 0;
    }

Put it into a :code:`hello_mpi.c` file and compile:

.. code:: bash

    $ mpicc -o hello_mpi.x hello_mpi.c
    $ mpirun -np 1 ./hello_mpi.x  # runs on master node
    I am 0 of 1, on host ip-172-31-7-214

To run it on compute nodes, the classic MPI way is to specify the node list via :code:`--host` or :code:`--hostfile` (for OpenMPI; other MPI implementations have similar options):

.. code:: bash

    $ mpirun -np 2 --host ip-172-31-5-150,ip-172-31-14-243 ./hello_mpi.x
    I am 0 of 2, on host ip-172-31-5-150
    I am 1 of 2, on host ip-172-31-14-243

Following :code:`--host` are compute node IPs shown by :code:`sinfo`.

A more sane approach is to launch it via :code:`srun`, which takes care of the placement of MPI processes:

.. code:: bash

    $ srun -N 2 --ntasks-per-node 2 ./hello_mpi.x
    I am 1 of 4, on host ip-172-31-5-150
    I am 0 of 4, on host ip-172-31-5-150
    I am 3 of 4, on host ip-172-31-14-243
    I am 2 of 4, on host ip-172-31-14-243

HDF5 and NetCDF libraries
-------------------------

`HDF5 <https://www.hdfgroup.org/>`_ and `NetCDF <https://www.unidata.ucar.edu/software/netcdf/>`_ are very common I/O libraries for HPC, widely used in Earth science and many other fields.

In principle, installing HDF5 is simply:

.. code:: bash

    $ spack install hdf5  # not what we will use here

Many HPC code (like WRF) needs the full HDF5 suite (use :code:`spack info` to check all the variants):

.. code:: bash

    $ spack install hdf5+fortran+hl  # not what we will use here

Further specify MPI dependencies:

.. code:: bash

    $ spack install hdf5+fortran+hl ^openmpi+pmi schedulers=slurm  # use this

Similarly, for NetCDF C & Fortran, in principle it is simply:

.. code:: bash

    $ spack install netcdf-fortran  # not what we will use here

To specify the full dependency, we end up having:

.. code:: bash

    $ spack install netcdf-fortran ^hdf5+fortran+hl ^openmpi+pmi schedulers=slurm  # use this

Further reading on advanced package management
----------------------------------------------

For HPC development you generally need to test many combinations of libraries. To better organize multiple environments, check out:

- :code:`spack env` and :code:`spack.yaml` at: https://spack.readthedocs.io/en/latest/tutorial_environments.html. For Python users, this is like :code:`virtualenv` or :code:`conda env`.
- Integration with :code:`module` at: https://spack.readthedocs.io/en/latest/tutorial_modules.html. This should be a familiar utility for existing HPC users.

Note on reusing software installation
-------------------------------------

For a single EC2 instance, it is easy to save the environment - create an AMI, or just build a Docker image. Things get quite cumbersome with a multi-node cluster environment. From official docs, "Building a custom AMI is not the recommended approach for customizing AWS ParallelCluster." [#pcluster-ami]_. 

Fortunately, Spack installs everything to a single, non-root directory (similar to Anaconda), so you can simply tar-ball the entire directory and then upload to S3 or other persistent storage:

.. code:: bash

    $ spack clean --all  # clean all kinds of caches
    $ tar zcvf spack.tar.gz spack  # compression
    $ aws s3 mb [your-bucket-name]  # create a new bucket. might need to configure AWS credentials for permission
    $ aws s3 cp spack.tar.gz s3://[your-bucket-name]/   # upload to S3 bucket

Also remember to save (and later recover) your custom settings in :code:`~/.spack/packages.yaml`, :code:`~/.spack/linux/compilers.yaml` and :code:`.bashrc`.

Then you can safely delete the cluster. For the next time, simply pull the tar-ball from S3 and decompress it. The environment would look exactly the same as the last time. You should use the same :code:`base_os` to minimize binary-compatibility errors.

A minor issue is regarding dynamic linking. When re-creating the cluster environment, make sure that the :code:`spack/` directory is located at the same location where the package was installed last time. For example, if it was at :code:`/shared/spack/`, then the new location should also be exactly :code:`/shared/spack/`.

The underlying reason is that Spack uses `RPATH <https://en.wikipedia.org/wiki/Rpath>`_ for library dependencies, to avoid messing around :code:`$LD_LIBRARY_PATH` [#spack-rapth]_. Simply put, it hard-codes the dependencies into the binary. You can check the hard-coded paths by, for example:

.. code:: bash

    $ readelf -d $(spack location -i openmpi)/bin/mpicc | grep RPATH

If the new shared EBS volume is mounted to a new location like :code:`/shared_new`, a quick-and-dirty fix would be:

.. code:: bash

    $ sudo ln -s /shared_new /shared/

Special note on Intel compilers
-------------------------------

Although I'd like to stick with open-source software, sometimes there is a solid reason to use proprietary ones like the Intel compiler -- WRF being a well-known example that runs much faster with ``ifort`` than with ``gfortran`` [#wrf-benchmark]_. Note that Intel is generous enough to `provide student licenses for free <https://software.intel.com/en-us/qualify-for-free-software/student>`_.

Although Spack can install Intel compilers by itself, a more robust approach is to install it externally and add as an external package [#spack-intel]_. Intel has a dedicated guide for installation on EC2 [#intel-aws]_ so I won't repeat the steps here.

Once you have a working :code:`icc`/:code:`ifort` in :code:`$PATH`, just running :code:`spack compiler add` should discover the new compilers [#spack-external]_.

Then, you should also add something like

.. code:: bash 

    extra_rpaths: ['/shared/intel/lib/intel64']

to :code:`~/.spack/linux/compilers.yaml` under the Intel compiler section. Otherwise you will see interesting linking errors when later building libraries with the Intel compiler [#intel-rpath]_.

After those are all set, simple add :code:`%intel` to all :code:`spack install` commands to build new libraries with Intel.

Build real applications -- example with WRF
===========================================

With common libraries like MPI, HDF5, and NetCDF installed, compiling real applications shouldn't be difficult. Here I show how to build WRF, a household name in the Atmospheric Science community. We will hit a few small issues (as likely for other HPC code), but they are all easy to fix by just Googling the error messages.

Get the recently released WRF v4::

    $ wget https://github.com/wrf-model/WRF/archive/v4.0.3.tar.gz
    $ tar zxvf v4.0.3.tar.gz

Here I only provide the minimum steps to build the WRF model, without diving into the actual model usage. If you plan to use WRF for either research or operation, please carefully study:

- The official user guide: http://www2.mmm.ucar.edu/wrf/users/
- A user-friendly tutorial: http://www2.mmm.ucar.edu/wrf/OnLineTutorial/index.php

Environment setup
-----------------

Add those to your :code:`~/.bashrc` (adapted from the `WRF compile tutorial <http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compilation_tutorial.php>`_):

.. code:: bash

    # Let WRF discover necessary executables
    export PATH=$(spack location -i gcc)/bin:$PATH  # only needed if you installed a new gcc
    export PATH=$(spack location -i openmpi)/bin:$PATH
    export PATH=$(spack location -i netcdf)/bin:$PATH
    export PATH=$(spack location -i netcdf-fortran)/bin:$PATH

    # Environment variables required by WRF
    export HDF5=$(spack location -i hdf5)
    export NETCDF=$(spack location -i netcdf-fortran)

    # run-time linking
    export LD_LIBRARY_PATH=$HDF5/lib:$NETCDF/lib:$LD_LIBRARY_PATH

    # this prevents segmentation fault when running the model
    ulimit -s unlimited  

    # WRF-specific settings
    export WRF_EM_CORE=1
    export WRFIO_NCD_NO_LARGE_FILE_SUPPORT=0

WRF also requires NetCDF-C and NetCDF-Fortran to be located in the same directory [#wrf-netcdf]_. A quick-and-dirty fix is to copy NetCDF-C libraries and headers to NetCDF-Fortran's directory: 

.. code:: bash

    $ NETCDF_C=$(spack location -i netcdf)
    $ ln -sf $NETCDF_C/include/*  $NETCDF/include/
    $ ln -sf $NETCDF_C/lib/*  $NETCDF/lib/

Compile WRF
-----------

.. code:: bash

    $ cd WRF-4.0.3
    $ ./configure

- For the first question, select ``34``, which uses GNU compilers and pure MPI ("dmpar" -- Distributed Memory PARallelization).
- For the second question, select ``1``, whichs uses basic nesting.

You should get this successful message::

    (omitting many lines...)
    ------------------------------------------------------------------------
    Settings listed above are written to configure.wrf.
    If you wish to change settings, please edit that file.
    If you wish to change the default options, edit the file:
        arch/configure.defaults


    Testing for NetCDF, C and Fortran compiler

    This installation of NetCDF is 64-bit
                    C compiler is 64-bit
            Fortran compiler is 64-bit
                It will build in 64-bit

    *****************************************************************************
    This build of WRF will use NETCDF4 with HDF5 compression
    *****************************************************************************

To fix a minor issue regarding WRF + GNU + OpenMPI [#wrf-openmpi]_, modify the generated :code:`configure.wrf` so that:

.. code:: bash

    DM_CC = mpicc -DMPI2_SUPPORT

Then build the WRF executable for the commonly used :code:`em_real` case:

.. code:: bash

    $ ./compile em_real 2>&1 | tee wrf_compile.log

You might also use a bigger master node (or go to a compute node) and add something like :code:`-j 8` for parallel build.

It should finally succeed::

    (omitting many lines...)
    ==========================================================================
    build started:   Mon Mar  4 01:32:52 UTC 2019
    build completed: Mon Mar 4 01:41:36 UTC 2019

    --->                  Executables successfully built                  <---

    -rwxrwxr-x 1 centos centos 41979152 Mar  4 01:41 main/ndown.exe
    -rwxrwxr-x 1 centos centos 41852072 Mar  4 01:41 main/real.exe
    -rwxrwxr-x 1 centos centos 41381488 Mar  4 01:41 main/tc.exe
    -rwxrwxr-x 1 centos centos 45549368 Mar  4 01:41 main/wrf.exe

    ==========================================================================

Now you have the WRF executables. This is a good first step, considering that so many people are stuck at simply getting the code compiled [#wrf-error]_. Actually using WRF for research or operational purposes requires a lot more steps and domain expertise, which is way beyond this guide. You will also need to build the WRF Preprocessing System (WPS), obtain the geographical data and the boundary/initial conditions for your specific problem, choose the proper model parameters and numerical schemes, and interpret the model output in a scientific way.

In the future, you might be able to install WRF with one-click by Spack [#spack-wrf]_. For WRF specifically, you might also be interested in `EasyBuild <https://easybuild.readthedocs.io>`_ for one-click install. A fun fact is that Spack can also install EasyBuild (see :code:`spack info easybuild`), despite their similar purposes.

That's the end of this guide, which I believe has covered the common patterns for cloud-HPC.

References
==========
.. [#cloud-hpc-growth] Cloud Computing in HPC Surges: https://www.top500.org/news/cloud-computing-in-hpc-surges/
.. [#manual-cluster] See Quick MPI Cluster Setup on Amazon EC2: https://glennklockwood.blogspot.com/2013/04/quick-mpi-cluster-setup-on-amazon-ec2.html. It was written in 2013 but all steps still apply. AWS console looks quite different now, but the concepts are not changed.
.. [#dl-course] For example, fast.ai's tutorial on AWS EC2 https://course.fast.ai/start_aws.html, or Amazon's DL book https://d2l.ai/chapter_appendix/aws.html.
.. [#hpc-carpentry] See Introduction to High-Performance Computing at: https://hpc-carpentry.github.io/hpc-intro/. It only covers very simple cluster usage, not parallel programming.
.. [#placement-group] See Placement Groups in AWS docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html#placement-groups-cluster
.. [#avail-zone] You might want to review "Regions and Availability Zones" in AWS docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html
.. [#ebs-type] See Amazon EBS Volume Types: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html. HDD is cheap and good enough. If I/O is a real problem then you should use FSx for Lustre.
.. [#pcluster-config] The Configuration section in the docs: https://aws-parallelcluster.readthedocs.io/en/latest/configuration.html
.. [#hyper] See Disabling Intel Hyper-Threading Technology on Amazon Linux at: https://aws.amazon.com/blogs/compute/disabling-intel-hyper-threading-technology-on-amazon-linux/
.. [#pcluster-components] AWS Services used in AWS ParallelCluster: https://aws-parallelcluster.readthedocs.io/en/latest/aws_services.html.
.. [#autoscaling] See AutoScaling groups in AWS docs https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html
.. [#private-ip] You might want to review the IP Addressing section in AWS docs: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html
.. [#ena] See Enhanced Networking on AWS docs. https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking.html. For a more techinical discussion, see SR-IOV and Amazon's C3 Instances: https://glennklockwood.blogspot.com/2013/12/high-performance-virtualization-sr-iov.html
.. [#pcluster-autoscaling] See AWS ParallelCluster Auto Scaling: https://aws-parallelcluster.readthedocs.io/en/latest/autoscaling.html
.. [#pcluster-update] See my comment at https://github.com/aws/aws-parallelcluster/issues/307#issuecomment-462215214
.. [#build-netcdf] For example, try building MPI-enabled NetCDF once, and you will never want to do it again: https://www.unidata.ucar.edu/software/netcdf/docs/getting_and_building_netcdf.html
.. [#ompi-slurm] See Running jobs under Slurm in OpenMPI docs: https://www.open-mpi.org/faq/?category=slurm
.. [#slurm-spack] https://github.com/spack/spack/pull/8427#issuecomment-395770378
.. [#remove-mpirun] See the discussion in my PR: https://github.com/spack/spack/pull/10340
.. [#ompi-self] See "3. How do I specify use of sm for MPI messages?" in OpenMPI docs: https://www.open-mpi.org/faq/?category=sm#sm-btl
.. [#ompi-tcp] See Tuning the run-time characteristics of MPI TCP communications in OpenMPI docs: https://www.open-mpi.org/faq/?category=tcp
.. [#ompi-vader] See "What is the vader BTL?" in OpenMPI docs: https://www.open-mpi.org/faq/?category=sm#what-is-vader
.. [#pcluster-ami] See Building a custom AWS ParallelCluster AMI at: https://aws-parallelcluster.readthedocs.io/en/latest/tutorials/02_ami_customization.html
.. [#spack-rapth] A somewhat relevant discussion is that the "Transitive Dependencies" section of Spack docs: https://spack.readthedocs.io/en/latest/workflows.html#transitive-dependencies
.. [#spack-intel] See Integration of Intel tools installed external to Spack: https://spack.readthedocs.io/en/latest/build_systems/intelpackage.html#integration-of-intel-tools-installed-external-to-spack
.. [#intel-aws] See Install Intel® Parallel Studio XE on Amazon Web Services (AWS) https://software.intel.com/en-us/articles/install-intel-parallel-studio-xe-on-amazon-web-services-aws
.. [#spack-external] See Integrating external compilers in Spack docs: https://spack.readthedocs.io/en/latest/build_systems/intelpackage.html?highlight=intel#integrating-external-compilers
.. [#intel-rpath] See this comment at: https://github.com/spack/spack/issues/8315#issuecomment-393160339
.. [#wrf-netcdf] Related discussions are at https://github.com/spack/spack/issues/8816 and https://github.com/wrf-model/WRF/issues/794
.. [#wrf-openmpi] http://forum.wrfforum.com/viewtopic.php?f=5&t=3660
.. [#wrf-error] Just Google "WRF compile error"
.. [#wrf-benchmark] Here's a modern WRF benchmark conducted in 2018: https://akirakyle.com/WRF_benchmarks/results.html
.. [#spack-wrf] Until this PR gets merged: https://github.com/spack/spack/pull/9851
