More on Worksharing
-------------------

.. objectives::

    This guide covers advanced worksharing concepts:

    - ``single`` and ``master`` constructs
    - ``if`` clause for conditional parallelization
    - Flushes and implicit barriers
    - ``nowait`` clause for removing barriers
    - Orphan directives



Work for Single Threads
^^^^^^^^^^^^^^^^^^^^^^^

**Single Construct**

The ``single`` construct is a worksharing construct placed inside a parallel region.

As the name suggests, a single thread executes the region.

- **Not specified** which thread executes the region
- Other threads wait at an implicit barrier at the end
- Useful for operations that should be done only once

*Use Cases*

**Guard when writing to shared variables:**

- Enforcing a single write

**Guard I/O operations:**

- Writing to stdout or file (single write)
- Reading from stdin or file (data read once)

**Starting tasks:**

- Task creation (covered later in course)



*Example: single Construct (Fortran)*


.. code-block:: fortran

    !$omp parallel shared(a, b, n) private(i)
        !$omp single
        a = omp_get_num_threads()
        !$omp end single  ! implied barrier, required!
        
        !$omp do
        do i = 1, n
            b(i) = a
        enddo
    !$omp end parallel

.. note::
   The barrier after ``single`` ensures that ``a`` is set before any thread uses it in the loop.



*Example: single Construct (C)*


.. code-block:: c

    #pragma omp parallel shared(a, b, n) private(i)
    {
        #pragma omp single
        {
            a = omp_get_num_threads();
        }  // implied barrier, required!
        
        #pragma omp for
        for (i = 0; i < n; i++)
            b[i] = a;
    }

.. note::
   The barrier after ``single`` ensures that ``a`` is set before any thread uses it in the loop.



**Master Construct**


Similar to ``single``, but with specific differences.

*Key Differences from single*

**Execution:**

- Work is always done on the master thread (thread 0)
- Deterministic behavior

**Synchronization:**

- **No** implied barrier/synchronization
- More lightweight than ``single`` if barrier is not needed

*When to Use*

Use ``master`` when:

- You specifically need thread 0 to do the work
- You don't need synchronization afterward
- Performance is critical and barrier overhead should be avoided



**Ordered Construct**



Execute part of a loop body in sequential order.

.. warning::
   Significant performance penalty! Requires enough other parallel work to pay the overhead.

*How It Works*

1. Thread working on first iteration enters the ordered region, others wait
2. When done, thread for second iteration enters
3. And so on, in sequential order

*Requirements*

- ``ordered`` clause must also be specified on the loop construct (``omp for``/``omp do``)
- No more than one ``ordered`` region per thread and iteration

*Use Cases*

- Ordered printing from parallel loops
- Debugging (e.g., data races)



*Example: Ordered Construct*

.. code-block:: c

    #pragma omp parallel default(none) shared(b)
    {
        #pragma omp for ordered schedule(dynamic, 1)
        for (int i = 0; i < PSIZE; i++)
        {
            b[i] = expensiveFunction(i);
            
            #pragma omp ordered
            printf("b[%3i] = %4i\n", i, b[i]);
        }
    }



- The computation ``expensiveFunction(i)`` happens in parallel
- The ``printf`` statements execute in sequential order (i=0, 1, 2, ...)
- This ensures ordered output despite parallel execution



Clauses for Parallel Construct
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**if Clause**

The ``if`` clause can be specified on the ``parallel`` construct.



If the condition evaluates to false:

- No parallel region is started
- Code executes serially
- Useful for runtime evaluation (e.g., loop count too small to benefit from parallelization)

*Syntax*

.. code-block:: fortran

    !$omp parallel if (condition)

.. code-block:: c

    #pragma omp parallel if (condition)



*Example: if Clause (Fortran)*


.. code-block:: fortran

    integer :: n = 20
    
    !$omp parallel if (n > 5) shared(n)
        !$omp single
        print *, "The n is: ", n
        !$omp end single
        
        print *, "Hello, I am thread", &
                 omp_get_thread_num(), " of", &
                 omp_get_num_threads()
    !$omp end parallel



- If ``n > 5``: parallel region with multiple threads
- If ``n <= 5``: serial execution with single thread



*Example: if Clause (C)*

.. code-block:: c

    int n = 20;
    
    #pragma omp parallel if (n > 5) shared(n)
    {
        #pragma omp single
        printf("The n is %i\n", n);
        
        printf("Hello, I am thread %i of %i\n",
               omp_get_thread_num(),
               omp_get_num_threads());
    }



- If ``n > 5``: parallel region with multiple threads
- If ``n <= 5``: serial execution with single thread



Clause: num_threads
^^^^^^^^^^^^^^^^^^^


The ``num_threads`` clause specifies the number of threads to start in a parallel region.

*Syntax*

**C:**

.. code-block:: c

    int nthread = 3;
    #pragma omp parallel num_threads(nthread)

**Fortran:**

.. code-block:: fortran

    integer :: nthread = 3
    !$omp parallel num_threads(nthread)

.. note::
   This overrides the default thread count and environment variables for this specific parallel region.



Keeping Memory Consistent
^^^^^^^^^^^^^^^^^^^^^^^^^

*OpenMP: Relaxed Memory Model*

OpenMP uses a relaxed memory model for performance.

Threads are allowed to have their "own temporary view" of memory:

- Not required to be consistent with main memory
- Data may be in registers or cache, invisible to other threads

**Programmer Responsibility**

.. important::
   This is a "may be" for the hardware, but the programmer must assume it is (for portability).

*Scope for Data Races*

Without proper synchronization:

- Memory modified by other threads may not be in temporary view
- Own changes may not be visible to other threads



Ensuring Memory Consistency: flush
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


Use ``flush`` to ensure memory consistency across threads.

*What flush Does*

**Writes modifications to memory:**

- Modifications in temporary view are written to memory system
- Guaranteed to be visible to other threads

**Discards temporary view:**

- Temporary view gets discarded
- Next access needs to read from memory subsystem
- Ensures modifications from other threads are "known"

**Prevents reordering:**

- No reordering of memory access and flush



*Example: Without flush (Problem)*


.. code-block:: fortran

    integer :: i
    integer, dimension(4) :: b
    b = (/ 3, 4, 5, 6 /)
    
    !$OMP parallel &
    !$OMP shared(b), private(i)
        i = omp_get_thread_num() + 1
        b(i) = b(i) + i
        b(i+1) = b(i+1) + 1
    !$OMP end parallel

*Memory Behavior (3 threads)*

.. code-block:: text

    Initial:     [3, 4, 5, 6]
    
    Thread 0: i=1
      b(1) = 3 + 1 = 4
      b(2) = 4 + 1 = 5    (but may read stale value!)
    
    Thread 1: i=2
      b(2) = 4 + 2 = 6    (conflict!)
      b(3) = 5 + 1 = 6
    
    Thread 2: i=3
      b(3) = 5 + 3 = 8    (conflict!)
      b(4) = 6 + 1 = 7
    
    Result: [4, 6, 8, 7]  ← Not what we want!

.. warning::
   Without synchronization, threads may read stale values and overwrite each other's changes.



*Example: With barrier (Solution)*

.. code-block:: fortran

    integer :: i
    integer, dimension(4) :: b
    b = (/ 3, 4, 5, 6 /)
    
    !$OMP parallel &
    !$OMP shared(b), private(i)
        i = omp_get_thread_num() + 1
        b(i) = b(i) + i
        !$OMP barrier
        b(i+1) = b(i+1) + 1
    !$OMP end parallel

*Memory Behavior (3 threads)*

.. code-block:: text

    Initial:     [3, 4, 5, 6]
    
    Phase 1 (before barrier):
      Thread 0: b(1) = 4
      Thread 1: b(2) = 6
      Thread 2: b(3) = 8
    
    Result after phase 1: [4, 6, 8, 6]
    
    BARRIER (flush to memory)
    
    Phase 2 (after barrier):
      Thread 0: b(2) = 6 + 1 = 7
      Thread 1: b(3) = 8 + 1 = 9
      Thread 2: b(4) = 6 + 1 = 7
    
    Final result: [4, 7, 9, 7]  ← Correct!

.. note::
   The barrier ensures all writes from phase 1 are visible before phase 2 begins.



**Sequence Required for Data Visibility**

For data to be visible on another thread, the following sequence is required:

1. **First thread writes** to shared memory
2. **First thread flush** - change goes into memory system
3. **Second thread flush** - discard local temporary view
4. **Second thread reads** - gets updated value from memory

*Important Notes*

.. important::
   - A flush doesn't "push" data to other threads
   - Fixing data races typically also requires synchronization
   - Implied flushes are often sufficient

*Explicit Flush*

You can issue an explicit flush:

**Fortran:**

.. code-block:: fortran

    !$OMP flush

**C:**

.. code-block:: c

    #pragma omp flush



*Implicit Barriers and Data Flushes*

OpenMP automatically performs barriers and flushes at specific points.

*Constructs with Barrier and Flush*

**At barrier:**

- ``!$omp barrier`` / ``#pragma omp barrier`` (flush)

**Start and end of constructs:**

- ``parallel`` region (barrier & flush)

**Start and end:**

- ``critical`` region (flush)
- ``ordered`` region (flush)

**End only:**

- Loop constructs (``for``/``do``) (barrier & flush)
- ``single`` (barrier & flush)
- ``workshare`` (barrier & flush)
- ``sections`` (barrier & flush)

.. note::
   **No barrier or flush at the start** of loop, single, workshare, or sections!

**Other Operations**

- Various locking operations (flush)
- Start and end of ``atomic`` flushes "protected" variable
  
  - Use ``seq_cst`` on ``atomic`` to include "global" flush

**No barrier or flush associated with master construct!**



Memory Reorder: Out-of-Order Execution
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*Problem Scenario*

Consider this code:

.. code-block:: fortran

    ...
    A(5) = 3.0
    !$omp atomic write
    matrix_set = 1
    ...

*Potential Problems*

1. **No guarantee A(5) is in memory:**
   
   - Value might still be in registers/cache

2. **No guarantee order is maintained:**
   
   - Optimizing compiler might reorder:
   
   .. code-block:: fortran
   
       matrix_set = 1
       ...
       A(5) = 3.0

.. warning::
   Another thread might see ``matrix_set = 1`` but read an old value of ``A(5)``!



*Fix: Using flush to Prevent Reordering*


.. code-block:: fortran

    ...
    A(5) = 3.0
    !$omp flush
    !$omp atomic write
    matrix_set = 1
    ...

*What the flush Does*

1. **Ensures modified A is in memory:**
   
   - All threads can see the updated value

2. **Prohibits reordering of memory accesses:**
   
   - Compiler and hardware cannot move ``matrix_set = 1`` before the flush
   - Guarantees ``A(5)`` is written before ``matrix_set`` is set



Clause: nowait
^^^^^^^^^^^^^^

Barriers have performance implications. The implied barrier of a construct may not be required for correctness.

*Removing Barriers*


Specifying ``nowait``:

- **In C:** on the construct itself
- **In Fortran:** on the end construct directive

This suppresses the implied barrier (including flush).

*When to Use*

Use ``nowait`` when:

- Threads don't need to wait for each other
- No data dependencies between constructs
- You want to improve performance by allowing threads to continue immediately



*Example: Tensor Product (C)*

.. code-block:: c

    #pragma omp parallel shared(a, b, t, n, m)
    {
        #pragma omp for nowait
        for (int i = 0; i < n; i++)
            a[i] = funcA(i);  // no barrier needed!
        
        #pragma omp for
        for (int j = 0; j < m; j++)
            b[j] = funcB(j);  // barrier needed!
        
        #pragma omp for
        for (int i = 0; i < n; i++)
            for (int j = 0; j < m; j++)
                t[i][j] = a[i] * b[j];  // bad access to b!
    }



- First loop initializes ``a`` with ``nowait`` - threads can continue immediately
- Second loop initializes ``b`` - implicit barrier ensures all threads finish before tensor product
- Third loop uses both ``a`` and ``b`` - needs both to be complete



*Example: Adding Vectors (Fortran)*

.. code-block:: fortran

    !$omp parallel shared(a, b, t, n)
        !$omp do
        do i = 1, n
            a(i) = sin(real(i))
        !$omp end do nowait  ! no barrier here!
        
        !$omp do
        do j = 1, n
            b(j) = cos(real(j))  ! barrier here!
        
        !$omp do
        do i = 1, n
            t(i) = a(i) + b(i)
    !$omp end parallel

.. note::
   Demo code - a single loop would help performance.



- First loop fills ``a`` - can proceed without waiting
- Second loop fills ``b`` - implicit barrier before final loop
- Third loop needs both ``a`` and ``b`` complete



*Example: Adding Vectors (C)*

.. code-block:: c

    #pragma omp parallel shared(a, b, t, n)
    {
        #pragma omp for nowait
        for (int i = 0; i < n; i++)
            a[i] = sin((double)i);  // no barrier here!
        
        #pragma omp for
        for (int j = 0; j < n; j++)
            b[j] = cos((double)j);  // barrier needed!
        
        #pragma omp for
        for (int i = 0; i < n; i++)
            t[i] = a[i] + b[i];
    }

.. note::
   Demo code - a single loop would help performance.



- First loop fills ``a`` - can proceed without waiting
- Second loop fills ``b`` - implicit barrier before final loop
- Third loop needs both ``a`` and ``b`` complete



**Performance Impact of nowait**

Benchmark Setup


**Hardware:**

- Dual socket, quad-core Intel Xeon E5520 (2.26 GHz)

**Compilers tested:**

- PGI 10.9
- GCC 4.4
- Intel 12.0

**Problem:**

- Vector addition example with ``n = 1000``
- Time measured in microseconds (μs)
- Tested with 4, 6, and 8 threads

Results


.. code-block:: text

    Threads    Savings from nowait
    -------    -------------------
    4-8        0.6 - 1.3 μs

Performance Chart
-----------------

.. code-block:: text

    Time (μs)
      40 ┤                                    ■ PGI wait
         │                                    □ PGI nowait
      35 ┤                                    ● GNU wait
         │                                    ○ GNU nowait
      30 ┤                                    ▲ Intel wait
         │                                    △ Intel nowait
      25 ┤     ■
         │     □     ■
      20 ┤     ●     □     ■
         │     ○     ●     □
      15 ┤     ▲     ○     ●
         │     △     ▲     ○
      10 ┤           △     ▲
         │                 △
       0 └─────┴─────┴─────┴─────
            4     6     8   Threads

.. note::
   Even small savings (0.6-1.3 μs) can add up in frequently executed code.



Specialty of Static Schedule
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When specifying a static schedule with:

- Same iteration count
- Same chunk size (or default)
- Loops bound to same parallel region

**Guarantee:**

You can safely assume the same thread works on the same iteration in all loops.



Can use ``nowait`` even with data dependencies between loops!

.. important::
   This only works with **static** scheduling. Other schedules don't guarantee iteration-to-thread mapping.



*Example: Static Schedule with Dependencies (Fortran)*

.. code-block:: fortran

    !$omp parallel shared(a, b, t, n)
        !$omp do schedule(static)
        do i = 1, n
            a(i) = sin(real(i))
        !$omp end do nowait  ! no barrier here!
        
        !$omp do schedule(static)
        do j = 1, n
            b(j) = cos(real(j))
        !$omp end do nowait  ! no barrier here!
        
        !$omp do schedule(static)
        do i = 1, n
            t(i) = a(i) + b(i)
        !$omp end do nowait  ! no barrier here!
    !$omp end parallel

.. important::
   The static schedule is crucial! Each thread processes the same indices in all three loops.



*Example: Static Schedule with Dependencies (C)*

.. code-block:: c

    #pragma omp parallel shared(a, b, t, n)
    {
        #pragma omp for schedule(static) nowait
        for (int i = 0; i < n; i++)
            a[i] = sin((double)i);  // no barrier here!
        
        #pragma omp for schedule(static) nowait
        for (int j = 0; j < n; j++)
            b[j] = cos((double)j);  // no barrier here!
        
        #pragma omp for schedule(static)
        for (int i = 0; i < n; i++)
            t[i] = a[i] + b[i];
    }

.. important::
   The static schedule is crucial! Each thread processes the same indices in all three loops.

*Why This Works*

With static scheduling:

- Thread 0 always processes indices 0 to n/num_threads-1
- Thread 1 always processes indices n/num_threads to 2*n/num_threads-1
- And so on...

Each thread only reads values it wrote, so no race conditions occur!



Orphan Directives
^^^^^^^^^^^^^^^^^


"Orphan" directives are OpenMP directives that appear inside functions/subroutines called from within a parallel region, rather than directly inside the parallel region.

*Thread Safety Assumption*

Calling subroutines and functions inside a parallel region is legal, assuming thread safety.

*What Can Be Orphaned*

The called procedures may contain:

- Worksharing constructs (``for``, ``do``, ``sections``)
- Synchronization constructs (``barrier``, ``critical``, etc.)


*Example: Orphan Directive (C)*

Main Function


.. code-block:: c

    #pragma omp parallel shared(v, vl) reduction(+:nm)
    {
        vectorinit(v, vl);
        nm = vectornorm(v, vl);
    }

Called Function with Orphan Directive


.. code-block:: c

    void vectorinit(double* vdata, int leng)
    {
        #pragma omp for
        for (int i = 0; i < leng; i++)
        {
            vdata[i] = i;
        }
        return;
    }

.. note::
   The ``#pragma omp for`` directive is "orphaned" - it's not directly inside the parallel region but binds to the active parallel region when called.



*Example: Orphan Directive (Fortran)*

Main Program


.. code-block:: fortran

    !$omp parallel shared(v, vl) reduction(+:nm)
        call vectorinit(v, vl)
        nm = vectornorm(v, vl)
    !$omp end parallel

Subroutine with Orphan Directive


.. code-block:: fortran

    subroutine vectorinit(vdata, leng)
        double precision, dimension(leng) :: vdata
        integer :: leng, i
        
        !$omp do
        do i = 1, leng
            vdata(i) = i
        enddo
    end subroutine vectorinit

.. note::
   The ``!$omp do`` directive is "orphaned" - it's not directly inside the parallel region but binds to the active parallel region when called.


**Performance Impact of Orphaning**


Benchmark Setup


**Test:** Vector initialization and norm calculation
**Vector length:** 40,000
**Hardware:** Xeon E5-2650 v3
**Compilers:** GCC 4.9.3, ICC 16.0

Configurations Tested


1. ``parallel for`` in each function (no orphaning)
2. Orphaned ``for`` in each function
3. Orphaned ``for nowait`` in each function

Results


.. code-block:: text

    Time (ms)
    0.06 ┤
         │                          ■ gcc: parallel for
    0.05 ┤                          □ gcc: orphaned for
         │                          ○ gcc: orphaned for nowait
    0.04 ┤  ■                       ● icc: parallel for
         │     ■                    ▲ icc: orphaned for
    0.03 ┤        □                 △ icc: orphaned for nowait
         │        ■  □
    0.02 ┤           ○  ■  □  ○
         │              ●  ▲  △
    0.01 ┤
         │
       0 └─────┴─────┴─────┴─────┴─────
            2     4     6     8    10  Cores

Key Observations


- Orphaned directives perform **better** than creating new parallel regions
- Using ``nowait`` provides additional performance gains
- Starting/closing parallel regions is very expensive



*Discussion of Orphan Directives*

Advantages:

**Reduces need for code restructuring:**

- Can parallelize existing functions without major changes

**Allows for longer parallel regions:**

- Starting/closing parallel regions is very expensive
- One long parallel region is more efficient than many short ones

**Better performance:**

- As shown in benchmarks, avoids parallel region overhead

Potential Issues:

.. warning::
   **Problem:** Routine with orphan directive called outside parallel region
   
   If a function with an orphaned directive is called from serial code, the directive may have no effect or cause unexpected behavior.

Best Practices:

- Document functions that contain orphan directives
- Consider adding checks for parallel context if needed
- Design functions to work correctly both inside and outside parallel regions


Summary
^^^^^^^

This guide covered advanced worksharing concepts in OpenMP:

**Constructs**

- **single construct:** Execute code on one thread (with barrier)
- **master construct:** Execute code on master thread (no barrier)
- **ordered construct:** Execute loop iterations in sequential order

**Clauses**

- **if clause:** Conditional parallelization
- **num_threads clause:** Control thread count
- **nowait clause:** Remove implicit barriers for performance

**Memory Consistency**


- **flush:** Ensure memory consistency across threads
- **Implicit barriers and flushes:** Automatic synchronization points
- **Memory reordering:** Understanding and preventing issues

**Advanced Techniques**

- **Static schedule specialty:** Using nowait with dependencies
- **Orphan directives:** Worksharing constructs in called functions

**Performance Considerations**

- Balance between synchronization overhead and correctness
- Strategic use of ``nowait`` can improve performance
- Orphan directives reduce parallel region overhead
