.. title: MPI over Multiple TCP Connections on EC2 C5n Instances
.. slug: mpi-tcp-ec2
.. date: 2019-03-11 15:42:46 UTC-04:00
.. tags: AWS, Cloud, HPC, MPI
.. category: 
.. link: 
.. description: 
.. type: text

This is just a quick note regarding interesting MPI behaviors on EC2.

EC2 C5n instances provide an amazing 100 Gb/s bandwidth [#c5n]_ , much higher than the 56 Gb/s FDR InfiniBand network on Harvard's HPC cluster [#odyssey]_. It turns out that actually getting the 100 Gb/s bandwidth needs a bit tweak. This post sorely focuses on bandwidth. In terms of latency, Ethernet + TCP (20~30 us) is hard to compete with InfiniBand + RDMA (~1 us).

Tests are conducted on an AWS ParallelCluster with two ``c5n.18xlarge`` instances, as in my `cloud-HPC guide <link://slug/aws-hpc-guide>`_.

.. contents::
.. section-numbering::

TCP bandwidth test with iPerf
=============================

Before doing any MPI stuff, first use the general-purpose network testing tool `iPerf/iPerf3 <https://iperf.fr/>`_. AWS has provided an example of using iPerf on EC2 [#ec2-iperf]_. Note that iPerf2 and iPerf3 handle parallelism quite differently [#iperf-parallel]_: the :code:`--parallel`/:code:`-P` option in iPerf2 creates multiplt threads (thus can ulitize multiple CPU cores), while the same option in iPerf3 opens multiple TCP connections but only one thread (thus can only use a single CPU core). This can lead to quite different benchmark results at high concurrency.

Single-thread results with iPerf3
---------------------------------

Start server:

.. code:: bash

    $ ssh ip-172-31-2-150  # go to one compute node 
    $ sudo yum install iperf3
    $ iperf3 -s  # let it keep running

Start client (in a separate shell):

.. code:: bash

    $ ssh ip-172-31-11-54  # go to another compute node
    $ sudo yum install iperf3

    # Single TCP stream
    # `-c` specifies the server hostname (EC2 private IP). 
    # Most parameters are kept as default, which seem to perform well.
    $ iperf3 -c ip-172-31-2-150 -t 4
    ...
    [ ID] Interval           Transfer     Bandwidth       Retr
    [  4]   0.00-4.00   sec  4.40 GBytes  9.46 Gbits/sec    0             sender
    [  4]   0.00-4.00   sec  4.40 GBytes  9.46 Gbits/sec                  receiver
    ...

    # Multiple TCP stream
    $ iperf3 -c ip-172-31-2-150 -P 4 -i 1 -t 4
    ...
    [SUM]   0.00-4.00   sec  11.8 GBytes  25.4 Gbits/sec    0             sender
    [SUM]   0.00-4.00   sec  11.8 GBytes  25.3 Gbits/sec                  receiver
    ...

A single stream gets ~9.5 Gb/s, while :code:`-P 4` achieves the maximum bandwidth of ~25.4 Gb/s. Using more streams does not help further, as the workload starts to become CPU-limited.


Multi-thread results with iPerf2
--------------------------------

On server:

.. code:: bash

    $ sudo yum install iperf
    $ iperf -s

On client:

.. code:: bash

    $ sudo yum install iperf

    # Single TCP stream
    $ iperf -c ip-172-31-2-150 -t 4  # consistent with iPerf3 result
    ...
    [ ID] Interval       Transfer     Bandwidth
    [  3]  0.0- 4.0 sec  4.40 GBytes  9.45 Gbits/sec
    ...

    # Multiple TCP stream
    $ iperf -c ip-172-31-2-150 -P 36 -t 4  # much higher than with iPerf3
    ...
    [SUM]  0.0- 4.0 sec  43.4 GBytes  93.0 Gbits/sec
    ...

Unlike iPerf3, iPerf2 is able to approach the theoretical 100 Gb/s by using all the available cores.

MPI bandwidth test with OSU mirco-benchmarks
============================================

Next, do `OSU Micro-Benchmarks <http://mvapich.cse.ohio-state.edu/benchmarks/>`_, a well-known MPI benchmarking framework. Similar tests can be done with `Intel MPI Benchmarks <https://github.com/intel/mpi-benchmarks>`_.

Get OpenMPI v4.0.0, which allows a single pair of MPI processes to use multiple TCP connections [#openmpi-multi-tcp]_.

.. code:: bash

    $ spack install openmpi@4.0.0+pmi schedulers=slurm  # Need to fix https://github.com/spack/spack/pull/10853

Get OSU:

.. code:: bash

    $ spack install osu-micro-benchmarks ^openmpi@4.0.0+pmi schedulers=slurm

Focus on point-to-point communication here:

.. code:: bash

    # all the later commands are executed in this directory
    $ cd $(spack location -i osu-micro-benchmarks)/libexec/osu-micro-benchmarks/mpi/pt2pt/

Single-stream
-------------

:code:`osu_bw` tests bandwidth between a single pair of MPI processes.

.. code:: bash

    $ srun -N 2 -n 2 ./osu_bw
      OSU MPI Bandwidth Test v5.5
      Size      Bandwidth (MB/s)
    1                       0.47
    2                       0.95
    4                       1.90
    8                       3.78
    16                      7.66
    32                     15.17
    64                     29.94
    128                    53.69
    256                   105.53
    512                   202.98
    1024                  376.49
    2048                  626.50
    4096                  904.27
    8192                 1193.19
    16384                1178.43
    32768                1180.01
    65536                1179.70
    131072               1180.92
    262144               1181.41
    524288               1181.67
    1048576              1181.62
    2097152              1181.72
    4194304              1180.56

This matches the single-stream result 9.5 Gb/s = 9.5/8 GB/s ~ 1200 MB/s from iPerf.

.. note::

    1 GigaByte (GB) = 8 Gigabits (Gb)

Multi-stream
------------

:code:`osu_mbw_mr` tests bandwidth between multiple pairs of MPI processes.

.. code:: bash

    # Simply calling `srun` on `osu_mbw_mr` seems to hang forever. Not sure why.
    $ # srun -N 2 --ntasks-per-node 36 ./osu_mbw_mr  # in principle it should work

    # Do it in two steps fixes the problem.
    $ srun -N 2 --ntasks-per-node 72 --pty /bin/bash  # request interactive shell

    # `osu_mbw_mr` requires the first half of MPI ranks to be on one node.
    # Check it with the verbose output below. Slurm should have the correct placement by default.
    $ $(spack location -i openmpi)/bin/orterun --tag-output hostname
    ...

    # Actually running the benchmark
    $ $(spack location -i openmpi)/bin/orterun ./osu_mbw_mr
      OSU MPI Multiple Bandwidth / Message Rate Test v5.5
      [ pairs: 72 ] [ window size: 64 ]
      Size                  MB/s        Messages/s
    1                      17.94       17944422.39
    2                      35.76       17878198.29
    4                      71.85       17963002.53
    8                     143.80       17974644.52
    16                    283.00       17687790.85
    32                    551.03       17219816.70
    64                   1067.73       16683260.55
    128                  2076.05       16219122.14
    256                  3890.82       15198501.12
    512                  6790.84       13263356.64
    1024                10165.19        9926942.84
    2048                11454.89        5593209.95
    4096                11967.32        2921708.63
    8192                12597.32        1537758.49
    16384               12686.13         774299.68
    32768               12765.72         389578.83
    65536               12857.16         196184.66
    131072              12829.56          97881.74
    262144              12994.67          49570.75
    524288              12988.97          24774.49
    1048576             12983.20          12381.74
    2097152             13011.67           6204.45
    4194304             12910.31           3078.06

This matches the theoretical maximum bandwidth (100 Gb/s ~ 12500 MB/s).

On an InfiniBand cluster there is typically little difference between single-stream and multi-stream bandwidth. Something to keep in mind regarding TCP/Ethernet/EC2. 

Tweaking TCP connections for OpenMPI
====================================

OpenMPI v4.0.0 allows one pair of MPI processes to use multiple TCP connections via the :code:`btl_tcp_links` parameter [#openmpi-multi-tcp]_.

.. code:: bash

    $ export OMPI_MCA_btl_tcp_links=36  # https://www.open-mpi.org/faq/?category=tuning#setting-mca-params
    $ ompi_info --param btl tcp --level 4 | grep btl_tcp_links  # double check
    MCA btl tcp: parameter "btl_tcp_links" (current value: "36", data source: environment, level: 4 tuner/basic, type: unsigned_int)
    $ srun -N 2 -n 2 ./osu_bw
      OSU MPI Bandwidth Test v5.5
      Size      Bandwidth (MB/s)
    1                       0.46
    2                       0.92
    4                       1.88
    8                       3.78
    16                      7.60
    32                     14.95
    64                     30.34
    128                    56.95
    256                   113.52
    512                   213.36
    1024                  400.18
    2048                  665.80
    4096                  963.67
    8192                 1187.67
    16384                1180.56
    32768                1179.53
    65536                2349.06
    131072               2379.48
    262144               2589.47
    524288               2805.73
    1048576              2853.21
    2097152              2882.12
    4194304              2811.13

This matches the previous iPerf3 result (25 Gb/s ~ 3000 MB/s) regarding single-thread, mutli-TCP bandwidth. A single MPI pair is hard to go further, as the communication is now limited by thread/CPU.

This tweak doesn't actually improve the performance of my real-world HPC code, which should already have multiple MPI connections. The lesson learned is probably -- be careful when conducting micro-benchmarks. The out-of-box :code:`osu_bw` can be misleading on EC2.

References
==========

.. [#c5n] New C5n Instances with 100 Gbps Networking: https://aws.amazon.com/blogs/aws/new-c5n-instances-with-100-gbps-networking/ 
.. [#odyssey] Odyssey Architecture: https://www.rc.fas.harvard.edu/resources/odyssey-architecture/
.. [#ec2-iperf] "How do I benchmark network throughput between Amazon EC2 Linux instances in the same VPC?" https://aws.amazon.com/premiumsupport/knowledge-center/network-throughput-benchmark-linux-ec2/
.. [#iperf-parallel] Discussions on multithreaded iperf3 at: https://github.com/esnet/iperf/issues/289
.. [#openmpi-multi-tcp] "Can I use multiple TCP connections to improve network performance?" https://www.open-mpi.org/faq/?category=tcp#tcp-multi-links
