Worksharing and Scheduling
==========================

.. objectives::
    
    - Loop construct 
    - Workshare construct 
    - Section construct

Worksharing constructs distribute work among the threads of a team.

- The **loop worksharing construct** distributes iterations of a loop and can
  use the ``schedule`` clause to reduce load imbalance.
- The **workshare construct** distributes Fortran array operations. It has no
  direct C/C++ equivalent.
- The **sections construct** distributes independent blocks of code. OpenMP
  tasks are another option for expressing independent work.

Distributing Loops
------------------

Distributing large loops is a typical target for OpenMP parallelization.

**Traditional approach**

Manual loop distribution requires the programmer to calculate the number of
iterations assigned to each thread, the starting index, and the final index.
Special care is needed when the iteration count is not divisible by the
number of threads.

**OpenMP loop construct**

A worksharing loop simplifies the code by assigning iterations to threads.
It automatically handles the iteration distribution, makes the associated
loop iteration variable private, and supports different scheduling policies.
An implicit barrier occurs at the end unless ``nowait`` is specified.
There is no implicit barrier at the beginning of the loop construct.

**Basic syntax**

.. tabs::

   .. tab:: C/C++

      The ``for`` worksharing directive applies to an eligible canonical
      ``for`` loop. The initialization, comparison, and increment must follow
      the OpenMP loop requirements, and loop bounds must be invariant during
      execution.

      .. code-block:: c

         #pragma omp parallel
         {
             #pragma omp for
             for (int i = 0; i < N; ++i) {
                 /* loop body */
             }
         }

      Supported canonical forms include integer or pointer-based induction
      variables (and random-access iterators in C++), appropriate comparison
      operators, and permitted increments.

   .. tab:: Fortran

      The ``do`` worksharing directive applies to a counted ``do`` loop;
      an uncontrolled ``do`` or ``do while`` loop is not supported by this
      form of the construct.

      .. code-block:: fortran

         !$omp parallel
         !$omp do
         do i = 1, N
             ! loop body
         end do
         !$omp end do
         !$omp end parallel

      The ``!$omp end do`` directive is optional when no end-directive
      clauses are needed.

**Example: Vector Norm**

The vector norm is :math:`\sqrt{\sum_i v_i^2}`. The next two approaches
calculate it by first accumulating the sum of squares and then taking the
square root. These are *code fragments*: assume the arrays and scalar
variables have been declared, and that the OpenMP runtime has been made
available.

**Manual loop management**

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         norm = 0.0;
         #pragma omp parallel default(none) shared(vect, vleng, norm)
         {
             int tid = omp_get_thread_num();
             int nthreads = omp_get_num_threads();
             int begin = (tid * vleng) / nthreads;
             int end = ((tid + 1) * vleng) / nthreads;
             double local_norm = 0.0;

             for (int i = begin; i < end; ++i)
                 local_norm += vect[i] * vect[i];

             #pragma omp atomic update
             norm += local_norm;
         }
         norm = sqrt(norm);

   .. tab:: Fortran

      .. code-block:: fortran

         norm = 0.0d0
         !$omp parallel default(none) shared(vect, vleng, norm) &
         !$omp private(tid, nthreads, first, last, i, local_norm)
             tid = omp_get_thread_num()
             nthreads = omp_get_num_threads()
             first = 1 + (tid * vleng) / nthreads
             last = ((tid + 1) * vleng) / nthreads
             local_norm = 0.0d0

             do i = first, last
                 local_norm = local_norm + vect(i) * vect(i)
             end do

             !$omp atomic update
             norm = norm + local_norm
         !$omp end parallel
         norm = sqrt(norm)

.. note::

   This approach explicitly calculates each thread's range. The example
   handles a vector length that is not divisible by the number of threads,
   rather than dropping the remainder.

**Using the OpenMP loop construct**

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         norm = 0.0;
         #pragma omp parallel default(none) shared(vect, vleng, norm)
         {
             double local_norm = 0.0;

             #pragma omp for
             for (int i = 0; i < vleng; ++i)
                 local_norm += vect[i] * vect[i];

             #pragma omp atomic update
             norm += local_norm;
         }
         norm = sqrt(norm);

   .. tab:: Fortran

      .. code-block:: fortran

         norm = 0.0d0
         !$omp parallel default(none) shared(vect, vleng, norm) &
         !$omp private(i, local_norm)
             local_norm = 0.0d0

             !$omp do
             do i = 1, vleng
                 local_norm = local_norm + vect(i) * vect(i)
             end do
             !$omp end do

             !$omp atomic update
             norm = norm + local_norm
         !$omp end parallel
         norm = sqrt(norm)

.. important::

   The loop bounds now match the serial code, while OpenMP handles iteration
   distribution. The accumulation uses a thread-local partial sum and an
   atomic update of the global sum. A ``reduction`` clause is another common
   approach to this operation.

Parallel Loop Construct
-----------------------

When a parallel region contains only a worksharing loop, OpenMP provides a
combined ``parallel for`` (C/C++) or ``parallel do`` (Fortran) construct.

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         #pragma omp parallel for
         for (int i = 0; i < N; ++i) {
             /* loop body */
         }

   .. tab:: Fortran

      .. code-block:: fortran

         !$omp parallel do
         do i = 1, N
             ! loop body
         end do
         !$omp end parallel do

      ``!$omp end parallel do`` can be omitted if it has no required
      end-directive clauses.

The rules of both the parallel region and the worksharing loop apply to
the combined construct.

Loop Reordering and Data Dependencies
-------------------------------------

Iterations of a parallel loop are not guaranteed to execute in serial order.
The result is correct only if loop-carried dependencies are absent, or if
those dependencies are handled explicitly.

When dependencies exist, possible approaches are to change the algorithm,
protect or order dependent operations with suitable OpenMP constructs, or
keep the loop serial.

**Example: Loop-carried dependency**

.. tabs::

   .. tab:: C/C++

      The current iteration depends on the previous iteration:

      .. code-block:: c

         a[0] = 0;
         for (int i = 1; i < N; ++i)
             a[i] = a[i - 1] + i;

      An independent closed-form computation avoids that dependency:

      .. code-block:: c

         for (int i = 0; i < N; ++i)
             a[i] = i * (i + 1) / 2;

   .. tab:: Fortran

      The current iteration depends on the previous iteration:

      .. code-block:: fortran

         a(1) = 0
         do i = 2, N
             a(i) = a(i - 1) + (i - 1)
         end do

      An independent closed-form computation avoids that dependency:

      .. code-block:: fortran

         do i = 1, N
             a(i) = (i - 1) * i / 2
         end do

.. warning::

   Verify that iterations are independent before adding an OpenMP
   worksharing directive. In the original example, simply adding a
   ``parallel for`` or ``parallel do`` would be incorrect.

Scheduling Loop Iterations
--------------------------

Previous examples assumed approximately equal work per iteration, but
real-world workloads can vary substantially. Some threads may finish well
before others, reducing performance.

**Example: A triangular iteration space**

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         for (int i = 0; i < N; ++i) {
             for (int j = 0; j <= i; ++j) {
                 /* work */
             }
         }

   .. tab:: Fortran

      .. code-block:: fortran

         do i = 1, N
             do j = 1, i
                 ! work
             end do
         end do

The later outer iterations do more work. Similar imbalances arise when a
loop body runs until a requested accuracy is reached. Handling uneven
workloads is often simpler in shared-memory than distributed-memory code.

**The schedule clause**

The ``schedule(kind [, chunk_size])`` clause controls how loop iterations
are assigned to threads. Common schedule kinds include ``static``,
``dynamic``, ``guided``, ``auto``, and ``runtime``. If no schedule clause is
provided, the exact policy is implementation dependent.

Static Scheduling
~~~~~~~~~~~~~~~~~

With ``static`` scheduling, iterations are assigned to threads before loop
execution. If a chunk size is specified, the chunks are assigned to threads
in round-robin order. The last chunk can be smaller than the others.
Without an explicit chunk size, implementations typically assign roughly
contiguous blocks. Static scheduling usually has little runtime overhead.

For illustration, with four threads and ``schedule(static, 2)``, successive
two-iteration chunks are assigned to threads 0, 1, 2, 3, 0, 1, etc.

**Example: Triangular matrix operation, static schedule**

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         #pragma omp parallel for shared(a) schedule(static, 100)
         for (int j = 0; j < 1200; ++j) {
             for (int i = j + 1; i < 1200; ++i) {
                 a[i][j] = func(i, j);
                 a[j][i] = -a[i][j];
             }
         }

   .. tab:: Fortran

      .. code-block:: fortran

         !$omp parallel do private(i) shared(a) schedule(static, 100)
         do j = 1, 1200
             do i = j + 1, 1200
                 a(i,j) = func(i,j)
                 a(j,i) = -a(i,j)
             end do
         end do
         !$omp end parallel do

For this triangular workload, smaller chunks can improve balance, at the
cost of more scheduling overhead. It shows a
maximum work share of ``7/16`` for a default static distribution against
``5/16`` for a chunk size of 100; these are illustrative workload estimates,
not general guarantees for every thread count or scheduling implementation.

Dynamic Scheduling
~~~~~~~~~~~~~~~~~~

With ``dynamic`` scheduling, threads request more iterations as they finish
their current chunks. Its default chunk size is one iteration. It often
balances unpredictable work better than static scheduling, but incurs more
runtime overhead.

**Example: Triangular matrix operation, dynamic schedule**

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         #pragma omp parallel for shared(a) schedule(dynamic, 100)
         for (int j = 0; j < 1200; ++j) {
             for (int i = j + 1; i < 1200; ++i) {
                 a[i][j] = func(i, j);
                 a[j][i] = -a[i][j];
             }
         }

   .. tab:: Fortran

      .. code-block:: fortran

         !$omp parallel do private(i) shared(a) schedule(dynamic, 100)
         do j = 1, 1200
             do i = j + 1, 1200
                 a(i,j) = func(i,j)
                 a(j,i) = -a(i,j)
             end do
         end do
         !$omp end parallel do

For the triangular example, the course estimates that the largest
thread share falls to about ``0.27`` with ``schedule(dynamic, 100)``.
Dynamic scheduling can improve load balance but has more dispatch overhead.

Guided Scheduling
~~~~~~~~~~~~~~~~~

``guided`` is similar to ``dynamic``, but the chunk sizes generally start
larger and decrease as work is assigned. The amount of remaining work and
the number of threads influence the size of a new chunk. The specified
chunk size is a lower bound (except for a possible final chunk), and the
default is one iteration.

The goal is to reduce scheduling overhead early while still improving load
balance toward the end of the loop.

Auto and Runtime Schedules
~~~~~~~~~~~~~~~~~~~~~~~~~

With ``schedule(auto)``, the OpenMP implementation chooses the scheduling
policy. With ``schedule(runtime)``, the policy is chosen at runtime, for
example through an API call or the ``OMP_SCHEDULE`` environment variable.

.. tabs::

   .. tab:: C/C++

      **Auto schedule**

      .. code-block:: c

         #pragma omp parallel for schedule(auto)
         for (int i = 0; i < N; ++i) {
             /* work */
         }

      **Runtime schedule**

      .. code-block:: c

         #include <omp.h>

         omp_set_schedule(omp_sched_static, 10);
         #pragma omp parallel for schedule(runtime)
         for (int i = 0; i < N; ++i) {
             /* work */
         }

   .. tab:: Fortran

      **Auto schedule**

      .. code-block:: fortran

         !$omp parallel do schedule(auto)
         do i = 1, N
             ! work
         end do
         !$omp end parallel do

      **Runtime schedule** (with ``use omp_lib`` in the containing scope)

      .. code-block:: fortran

         call omp_set_schedule(omp_sched_static, 10)
         !$omp parallel do schedule(runtime)
         do i = 1, N
             ! work
         end do
         !$omp end parallel do

**Runtime schedule environment variables**

For Bash:

.. code-block:: bash

   export OMP_SCHEDULE="guided,4"

For C-shell:

.. code-block:: csh

   setenv OMP_SCHEDULE "guided,4"

.. warning::

   Do not specify a ``chunk_size`` on a ``schedule(auto)`` or
   ``schedule(runtime)`` directive. For ``runtime``, the chunk size is
   instead supplied by the runtime setting.

Multiple Loop Parallelization
-----------------------------

Consider two nested loops with 3 outer iterations and 4 inner iterations.
Three common choices are to distribute the outer loop, distribute the inner
loop, or collapse them into one larger iteration space.

**Serial nested loop**

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         for (int i = 0; i < 3; ++i) {
             for (int j = 0; j < 4; ++j) {
                 a[i][j] = expensiveFunc(i, j);
             }
         }

   .. tab:: Fortran

      .. code-block:: fortran

         do j = 1, 3
             do i = 1, 4
                 a(i,j) = expensiveFunc(i,j)
             end do
         end do

Option 1: Distribute the Outer Loop
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         #pragma omp parallel for
         for (int i = 0; i < 3; ++i) {
             for (int j = 0; j < 4; ++j) {
                 a[i][j] = expensiveFunc(i, j);
             }
         }

   .. tab:: Fortran

      .. code-block:: fortran

         !$omp parallel do
         do j = 1, 3
             do i = 1, 4
                 a(i,j) = expensiveFunc(i,j)
             end do
         end do
         !$omp end parallel do

This distributes the outer loop, exposing at most three outer-loop
iterations for worksharing. It works well when the outer loop has enough
iterations for good load balance.

Option 2: Distribute the Inner Loop
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         #pragma omp parallel
         {
             for (int i = 0; i < 3; ++i) {
                 #pragma omp for
                 for (int j = 0; j < 4; ++j) {
                     a[i][j] = expensiveFunc(i, j);
                 }
             }
         }

   .. tab:: Fortran

      .. code-block:: fortran

         !$omp parallel private(j)
         do j = 1, 3
             !$omp do
             do i = 1, 4
                 a(i,j) = expensiveFunc(i,j)
             end do
             !$omp end do
         end do
         !$omp end parallel

This distributes the inner loop, exposing four iterations per outer-loop
step. The worksharing directive is encountered three times, and each
encounter has a synchronization cost unless ``nowait`` is used appropriately.
Depending on data layout, it can also increase cache-line contention.

.. note::

   Here, the outer loop is executed by every thread in the team, while the
   inner loop's iterations are shared. The outer-loop index must be
   thread-private (as written in the examples).

Option 3: Collapse Two Loops
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         #pragma omp parallel for collapse(2)
         for (int i = 0; i < 3; ++i) {
             for (int j = 0; j < 4; ++j) {
                 a[i][j] = expensiveFunc(i, j);
             }
         }

   .. tab:: Fortran

      .. code-block:: fortran

         !$omp parallel do collapse(2)
         do j = 1, 3
             do i = 1, 4
                 a(i,j) = expensiveFunc(i,j)
             end do
         end do
         !$omp end parallel do

``collapse(2)`` makes both loops one logical iteration space (12
iterations in this example). The clause has been available since OpenMP
3.0. This can expose more parallelism when the individual loops are short,
but memory-access patterns and cache contention still matter.

Workshare in Fortran
--------------------

The OpenMP ``workshare`` construct is **Fortran-only**. It can distribute
Fortran array syntax, ``FORALL``, ``WHERE``, and permitted scalar
assignments. There is **no direct equivalent** directive in C/C++. The
following tabs show a related *element-wise parallel loop* in C/C++ and the
``workshare`` construct in Fortran; the two mechanisms are not
identical.

**Example: Updating arrays**

.. tabs::

   .. tab:: C/C++

      **Analogous array-element loop, not an OpenMP workshare directive**

      .. code-block:: c

         #pragma omp parallel for
         for (int i = 0; i < n; ++i) {
             b[i] = b[i] + 1;
             c[i] = c[i] + 2;
             a[i] = b[i] + c[i];
         }

      This is valid for element-wise independent operations, but does not
      provide the general semantics of Fortran ``workshare``.

   .. tab:: Fortran

      .. code-block:: fortran

         !$omp parallel shared(n, a, b, c)
         !$omp workshare
             b(1:n) = b(1:n) + 1
             c(1:n) = c(1:n) + 2
             a(1:n) = b(1:n) + c(1:n)
         !$omp end workshare
         !$omp end parallel

      The implementation must preserve the dependencies between these
      statements, so the array assignments have their normal Fortran
      meaning. User-defined functions in the ``workshare`` region are
      subject to OpenMP restrictions, for example requiring elemental
      functions where applicable.

.. note::

   A ``workshare`` region has an implicit barrier at the end unless its
   ``end workshare`` directive specifies ``nowait``.

Scalar Assignments in Fortran Workshare
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This subsection is specific to Fortran. A scalar assignment is performed as
one unit of work, which can be executed by a single participating thread.

**Shared scalar (valid)**

.. code-block:: fortran

   real :: AA(N,N), BB(N,N), CC(N,N), DD(N,N)
   integer :: SHR

   !$omp parallel shared(SHR)
   !$omp workshare
       AA = BB
       SHR = 1
       CC = DD * SHR
   !$omp end workshare
   !$omp end parallel

This is legal when ``SHR`` is shared: all threads refer to the same scalar,
and the implementation preserves the necessary statement dependencies.

**Private scalar (unsafe / unspecified result)**

.. code-block:: fortran

   real :: AA(N,N), BB(N,N), CC(N,N), DD(N,N)
   integer :: PRI

   !$omp parallel private(PRI)
   !$omp workshare
       AA = BB
       PRI = 1
       CC = DD * PRI
   !$omp end workshare
   !$omp end parallel

.. danger::

   Do not write the second version. A scalar assignment may be performed
   by only one thread, so the values of private copies belonging to other
   threads are not established. The result is unspecified, not a safe
   parallel assignment.

Sections
--------

The ``sections`` construct distributes independent blocks of code among
threads. Typical uses include initialization of different data structures
or execution of different independent operations. A thread may execute
multiple sections, and some threads may execute none.

**Potential load imbalance**

The sections can require different amounts of work, and their count may
not match the number of threads. There is an implicit barrier at the end
of ``sections`` unless ``nowait`` is specified.

A research application is given in
*Acceleration of Semiempirical QM/MM Methods*, JCTC 13, 3525--3536 (2017).

**Example: Two independent sections**

.. tabs::

   .. tab:: C/C++

      .. code-block:: c

         #pragma omp parallel shared(a, b, N, M)
         {
             #pragma omp sections
             {
                 #pragma omp section
                 {
                     for (int i = 0; i < N; ++i)
                         a[i] = i;
                 }

                 #pragma omp section
                 {
                     for (int i = 0; i < M; ++i)
                         b[i] = initBmatrix(i, M);
                 }
             }
         }

      **Alternative: combined parallel sections**

      .. code-block:: c

         #pragma omp parallel sections shared(a, b, N, M)
         {
             #pragma omp section
             {
                 for (int i = 0; i < N; ++i)
                     a[i] = i;
             }

             #pragma omp section
             {
                 for (int i = 0; i < M; ++i)
                     b[i] = initBmatrix(i, M);
             }
         }

   .. tab:: Fortran

      .. code-block:: fortran

         !$omp parallel shared(a, b, N, M) private(i)
         !$omp sections
         !$omp section
             do i = 1, N
                 a(i) = i
             end do
         !$omp section
             do i = 1, M
                 b(i) = initBmatrix(i, M)
             end do
         !$omp end sections
         !$omp end parallel

      **Alternative: combined parallel sections**

      .. code-block:: fortran

         !$omp parallel sections shared(a, b, N, M) private(i)
         !$omp section
             do i = 1, N
                 a(i) = i
             end do
         !$omp section
             do i = 1, M
                 b(i) = initBmatrix(i, M)
             end do
         !$omp end parallel sections

.. note::

   ``parallel sections`` combines the creation of a parallel region and
   the distribution of independent sections into one construct. In the
   Fortran examples, the section loop variable is explicitly private to
   avoid concurrent writes to a shared loop index.


.. challenge::

    Parallelize a **for** loop which has 20 iterations by evenly divinding the number of interations 
    among the available threads. Then use a variable *var1* to store the number of iterations in the loop.
    **Hint** an *atomic* operation can help to protect *var1*. 

.. solution::

    .. code-block:: c
        :linenos:

        // On cluster Kebnekaise
        // ml foss
        // export OMP_NUM_THREADS=1 
        // gcc -O3 -march=native -fopenmp -o test.x 6a-forworksharing-openmp.c -lm 
        #include <stdio.h>
        #ifdef _OPENMP
        #include <omp.h>
        #endif

        int main()
        {

        int i,var1,var2,myWork;  
        int n = 20; // number of iterations
        var1 = 0;

        #pragma omp parallel private(myWork, var2) shared(var1)
            {
        #ifdef _OPENMP
            // The purpose of this code is to add 1 to var1 20 times
            myWork = n/omp_get_num_threads();

            for ( int i = myWork*omp_get_thread_num(); i < myWork*(1+omp_get_thread_num()); i++)
                var2 += 1;

            #pragma omp atomic update
            var1 += var2;
                
        #else
            printf("Serial code!\n");
        #endif
            }
        
            printf("var1 =  %i \n", var1);

        return 0;
        }

.. challenge::

    Parallelize the previous **for** loop code but this time using the loop construct.
    **Hint** an *atomic* operation can help to protect *var2*. 
    
.. solution::

    .. code-block:: c
        :linenos:

        // On cluster Kebnekaise
        // ml foss
        // export OMP_NUM_THREADS=1 
        // gcc -O3 -march=native -fopenmp -o test.x 6b-forworksharing-openmp.c -lm 
        #include <stdio.h>
        #ifdef _OPENMP
        #include <omp.h>
        #endif

        int main()
        {

        int i,var1;  
        int n = 20; // number of iterations
        var1 = 0;

        #pragma omp parallel 
            {
        #ifdef _OPENMP
            // The purpose of this code is to add 1 to var1 20 times
        #pragma omp for 
            for ( int i = 0; i < n; i++)
                #pragma omp atomic update
                var1 += 1;
                
        #else
            printf("Serial code!\n");
        #endif
            }
        
            printf("var1 =  %i \n", var1);

        return 0;
        }

.. challenge::

    Use a critical region to protect a counter variable in a parallel loop that counts the stores the
    number of iterations. Is this loop parallelization efficient? Why?

.. solution::

    .. code-block:: c
        :linenos:

        // On cluster Kebnekaise
        // ml foss
        // export OMP_NUM_THREADS=1 
        // gcc -O3 -march=native -fopenmp -o test.x 8-critical-openmp.c -lm 
        #include <stdio.h>
        #ifdef _OPENMP
        #include <omp.h>
        #endif

        int main()
        {

        int sum = 0;

        #pragma omp parallel for
            
        #ifdef _OPENMP
            for ( int i = 0; i < 10; i++) {
                #pragma omp critical
                {
                    sum += i;
                    printf("Thread %d added %d to sum, current sum = %d\n", omp_get_thread_num(), i, sum);
                }
            }
        #else
            printf("Serial code!\n");
        #endif
        
            printf("Final sum = %d\n", sum);

        return 0;
        }

.. challenge::

    Create a Section in a parallel region to protect two blocks of code where the thread ID
    is printed out in each block.

.. solution::

    .. code-block:: c
        :linenos:

        // On cluster Kebnekaise
        // ml foss
        // export OMP_NUM_THREADS=1 
        // gcc -O3 -march=native -fopenmp -o test.x 6c-sectionsworksharing-openmp.c -lm 
        #include <stdio.h>
        #include <omp.h>

        int main() {
            #pragma omp parallel num_threads(8)   // We have a total of 8 threads
            {
                #pragma omp sections
                {
                    #pragma omp section
                    {
                        printf("Section 1 executed by thread %d\n", omp_get_thread_num());
                    }
                    #pragma omp section
                    {
                        printf("Section 2 executed by thread %d\n", omp_get_thread_num());
                    }
                }
            }
            return 0;
        }

.. challenge::

    Create two nested parallel region with 2 threads each and print out the thread IDs for the
    two inner and the outer threads.

.. solution::

    .. code-block:: c
        :linenos:

        // On cluster Kebnekaise
        // ml foss
        // export OMP_NUM_THREADS=1 
        // gcc -O3 -march=native -fopenmp -o test.x 11-nested-openmp.c -lm 
        #include <stdio.h>
        #include <omp.h>

        int main() {
            // enabling/disabling nested parallelism
            omp_set_nested(1);

            #pragma omp parallel num_threads(2)
            {
                int outer_tid = omp_get_thread_num();
                printf("Outer thread %d in outer parallel region\n", outer_tid);

                #pragma omp parallel num_threads(2)
                {
                    int inner_tid = omp_get_thread_num();
                    printf("    Inner thread %d in inner parallel region but outer thread %d\n", inner_tid, outer_tid);
                }
            }

            return 0;
        }



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
