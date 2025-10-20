Worksharing and Scheduling
--------------------------


Worksharing constructs allow easy distribution of work onto threads:

**Loop Construct:**

- Easy distribution of loops onto threads
- Avoiding load imbalance using the ``schedule`` clause

**Workshare Construct (Fortran):**

- Parallelization of Fortran array syntax

**Sections Construct:**

- Distributing independent code blocks
- Modern alternative: task construct



Distributing Loops
^^^^^^^^^^^^^^^^^^


Distributing large loops is a typical target for OpenMP parallelization.

*Traditional Approach*

As seen in previous lectures, loop distribution can be accomplished manually but requires management code:

- Number of iterations per thread (may be unequal)
- Starting index of current thread
- Final index of current thread

**OpenMP Loop Construct**

OpenMP offers the "loop construct" to ease loop parallelization:

- Convenience: reduces code complexity
- Maintainability: cleaner, more readable code


*The Loop Construct Features*

The loop construct:

- Distributes the following loop (C/C++/Fortran) onto threads
- Makes the iteration variable automatically private
- Determines automatically (without management code):
  
  - Number of iterations per thread
  - Start index of current thread
  - Final index of current thread

- Flushes registers to memory at exit, unless ``nowait`` is specified
  
  - **Note:** No flush on entry!

- Offers mechanisms to balance the load for various situations


*Loop Construct in Fortran*

Works on Fortran standard-compliant do-construct:

- **Not supported:** ``do while``
- **Not supported:** ``do`` without loop control


.. code-block:: fortran

    !$omp parallel &
    !$omp shared(…) &
    !$omp private(…)
        !$omp do
        do i = 1, N
            loop-body
        end do
    !$omp end parallel

.. note::
   ``!$omp end do`` is not required but optional.


*Example: Vector Norm - Manual Loop Management (Fortran)*

.. code-block:: fortran

    norm = 0.0D0
    
    !$omp parallel default(none) &
    !$omp shared(vect, norm) private(myNum, i, lNorm)
        lNorm = 0.0D0
        myNum = vleng / omp_get_num_threads()  ! local size
        
        do i = 1 + myNum * omp_get_thread_num(), &
                myNum * (1 + omp_get_thread_num())
            lNorm = lNorm + vect(i) * vect(i)
        enddo
        
        !$omp atomic update
        norm = norm + lNorm
    !$omp end parallel
    
    norm = sqrt(norm)

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

.. note::
   This version requires explicit calculation of loop bounds for each thread.


*Example: Vector Norm - Loop Construct (Fortran)*

.. code-block:: fortran

    norm = 0.0d0
    
    !$omp parallel default(none) &
    !$omp shared(vect, norm) private(i, lNorm)
        lNorm = 0.0d0
        
        !$omp do
        do i = 1, vleng  ! same as serial case
            lNorm = lNorm + vect(i) * vect(i)
        enddo
        
        !$omp atomic update
        norm = norm + lNorm
    !$omp end parallel
    
    norm = sqrt(norm)

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

.. important::
   The loop bounds are the same as in the serial case. OpenMP handles the distribution automatically.


*Loop Construct in C*

The loop construct in C is limited to "canonical" loops:

**First Argument (Initialization):**

Assignment to:

- ``int``
- pointer
- random-access-iterator-type (C++)

**Second Argument (Condition):**

Comparison using: ``<=``, ``<``, ``>``, ``>=``

**Third Argument (Increment):**

- ``i++``, ``++i``, ``i--``, ``--i``
- ``i += inc``, ``i -= inc``
- ``i = i + inc``, ``i = inc + i``, ``i = i - inc``

**Additional Requirements:**

All bounds and increments must be loop-invariant.


.. code-block:: c

    #pragma omp parallel \
        shared(…) \
        private(…)
    {
        #pragma omp for
        for (i = 0; i < N; i++)
        {
            loop-body
        }
    }


*Example: Vector Norm - Manual Loop Management (C)*

.. code-block:: c

    norm = 0.0;
    
    #pragma omp parallel default(none) \
        shared(vect, norm) private(myNum, i, lNorm)
    {
        lNorm = 0.0;
        myNum = vleng / omp_get_num_threads();  // local size
        
        for (i = myNum * omp_get_thread_num();
             i < myNum * (1 + omp_get_thread_num()); i++)
            lNorm += vect[i] * vect[i];
        
        #pragma omp atomic update
        norm += lNorm;
    }
    
    norm = sqrt(norm);

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

.. note::
   This version requires explicit calculation of loop bounds for each thread.



*Example: Vector Norm - Loop Construct (C)*

.. code-block:: c

    norm = 0.0;
    
    #pragma omp parallel default(none) \
        shared(vect, norm) private(i, lNorm)
    {
        lNorm = 0.0;
        
        #pragma omp for
        for (i = 0; i < vleng; i++)  // same as serial case
            lNorm += vect[i] * vect[i];
        
        #pragma omp atomic update
        norm += lNorm;
    }
    
    norm = sqrt(norm);

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

.. important::
   The loop bounds are the same as in the serial case. OpenMP handles the distribution automatically.


Parallel Loop Construct 
^^^^^^^^^^^^^^^^^^^^^^^

*Parallel Loop Construct in Fortran*

When a parallel region contains only a loop construct, you can use a shorthand:

.. code-block:: fortran

    !$omp parallel do
    do i = 1, N
        loop-body
    enddo  ! parallel region ends here!

.. note::
   - ``!$omp end parallel do`` is not required (optional)
   - Features of parallel region and normal loop construct apply similarly



*Parallel Loop Construct in C*


When a parallel region contains only a loop construct, you can use a shorthand:

.. code-block:: c

    #pragma omp parallel for
    for (int i = 0; i < N; i++)
    {
        loop-body
    }  // parallel region & loop construct end here!

.. note::
   Features of parallel region and normal loop construct apply similarly.



**Loop Reordering and Data Dependency**

*Order of Execution*


In a parallel loop, iterations are executed in a different order from serial code.

*Data Dependency Requirement*


A correct result is only obtained if the current iteration is independent of previous iterations (no data dependency).

*Handling Data Dependencies*


If data dependency exists:

1. Modify/change the algorithm
2. Serialize relevant part of the loop using special OpenMP features (covered later in course)
3. Execute loop serially

*Example with Dependency*


**Problem (has dependency):**

.. code-block:: c

    a[0] = 0;
    for (i = 1; i < N; i++)
        a[i] = a[i-1] + i;

**Possible Fix (algorithm change):**

.. code-block:: c

    for (i = 0; i < N; i++)
        a[i] = 0.5 * i * (i + 1);

.. warning::
   Always verify that loop iterations are independent before parallelizing!



Scheduling Loop Iterations
^^^^^^^^^^^^^^^^^^^^^^^^^^

*Work Per Loop Iteration*

Previous examples assumed the same amount of work for each loop iteration. This is not always the case.

*Examples of Uneven Work*

**Summing over triangular area:**

.. code-block:: c

    for (i = 0; i < N; i++)
        for (j = 0; j < i + 1; j++)
            // work here

**Loop body iterates until required accuracy is achieved**

*Load Imbalance Problem*

Uneven work distribution often causes load imbalance:

- Some threads finish while others still work
- Results in **poor performance**

.. note::
   Dealing with such problems is typically easier in shared memory than in distributed memory programming.



**Schedule Clause**


To help load balance in a loop construct, use the ``schedule`` clause:

.. code-block:: fortran

    schedule(kind, [chunk_size])

*Default Behavior*

Default schedule is implementation-dependent (OpenMP 3.0).

*Schedule Kinds*

Choices for ``kind``:

- ``static``
- ``dynamic``
- ``guided``
- ``auto``
- ``runtime``


*Static Scheduling*


1. Divide iteration count into chunks of equal size
   
   - Last chunk may be smaller if needed

2. Thread assignment uses "round robin" distribution

*Default Chunk Size*

Default chunk size divides iteration count by number of threads.

*Performance*

Static scheduling has the **least overhead** compared to other schedules.

*Visual Representation*

.. code-block:: text

    Default static schedule (≈n/4 per thread):
    Thread 0: [===============]
    Thread 1: [===============]
    Thread 2: [===============]
    Thread 3: [===============]
    
    Static schedule with chunk size:
    T0 T1 T2 T3 T0 T1 T2 T3 T0 T1 T2 ...



*Example: Summation Over Triangular Area (Static)*


.. code-block:: fortran

    !$omp parallel do &
    !$omp private(i, j) shared(a) &
    !$omp schedule(static, 100)
    do j = 1, 1200
        do i = j + 1, 1200
            a(i,j) = func(i,j)
            a(j,i) = -a(i,j)
        enddo
    enddo

*Performance Comparison*

- **Default static:** maximum 7/16 of work area per thread
- **Static with chunk=100:** maximum 5/16 of work area per thread

*Trade-offs*

- **Smaller chunks:** better load balance
- **More chunks:** larger overhead



**Dynamic Scheduling**


1. Loop is split into work packages of ``chunk_size`` iterations
2. Each thread requests a new work package once done with the current one
3. Default ``chunk_size``: 1 iteration

*When to Use*

Use dynamic scheduling when:

- Work per iteration varies significantly
- The pattern of work is unpredictable

*Performance*

- Better load balance than static (for uneven work)
- Higher overhead than static due to runtime work distribution


*Example: Summation Over Triangular Area (Dynamic)*

.. code-block:: fortran

    !$omp parallel do &
    !$omp private(i, j) shared(a) &
    !$omp schedule(dynamic, 100)
    do j = 1, 1200
        do i = j + 1, 1200
            a(i,j) = func(i,j)
            a(j,i) = -a(i,j)
        enddo
    enddo

*Performance Comparison*

- **Default static:** maximum 7/16 of work area per thread
- **Dynamic with chunk=100:** maximum ≈0.27 of work area per thread

*Trade-offs*

- **Better balance** than static scheduling
- **Larger overhead** than static scheduling


**Guided Scheduling**


Similar to dynamic, but with adaptive chunk sizes:

1. Threads request new work packages once done
2. Work package size is proportional to:
   
   .. code-block:: text
   
       (number of unassigned iterations) / (number of threads)

3. Package size never smaller than ``chunk_size`` (unless last package)
4. Default ``chunk_size`` = 1

*Purpose*

The idea is to **prevent expensive work packages at the end** of the loop.

*Performance*

- Starts with large chunks (low overhead)
- Gradually decreases chunk size (better balance toward the end)


**Schedules: Auto and Runtime**


*Auto Schedule*


For ``auto``, the implementation decides the scheduling strategy.

.. code-block:: fortran

    !$omp parallel do schedule(auto)

*Runtime Schedule*

For ``runtime``, the schedule can be controlled at runtime:

**Method 1: Using Function (OpenMP 3.0)**

.. code-block:: c

    omp_set_schedule(omp_sched_static, 10);

**Method 2: Using Environment Variable**

Bash:

.. code-block:: bash

    export OMP_SCHEDULE="guided,4"

C-shell:

.. code-block:: csh

    setenv OMP_SCHEDULE "guided,4"

.. warning::
   Do not specify ``chunk_size`` with ``auto`` or ``runtime`` in the directive itself.



Multiple Loop Parallelization
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Simple Example with Nested Loops

Consider this nested loop structure:

.. code-block:: fortran

    do j = 1, 3
        do i = 1, 4
            a(i,j) = expensiveFunc(i,j)
        enddo
    enddo

There are **three basic options** to parallelize nested loops. Which one is best depends on the specific situation.


*Option 1: Distribute Outer Loop (Fortran)*


.. code-block:: fortran

    !$omp parallel do
    do j = 1, 3
        do i = 1, 4
            a(i,j) = expensiveFunc(i,j)
        enddo
    enddo


- Distributes the j-loop
- Maximally **3 work packages**

*When to Use*

Use when the outer loop has sufficient iterations for good load balance.


*Option 2: Distribute Inner Loop (Fortran)*

.. code-block:: fortran

    !$omp parallel private(j)
    do j = 1, 3
        !$omp do
        do i = 1, 4
            a(i,j) = expensiveFunc(i,j)
        enddo
        !$omp end do
    enddo
    !$omp end parallel



- Distributes the i-loop
- Now **four work packages**
- Parallel region before j-loop provides better performance
- Requires ``i`` to be private (automatic for loop variable)
- Starts loop construct 3 times
- May cause more cache line conflicts when writing to ``a``



*Option 3: Collapse Clause (Fortran)*

.. code-block:: fortran

    !$omp parallel
    !$omp do collapse(2)
    do j = 1, 3
        do i = 1, 4
            a(i,j) = expensiveFunc(i,j)
        enddo
    enddo



- Use ``collapse`` clause to specify number of loops to collapse
- Available since **OpenMP 3.0**
- Distributes both loops by creating a single combined loop
- Schedules as specified (default in this case)
- Now: **12 work packages**
- May cause more cache line conflicts when writing to ``a``

*Benefits*

- Maximum parallelism exposure
- Best for cases where individual loops have few iterations


*Option 1: Distribute Outer Loop (C)*

.. code-block:: c

    #pragma omp parallel for
    for (int i = 0; i < 3; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            a[i][j] = expensiveFunc(i,j);
        }
    }


- Distributes the i-loop
- Maximally **3 work packages**

*When to Use*

Use when the outer loop has sufficient iterations for good load balance.


*Option 2: Distribute Inner Loop (C)*

.. code-block:: c

    #pragma omp parallel
    {
        for (int i = 0; i < 3; i++)
        {
            #pragma omp for
            for (int j = 0; j < 4; j++)
            {
                a[i][j] = expensiveFunc(i,j);
            }
        }
    }


- Distributes the j-loop
- Now **four work packages**
- Parallel region before i-loop provides better performance
- Requires ``i`` to be private (automatic for loop variable)
- Starts loop construct 3 times
- May cause more cache line conflicts when writing to ``a``


*Option 3: Collapse Clause (C)*


.. code-block:: c

    #pragma omp parallel
    #pragma omp for collapse(2)
    for (int i = 0; i < 3; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            a[i][j] = expensiveFunc(i,j);
        }
    }



- Use ``collapse`` clause to specify number of loops to collapse
- Available since **OpenMP 3.0**
- Distributes both loops by creating a single combined loop
- Schedules as specified (default in this case)
- Now: **12 work packages**
- May cause more cache line conflicts when writing to ``a``

*Benefits*


- Maximum parallelism exposure
- Best for cases where individual loops have few iterations



Workshare in Fortran
^^^^^^^^^^^^^^^^^^^^

*Workshare Construct*

OpenMP provides the ``workshare`` construct specifically for Fortran.

*Supported Constructs*

This allows distribution of:

- **Fortran array syntax**
  
  .. code-block:: fortran
  
      a(1:n, 1:m) = b(1:n, 1:m) + c(1:n, 1:m)

- **Fortran statements:** ``FORALL``, ``WHERE``

.. note::
   This construct is **Fortran-only** and has no C/C++ equivalent.



*Example: Workshare*


.. code-block:: fortran

    !$OMP PARALLEL SHARED(n, a, b, c)
    !$OMP WORKSHARE
        b(1:n) = b(1:n) + 1
        c(1:n) = c(1:n) + 2
        a(1:n) = b(1:n) + c(1:n)
    !$OMP END WORKSHARE
    !$OMP END PARALLEL



- OpenMP ensures there is no data race
- Arrays ``b`` and ``c`` are ready before assignment to ``a``
- Can include user-defined functions if declared ``ELEMENTAL``



*Scalar Assignment in Workshare*

Shared Scalar (Legal)

.. code-block:: fortran

    REAL :: AA(N,N), BB(N,N), CC(N,N), DD(N,N)
    INTEGER :: SHR
    
    !$OMP PARALLEL SHARED(SHR)
    !$OMP WORKSHARE
        AA = BB
        SHR = 1
        CC = DD * SHR
    !$OMP END WORKSHARE
    !$OMP END PARALLEL

This is **legal OpenMP**. A single thread performs the scalar assignment to ``SHR``.

*Private Scalar (Illegal!)*


.. code-block:: fortran

    REAL :: AA(N,N), BB(N,N), CC(N,N), DD(N,N)
    INTEGER :: PRI
    
    !$OMP PARALLEL PRIVATE(PRI)
    !$OMP WORKSHARE
        AA = BB
        PRI = 1
        CC = DD * PRI
    !$OMP END WORKSHARE
    !$OMP END PARALLEL

.. danger::
   This is **ILLEGAL**!
   
   - Single thread performs scalar assignment to ``PRI``
   - ``PRI`` is undefined on other threads



Sections
^^^^^^^^


*Sections Construct*

The ``sections`` construct allows parallelization when code blocks can be executed independently.


- Initialization of multiple data structures
- Different tasks executing different code

*Considerations*

**Mismatch between blocks and threads:**

- Individual threads might execute multiple code blocks
- Not every thread necessarily gets a code block

**Danger of load imbalance:**

- Code blocks may have different amounts of work
- Mismatch between number of blocks and number of threads

Real-World Application
~~~~~~~~~~~~~~~~~~~~~~

Example from research: "Acceleration of Semiempirical QM/MM methods," JCTC, 13, 3525-3536 (2017)


*Example: Sections Construct (C)*

.. code-block:: c

    #pragma omp parallel shared(a, b, N, M)
    {
        #pragma omp sections
        {
            #pragma omp section
            {
                for (int i = 0; i < N; i++)
                    a[i] = i;
            }
            
            #pragma omp section
            {
                for (int i = 0; i < M; i++)
                    b[i] = initBmatrix(i, M);
            }
        }
    }

*Alternative: Parallel Sections*

.. code-block:: c

    #pragma omp parallel sections shared(a, b, N, M)
    {
        #pragma omp section
        {
            for (int i = 0; i < N; i++)
                a[i] = i;
        }
        
        #pragma omp section
        {
            for (int i = 0; i < M; i++)
                b[i] = initBmatrix(i, M);
        }
    }

.. note::
   The ``parallel sections`` directive combines the ``parallel`` and ``sections`` directives for convenience.



Summary
^^^^^^^

This guide covered the following OpenMP worksharing constructs:

**OpenMP Loop Construct**


- Easy distribution of standard ``do``/``for`` loops
- The ``schedule`` clause deals with many cases of load imbalance
- Schedule types: ``static``, ``dynamic``, ``guided``, ``auto``, ``runtime``
- ``collapse`` clause for nested loops (OpenMP 3.0+)

*OpenMP Workshare Construct (Fortran)*

- Distributes Fortran array syntax statements
- Supports ``FORALL`` and ``WHERE``
- Handles user-defined ``ELEMENTAL`` functions

*OpenMP Sections Construct*


- Distributes independent code blocks on different threads
- Useful for heterogeneous parallel tasks
- Consider load balance issues
