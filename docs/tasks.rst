===============================
OpenMP Task Programming
===============================

Overview
========

OpenMP tasks provide a flexible way to parallelize irregular algorithms that don't fit the regular loop-based paradigm, including recursive algorithms, linked lists, while loops, and divide-and-conquer approaches.

Task Construct
==============

The ``task`` construct creates explicit tasks from code blocks and their data environments.

Fortran Syntax
--------------

.. code-block:: fortran

   !$omp task [clauses]
      code body
   !$omp end task

C/C++ Syntax
------------

.. code-block:: c

   #pragma omp task [clauses]
   {
      code body
   }

Key Characteristics
-------------------

- Tasks are created inside parallel regions
- Execution can be immediate or deferred
- Tasks can be executed by the encountering thread or other threads
- Provides flexibility for irregular parallel patterns

Data Sharing Attributes
======================

Allowed Attributes
------------------

- **private**: Data is private to the task
- **firstprivate**: Data is private and initialized when task is created
- **shared**: Data is shared (primary way to return results)
- **default**: 
  - Fortran: shared | private | firstprivate
  - C/C++: shared | none

Default Data Sharing
--------------------

When no default is declared:

- If variable is shared by **all** implicit tasks in current team: **shared**
- Otherwise: **firstprivate**

**Recommendation**: Use ``default(none)`` for explicit control.

Task Synchronization
====================

Taskwait
--------

Waits for completion of child tasks (not grandchildren):

.. code-block:: fortran

   !$omp taskwait

.. code-block:: c

   #pragma omp taskwait

Barrier
-------

Waits for all tasks in the innermost parallel region:

.. code-block:: fortran

   !$omp barrier

.. code-block:: c

   #pragma omp barrier

Taskyield
---------

Allows suspension of current task to execute different tasks:

.. code-block:: fortran

   !$omp taskyield

.. code-block:: c

   #pragma omp taskyield

Taskgroup (OpenMP 4.0)
-----------------------

Waits for all descendant tasks (including grandchildren):

Fortran:
~~~~~~~~

.. code-block:: fortran

   !$omp taskgroup
   do i=1, n
      !$omp task ...
      call processing(...)
      !$omp end task
   end do
   !$omp end taskgroup

C/C++:
~~~~~~~

.. code-block:: c

   #pragma omp taskgroup
   {
      for (int i=0; i<n; i++)
      {
         #pragma omp task ...
         {
            processing(...);
         }
      }
   }

Task Control Clauses
====================

If Clause
---------

Controls task creation based on work amount:

.. code-block:: fortran

   !$OMP task if(level .lt. 10)

If expression evaluates to false, the encountering thread executes the code body directly as an included task.

Final Clause
------------

Marks tasks as final, causing all encountered tasks to be included and final:

.. code-block:: fortran

   !$OMP task final(level .gt. 30)

Mergeable Clause
----------------

Allows implementation to optimize by reusing the generating task's data environment:

.. code-block:: fortran

   !$omp task mergeable

.. code-block:: c

   #pragma omp task mergeable

Task Scheduling Points
======================

Threads may switch tasks at:

- Immediately after explicit task generation
- After task completion
- At ``taskwait``, ``taskyield``
- At barrier (explicit or implicit)
- At end of ``taskgroup``

**Warning**: Untied tasks (not covered) may switch at any point, potentially causing deadlocks in critical regions.

Case Study 1: Fibonacci Numbers
===============================

Mathematical Definition
-----------------------

- \( F_0 = 0 \)
- \( F_1 = 1 \) 
- \( F_n = F_{n-1} + F_{n-2} \)

Serial Implementation
---------------------

.. code-block:: fortran

   recursive function recursive_fib(in) result(fibnum)
      integer, intent(in) :: in
      integer(lint) :: fibnum, sub1, sub2
      if (in .gt. 1) then
         sub1 = recursive_fib(in - 1)
         sub2 = recursive_fib(in - 2)
         fibnum = sub1 + sub2
      else
         fibnum = in
      endif
   end function recursive_fib

Parallel Implementation
-----------------------

.. code-block:: fortran

   recursive function recursive_fib(in) result(fibnum)
      integer, intent(in) :: in
      integer(lint) :: fibnum, sub1, sub2
      if (in .gt. 1) then
         !$OMP task shared(sub1) firstprivate(in)
         sub1 = recursive_fib(in - 1)
         !$OMP end task
         !$OMP task shared(sub2) firstprivate(in)
         sub2 = recursive_fib(in - 2)
         !$OMP end task
         !$OMP taskwait
         fibnum = sub1 + sub2
      else
         fibnum = in
      endif
   end function recursive_fib

Proper Calling Code
-------------------

.. code-block:: fortran

   program fibonacci
      !$ use omp_lib
      integer, parameter :: lint = selected_int_kind(10)
      integer(lint) :: fibres
      integer :: input
      read (*,*) input
      !$OMP parallel shared(input, fibres) default(none)
         !$OMP single
         fibres = recursive_fib(input)
         !$OMP end single
      !$OMP end parallel
      print *, "Fibonacci number", input," is:", fibres
   end program fibonacci

Performance Insights
-------------------

- Naïve implementation (2 tasks per iteration) shows poor performance
- Using ``if`` clause to limit task creation for small inputs improves performance
- Creating only 1 task per iteration helps more
- Too little work per task leads to overhead domination

Case Study 2: Self-Refining Recursive Integrator
================================================

Algorithm Overview
------------------

1. Evaluate function at 5 regular points in interval
2. Estimate integral using polygons with 5 and 3 points
3. Compare difference to threshold × interval length
4. If accurate: add to accumulator
5. If not accurate: split interval and recurse on both halves

Parallel Region Setup
---------------------

.. code-block:: fortran

   accumulator = 0.0D0
   !$OMP parallel default(none) &
   !$OMP shared(accumulator) &
   !$OMP shared(startv, stopv, unit_err, gen_num)
      !$OMP single
      call rec_eval_shared_update(startv, stopv, unit_err, gen_num)
      !$OMP end single
   !$OMP end parallel

Task Startup in Recursive Function
----------------------------------

.. code-block:: fortran

   !$OMP task shared(accumulator) firstprivate(my_start, my_stop) &
   !$OMP default(none) firstprivate(my_gen, u_err) &
   !$OMP if(task_start)
   call rec_eval_shared_update(my_start, 0.5_dpr * (my_start + my_stop), u_err, my_gen)
   !$OMP end task

Result Accumulation Strategies
------------------------------

- **Atomic updates**: Poor performance with millions of updates
- **Threadprivate variables**: Each thread accumulates locally, then atomic update to shared variable
- **OpenMP 5.0**: Task reduction constructs (preferred)

Performance Results
-------------------

- Threadprivate accumulation provides satisfactory scaling
- Efficient utilization up to 128 cores
- GCC shows inferior scalability beyond 20 cores compared to Intel compilers

Advanced Task Features
======================

Task Dependencies (OpenMP 4.0)
-------------------------------

Declare dependencies between tasks:

.. code-block:: fortran

   !$omp task depend (type : list)

.. code-block:: c

   #pragma omp task depend (type : list)

Dependency Types:
~~~~~~~~~~~~~~~~~

- **in**: Depends on previous tasks with ``out`` or ``inout`` on list items
- **out**, **inout**: Depends on previous tasks with ``in``, ``out``, or ``inout`` on list items

Example:
~~~~~~~~

.. code-block:: c

   #pragma omp task depend (out: a)
      task_function_1(&a);
   #pragma omp task depend (in: a)
      task_function_2(a);
   #pragma omp task depend (in: a)
      task_function_3(a);

Taskloop (OpenMP 4.5)
---------------------

Distributes loops across tasks with implied taskgroup:

Fortran:
~~~~~~~~

.. code-block:: fortran

   !$OMP taskloop default(none) shared(…) private(…)
   do i = 1, N
      ...
   enddo

C/C++:
~~~~~~

.. code-block:: c

   #pragma omp taskloop default(none) shared(…) private(…)
   for (i=0; i<N; i++)
   {
      ...
   }

Taskloop Clauses
~~~~~~~~~~~~~~~~

- ``if(scalar-expr)``, ``shared``, ``private``, ``firstprivate``, ``lastprivate``, ``default``, ``collapse``, ``final(scalar-expr)``
- ``nogroup``: Removes implied taskgroup
- ``grainsize``: Controls iterations per task
- ``num_tasks``: Specifies number of tasks created
- No ``reduction`` clause available

Task Granularity Control
------------------------

- **grainsize**: Each task gets between grainsize and 2×grainsize iterations
- **num_tasks**: Directly specifies number of tasks
- Use only one of these clauses

Best Practices
==============

1. **Control task creation**: Use ``if`` clause to avoid creating tasks for small work amounts
2. **Adequate work per task**: Ensure sufficient computation to overcome task overhead
3. **Limit task explosion**: Use ``final`` clause or conditional creation to prevent excessive tasks
4. **Efficient synchronization**: Prefer ``taskgroup`` over multiple ``taskwait`` calls
5. **Smart result accumulation**: Use threadprivate variables or task reductions instead of atomic updates

Summary
=======

- Tasks enable parallelization of irregular algorithms
- Proper synchronization (``taskwait``, ``taskgroup``) is crucial
- Control task granularity to balance overhead and parallelism
- Advanced features (dependencies, taskloop) provide additional flexibility
- Performance requires careful management of task creation and result accumulation