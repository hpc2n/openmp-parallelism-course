Introduction to Kebnekaise
--------------------------

.. objectives::

 - Learn how to load the necessary modules on Kebnekaise.
 - Learn how to compile C code on Kebnekaise.
 - Learn how to compile CUDA code on Kebnekaise.
 - Learn how to place jobs to the batch queue.
 - Learn how to use the course project and reservations.

Modules and toolchains
^^^^^^^^^^^^^^^^^^^^^^

You need to **load the correct toolchain** before compiling your code on Kebnekaise.

The **available modules** are listed using the :code:`ml avail` command:

.. code-block:: bash

    $ ml avail
    ------------------------- /hpc2n/eb/modules/all/Core --------------------------
    Bison/3.0.5                        fosscuda/2020a
    Bison/3.3.2                        fosscuda/2020b        (D)
    Bison/3.5.3                        gaussian/16.C.01-AVX2
    Bison/3.7.1                (D)     gcccuda/2019b
    CUDA/8.0.61                        gcccuda/2020a
    CUDA/10.1.243              (D)     gcccuda/2020b         (D)
    ...

The list shows the modules you can load directly, and so may change if you have loaded modules.

In order to see all the modules, including those that have prerequisites to load, use the command :code:`ml spider`. Many types of application software fall in this category. 

You can find more **information** regarding a particular module using the :code:`ml spider <module>` command:

.. code-block:: bash

    $ ml spider MATLAB

    ---------------------------------------------------------------------------
    MATLAB: MATLAB/2019b.Update2
    ---------------------------------------------------------------------------
        Description:
        MATLAB is a high-level language and interactive environment that
        enables you to perform computationally intensive tasks faster than
        with traditional programming languages such as C, C++, and Fortran.


        This module can be loaded directly: module load MATLAB/2019b.Update2

        Help:
        Description
        ===========
        MATLAB is a high-level language and interactive environment
        that enables you to perform computationally intensive tasks faster than with
        traditional programming languages such as C, C++, and Fortran.
        
        
        More information
        ================
        - Homepage: http://www.mathworks.com/products/matlab

You can **load** the module using the :code:`ml <module>` command:

.. code-block:: bash

    $ ml MATLAB/2019b.Update2

You can **list loaded modules** using the :code:`ml` command:

.. code-block:: bash

    $ ml

    Currently Loaded Modules:
     1) snicenvironment     (S)   7) libevent/2.1.11    13) PMIx/3.0.2
     2) systemdefault       (S)   8) numactl/2.0.12     14) impi/2018.4.274
     3) GCCcore/8.2.0             9) XZ/5.2.4           15) imkl/2019.1.144
     4) zlib/1.2.11              10) libxml2/2.9.8      16) intel/2019a
     5) binutils/2.31.1          11) libpciaccess/0.14  17) MATLAB/2019b.Update2
     6) iccifort/2019.1.144      12) hwloc/1.11.11

    Where:
     S:  Module is Sticky, requires --force to unload or purge
    
You can **unload all modules** using the :code:`ml purge` command:

.. code-block:: bash

    $ ml purge
    The following modules were not unloaded:
      (Use "module --force purge" to unload all):

      1) systemdefault   2) snicenvironment

Note that the :code:`ml purge` command will warn that two modules were not unloaded. 
This is normal and you should **NOT** force unload them.

.. challenge::

    1. Load the FOSS toolchain for source code compilation:
 
       .. code-block:: bash
       
            $ ml purge
    
       The :code:`foss` module loads the GNU compiler 
       
    2. Investigate which modules were loaded.
       
    3. Purge all modules.
       
    4. Find the latest FOSS toolchain (:code:`foss`). 
       Investigate the loaded modules.
       Purge all modules.

Compile C code
^^^^^^^^^^^^^^

Once the correct toolchain (:code:`foss`) has been loaded, we can compile C source files (:code:`*.c`) with the GNU compiler:

.. code-block:: bash

    $ gcc -o <binary name> <sources> -Wall

The :code:`-Wall` causes the compiler to print additional warnings.

.. challenge::

    Compile the following "Hello world" program:
    
    .. code-block:: c
        :linenos:
    
        #include <stdio.h>

        int main() {
            printf("Hello world!\n");
            return 0;
        }


Course project
^^^^^^^^^^^^^^

You can request to be a member of the course project hpc2n202w-xyz, where the letters
need to be substituted by the actual numerical values for the project.

Submitting jobs
^^^^^^^^^^^^^^^

The jobs are **submitted** using the :code:`srun` command:

.. code-block:: bash

    $ srun --account=<account> --ntasks=<task count> --time=<time> <command>

This places the command into the batch queue.
The three arguments are the project number, the number of tasks, and the requested time allocation.
For example, the following command prints the uptime of the allocated compute node:

.. code-block:: bash

    $ srun --account=hpc2n202w-xyz --ntasks=1 --time=00:00:15 uptime
    srun: job 12727702 queued and waiting for resources
    srun: job 12727702 has been allocated resources
     11:53:43 up 5 days,  1:23,  0 users,  load average: 23,11, 23,20, 23,27

Note that we are using the course project, the number of tasks is set to one, and we are requesting 15 seconds.


We could submit **multiple tasks** using the :code:`--ntasks=<task count>` argument:

.. code-block:: bash

    $ srun --account=hpc2n202w-xyz --ntasks=4 --time=00:00:15 uname -n
    b-cn0932.hpc2n.umu.se
    b-cn0932.hpc2n.umu.se
    b-cn0932.hpc2n.umu.se
    b-cn0932.hpc2n.umu.se
    
Note that all task are running on the same node.
We could request **multiple CPU cores** for each task using the :code:`--cpus-per-task=<cpu count>` argument:

.. code-block:: bash

    $ srun --account=hpc2n202w-xyz --ntasks=4 --cpus-per-task=14 --time=00:00:15 uname -n
    b-cn0935.hpc2n.umu.se
    b-cn0935.hpc2n.umu.se
    b-cn0932.hpc2n.umu.se
    b-cn0932.hpc2n.umu.se

If you want to measure the performance, it is advisable to request an **exclusive access** to the compute nodes (:code:`--exclusive`):

.. code-block:: bash

    $ srun --account=hpc2n202w-xyz --ntasks=4 --cpus-per-task=14 --exclusive --time=00:00:15 uname -n
    b-cn0935.hpc2n.umu.se
    b-cn0935.hpc2n.umu.se
    b-cn0932.hpc2n.umu.se
    b-cn0932.hpc2n.umu.se
    

.. challenge::

    Run both "Hello world" programs on the the compute nodes.
 
Aliases
^^^^^^^

In order to save time, you can create an **alias** for a command:

.. code-block:: bash

    $ alias <alist>="<command>"

For example:

.. code-block:: bash

    $ alias run_full="srun --account=hpc2n202w-xyz --ntasks=1 --cpus-per-task=28 --time=00:05:00"
    $ run_full uname -n
    b-cn0932.hpc2n.umu.se

Batch files
^^^^^^^^^^^

It is often more convenient to write the commands into a **batch file**.
For example, we could write the following to a file called :code:`batch.sh`:

.. code-block:: bash
    :linenos:

    #!/bin/bash
    #SBATCH --account=hpc2n202w-xyz
    #SBATCH --ntasks=1
    #SBATCH --time=00:00:15

    ml purge
    ml foss/2020b

    uname -n

Note that the same arguments that were earlier passed to the :code:`srun` command are now given as comments.
It is highly advisable to purge all loaded modules and re-load the required modules as the job inherits the environment.
The batch file is submitted using the :code:`sbatch <batch file>` command:
    
.. code-block:: bash

    sbatch batch.sh 
    Submitted batch job 12728675

By default, the output is directed to the file :code:`slurm-<job_id>.out`, where :code:`<job_id>` is the **job id** returned by the :code:`sbatch` command:

.. code-block:: bash

    $ cat slurm-12728675.out 
    The following modules were not unloaded:
     (Use "module --force purge" to unload all):

     1) systemdefault   2) snicenvironment
    b-cn0102.hpc2n.umu.se
    
.. challenge::
        
    Write two batch files that run both "Hello world" programs on the the compute nodes.
        
Job queue
^^^^^^^^^
        
You can **investigate the job queue** with the :code:`squeue` command:

.. code-block:: bash

    $ squeue -u $USER

If you want an estimate for when the job will start running, you can give the :code:`squeue` command the argument :code:`--start`. 

You can **cancel** a job with the :code:`scancel` command:

.. code-block:: bash

    $ scancel <job_id>

What is High Performance Computing?
"""""""""""""""""""""""""""""""""""

*High Performance Computing most generally refers to the practice of aggregating computing power in a way that delivers much higher performance than one could get out of a typical desktop computer or workstation in order to solve large problems in science, engineering, or business.* (`insideHPC.com <https://insidehpc.com/hpc-basic-training/what-is-hpc/>`__)

What does this mean?
 - Aggregating computing power
    - Kebnekaise: 602 nodes in 15 racks totalling 19288 cores
    - Your laptop: 4 cores
 - Higher performance
    - Kebnekaise: 728,000 billion arithmetical operations per second
    - Your laptop: 200 billion arithmetical operations per second
 - Solve large problems
    - **Time:** The time required to form a solution to the problem is very long.
    - **Memory:** The solution of the problem requires a lot of memory and/or storage.

.. figure:: img/hpc.png
    :align: center
    :scale: 70 %
    
Memory models
"""""""""""""

When it comes to the memory layout, (super)computers can be divided into two primary categories: 

:Shared memory: A **single** memory space for all data:

 - Everyone can access the same data.
 - Straightforward to use.

 .. figure:: img/sm.png
    :align: left
    :scale: 70 %
    
:Distributed memory: **Multiple distinct** memory spaces for the data:

 - Everyone has direct access only to the **local data**.
 - Requires **communication** and **data transfers**.

 .. figure:: img/dm.png
    :align: left
    :scale: 70 %

Computing clusters and supercomputers are generally distributed memory machines:
    
.. figure:: img/memory.png
    :align: center
    :scale: 70 %
    
Programming models
""""""""""""""""""

The programming model changes when we aim for extra performance and/or memory:

:Single-core: Matlab, Python, C, Fortran, ...

 - **Single stream of operations** (thread).
 - **Single pool of data**.
    
 .. figure:: img/single-core.png
    :align: left
    :scale: 70 %

:Multi-core: Vectorized Matlab, pthreads, **OpenMP**

 - **Multiple** streams of operations (multiple threads).
 - Single pool of data.
 - Extra challenges:
 
    - **Work distribution**.
    - **Coordination** (synchronization, etc).

 .. figure:: img/multi-core.png
    :align: left
    :scale: 70 %
    
:Distributed memory: **MPI**, ...

 - Multiple streams of operations (multiple threads).
 - **Multiple** pools of data.
 - Extra challenges:
 
    - Work distribution.
    - Coordination (synchronization, etc).
    - **Data distribution**.
    - **Communication** and **data transfers**.

 .. figure:: img/distributed-memory.png
    :align: left
    :scale: 70 %
 
:Accelerators / GPUs: **CUDA**, OpenCL, OpenACC, OpenMP, ...

 - Single/multiple streams of operations on the **host device**.
 - Many **lightweight** streams of operations on the **accelerator**.
 - Multiple pools of data on **multiple layers**.
 - Extra challenges:
 
    - Work distribution.
    - Coordination (synchronization, etc).
    - Data distribution across **multiple memory spaces**.
    - Communication and data transfers.

 .. figure:: img/gpu.png
    :align: left
    :scale: 70 %

:Hybrid: MPI **+** OpenMP, OpenMP **+** CUDA, MPI **+** CUDA, ...

 - Combines the benefits and the downsides of several programming models.
 
 .. figure:: img/hybrid.png
    :align: left
    :scale: 65 %

:Task-based: OpenMP tasks, StarPU

 - Does task-based programming count as a separate programming model?
 - StarPU = (implicit) MPI + (implicit) pthreads + CUDA
    
Functions and data dependencies
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Imagine the following computer program:

.. code-block:: c
    :linenos:
    
    #include <stdio.h>
    
    void function1(int a, int b) {
        printf("The sum is %d.\n", a + b);
    }
    
    void function2(int b) {
        printf("The sum is %d.\n", 10 + b);
    }
    
    int main() {
        int a = 10, b = 7;
        function1(a, b);
        function2(b);
        return 0;
    }

The program consists of two functions, :code:`function1` and :code:`function2`, that are called **one after another** from the :code:`main` function.
The first function reads the variables :code:`a` and :code:`b`, and the second function reads the variable :code:`b`:

.. figure:: img/functions_nodep.png

The program prints the line :code:`The sum is 17.` twice.
The key observation is that the two functions calls are **independent** of each other.
More importantly, the two functions can be executed in **parallel**:

.. figure:: img/functions_nodep_parallel.png

Let us modify the the program slightly:

.. code-block:: c
    :linenos:
    :emphasize-lines: 3-6,14
    
    #include <stdio.h>
    
    void function1(int a, int *b) {
        printf("The sum is %d.\n", a + *b);
        *b += 3;
    }
    
    void function2(int b) {
        printf("The sum is %d.\n", 10 + b);
    }
    
    int main() {
        int a = 10, b = 7;
        function1(a, &b);
        function2(b);
        return 0;
    }

This time the function :code:`function1` modifies the variable :code:`b`:
    
.. figure:: img/functions_dep.png

Therefore, the two function calls are **not** independent of each other and changing the order would change the printed lines.
Furthermore, executing the two functions in parallel would lead to an **undefined result** as the execution order would be arbitrary.

We could say that **in this particular context**, the function :code:`function2` is **dependent** on the function :code:`function1`.
That is, the function :code:`function1` must be executed completely before the function :code:`function2` can be executed:

.. figure:: img/functions_dep_explicit.png

However, this **data dependency** exists only when these two functions are called in this particular sequence using these particular arguments.
In a different context, this particular data dependency does not exists.
We can therefore conclude that the **data dependencies are separate from the functions definitions**.