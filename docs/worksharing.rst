Worksharing and Scheduling
==========================

**Authors:** Pedro Ojeda & Joachim Hein  
**Institutions:** High Performance Computing Center North (HPC2N), Umeå University, and Lund University  

Overview
---------

Worksharing constructs in OpenMP allow easy distribution of work onto threads.

**Main constructs:**

- **Loop construct** — Distribute loop iterations among threads and manage load imbalance using the ``schedule`` clause.
- **Workshare construct** — Parallelize Fortran array syntax.
- **Sections construct** — Distribute independent code blocks.
- **Modern alternative:** ``task`` construct for dynamic workloads.

Loop Construct
---------------

Distributes loop iterations across threads automatically.  
OpenMP determines for each thread:

- Number of iterations  
- Starting and ending indices  

Registers are flushed to memory at loop exit unless ``nowait`` is used.

Example (Fortran)
~~~~~~~~~~~~~~~~~

.. code-block:: fortran

    !$omp parallel shared(vect, norm) private(i, lNorm)
    lNorm = 0.0d0
    !$omp do
    do i = 1, N
        lNorm = lNorm + vect(i) * vect(i)
    end do
    !$omp atomic update
    norm = norm + lNorm
    !$omp end parallel
    norm = sqrt(norm)

Example (C)
~~~~~~~~~~~

.. code-block:: c

    #pragma omp parallel shared(vect, norm) private(i, lNorm)
    {
        lNorm = 0.0;
        #pragma omp for
        for (i = 0; i < N; i++)
            lNorm += vect[i] * vect[i];
        #pragma omp atomic update
        norm += lNorm;
    }
    norm = sqrt(norm);

Parallel Loop Constructs
~~~~~~~~~~~~~~~~~~~~~~~~

Shorthand syntax if the parallel region is only for the loop:

.. code-block:: fortran

    !$omp parallel do
    do i = 1, N
        loop_body
    end do

.. code-block:: c

    #pragma omp parallel for
    for (int i = 0; i < N; i++)
        loop_body;

Loop Dependencies
~~~~~~~~~~~~~~~~~

Iterations are executed in potentially different order from serial execution.
If iterations depend on previous results, correctness issues arise.

**Solutions:**

- Redesign the algorithm.
- Serialize dependent parts using OpenMP synchronization features.
- Run the loop serially.

Scheduling Loop Iterations
--------------------------

Uneven workload across loop iterations can cause **load imbalance**.

Use the ``schedule`` clause to control how iterations are distributed:

.. code-block:: fortran

    !$omp do schedule(kind, [chunk_size])

Supported ``kind`` values:

- ``static``
- ``dynamic``
- ``guided``
- ``auto``
- ``runtime``

Static Scheduling
~~~~~~~~~~~~~~~~~

Divides iterations into equal chunks (default behavior).  
Least overhead, but may cause imbalance if iterations have variable work.

Dynamic Scheduling
~~~~~~~~~~~~~~~~~~

Threads dynamically request new chunks after finishing their current work.  
Better balance, higher overhead.

Guided Scheduling
~~~~~~~~~~~~~~~~~

Similar to dynamic scheduling but with decreasing chunk sizes for better performance near the end of the loop.

Runtime and Auto Scheduling
~~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``runtime`` — Controlled by environment variable ``OMP_SCHEDULE``.
- ``auto`` — Left to the implementation to decide.

Nested Loops and Collapse
--------------------------

For nested loops, OpenMP can collapse loops into a single iteration space:

.. code-block:: fortran

    !$omp parallel
    !$omp do collapse(2)
    do j = 1, 3
        do i = 1, 4
            a(i,j) = expensiveFunc(i,j)
        end do
    end do

Workshare in Fortran
---------------------

Fortran provides ``workshare`` to parallelize array syntax operations:

.. code-block:: fortran

    !$omp parallel shared(n, a, b, c)
    !$omp workshare
        b(1:n) = b(1:n) + 1
        c(1:n) = c(1:n) + 2
        a(1:n) = b(1:n) + c(1:n)
    !$omp end workshare
    !$omp end parallel

**Notes:**

- All threads wait at ``end workshare``.
- Only scalar assignments to shared variables are allowed.

Sections Construct
-------------------

Use ``sections`` when different code blocks can be executed independently.

.. code-block:: c

    #pragma omp parallel sections shared(a, b, N, M)
    {
        #pragma omp section
        for (int i = 0; i < N; i++) a[i] = i;

        #pragma omp section
        for (int j = 0; j < M; j++) b[j] = initBmatrix(j, M);
    }

**Caution:** Possible load imbalance if blocks take different time or number of threads ≠ number of sections.

Summary
--------

- ``loop`` construct simplifies parallelization of standard loops.  
- ``schedule`` clause handles load imbalance.  
- ``workshare`` parallelizes Fortran array expressions.  
- ``sections`` distributes independent code blocks.  
- ``collapse`` allows nested loop flattening.  

OpenMP’s worksharing constructs form the foundation of shared-memory parallelism for structured workloads.


===============================
OpenMP Worksharing and Scheduling
===============================

Overview
========

OpenMP provides several worksharing constructs to distribute work across threads:

- **Loop construct**: Easy distribution of loops onto threads
- **Workshare construct**: Parallelization of Fortran array syntax
- **Sections construct**: Distributing independent code blocks

Distributing Loops
==================

Loop Construct
--------------

The loop construct distributes loop iterations across threads automatically:

- Iteration variable is automatically private
- Determines iterations per thread, start index, and final index
- Registers are flushed to memory at exit (unless ``nowait`` is used)
- No flush on entry

Fortran Syntax
~~~~~~~~~~~~~~

.. code-block:: fortran

   !$omp parallel shared(...) private(...)
   !$omp do
   do i = 1, N
      ! loop body
   end do
   !$omp end parallel

C/C++ Syntax
~~~~~~~~~~~~

.. code-block:: c

   #pragma omp parallel shared(...) private(...)
   {
      #pragma omp for
      for (int i = 0; i < N; i++) {
         // loop body
      }
   }

Parallel Loop Construct
-----------------------

Shorthand when a parallel region contains only a loop construct:

Fortran:
~~~~~~~~

.. code-block:: fortran

   !$omp parallel do
   do i = 1, N
      ! loop body
   end do

C/C++:
~~~~~~

.. code-block:: c

   #pragma omp parallel for
   for (int i = 0; i < N; i++) {
      // loop body
   }

Data Dependency Considerations
------------------------------

- In parallel loops, iterations execute in different order from serial code
- Correct results only if iterations are independent
- If data dependency exists:
  - Modify/change algorithm
  - Serialize relevant parts using OpenMP features
  - Execute loop serially

Scheduling Loop Iterations
==========================

Schedule Clause
---------------

Use the schedule clause to handle load imbalance:

.. code-block:: c

   schedule(kind [, chunk_size])

Available Schedule Types
------------------------

Static Scheduling
~~~~~~~~~~~~~~~~~

- Divides iterations into equal-sized chunks
- Round-robin thread assignment
- Least overhead
- Default: divide iterations by number of threads

Dynamic Scheduling
~~~~~~~~~~~~~~~~~~

- Loop split into work packages of ``chunk_size`` iterations
- Threads request new packages when done
- Default ``chunk_size``: 1 iteration

Guided Scheduling
~~~~~~~~~~~~~~~~~

- Similar to dynamic scheduling
- Work package size proportional to unassigned iterations/number of threads
- Never smaller than ``chunk_size`` (except last package)
- Default ``chunk_size``: 1

Auto and Runtime Scheduling
~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **auto**: Implementation decides schedule
- **runtime**: Schedule controlled at runtime via:
  - ``omp_set_schedule()`` function
  - ``OMP_SCHEDULE`` environment variable

Multiple Loop Parallelization
=============================

Options for Nested Loops
------------------------

1. **Distribute outer loop**: Limited to number of outer iterations
2. **Distribute inner loop**: More work packages, potential cache conflicts
3. **Use collapse clause**: Combines loops into single iteration space

Collapse Clause
---------------

- Specifies number of loops to collapse into single iteration space
- Available in OpenMP 3.0 and later

Fortran Example:
~~~~~~~~~~~~~~~~

.. code-block:: fortran

   !$omp parallel
   !$omp do collapse(2)
   do j = 1, 3
      do i = 1, 4
         a(i,j) = expensiveFunc(i,j)
      end do
   end do
   !$omp end parallel

C/C++ Example:
~~~~~~~~~~~~~~

.. code-block:: c

   #pragma omp parallel
   #pragma omp for collapse(2)
   for (int i = 0; i < 3; i++) {
      for (int j = 0; j < 4; j++) {
         a[i][j] = expensiveFunc(i,j);
      }
   }

Workshare in Fortran
====================

The workshare construct distributes:

- Fortran array syntax
- FORALL statements
- WHERE statements

Example:
--------

.. code-block:: fortran

   !$OMP PARALLEL SHARED(n, a, b, c)
   !$OMP WORKSHARE
      b(1:n) = b(1:n) + 1
      c(1:n) = c(1:n) + 2
      a(1:n) = b(1:n) + c(1:n)
   !$OMP END WORKSHARE
   !$OMP END PARALLEL

Important Notes:
----------------

- Scalar assignment to shared variables is legal
- Scalar assignment to private variables in workshare is ILLEGAL
- User-defined functions must be declared ELEMENTAL

Sections Construct
==================

The sections construct allows parallel execution of independent code blocks.

Basic Syntax:
-------------

.. code-block:: c

   #pragma omp parallel sections shared(a, b, N, M)
   {
      #pragma omp section
      {
         for(int i = 0; i < N; i++)
            a[i] = i;
      }
      #pragma omp section
      {
         for(int i = 0; i < M; i++)
            b[i] = initBmatrix(i,M);
      }
   }

Considerations:
---------------

- Mismatch possible between code blocks and threads
- Potential load imbalance if blocks have different workloads
- Useful for initializing multiple data structures

Summary
=======

- **Loop construct**: Easy distribution of standard loops with schedule clause for load balancing
- **Workshare construct**: Distribution of Fortran array syntax statements
- **Sections construct**: Distribution of independent code blocks across threads

