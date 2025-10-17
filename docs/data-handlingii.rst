========================
More on Private Data
========================

:Authors: Pedro Ojeda & Joachim Hein
:Institutions: High Performance Computing Center North & Lund University

----

Outline
=======

This guide covers special versions of private data:

- ``firstprivate`` - initialization of private variables
- ``lastprivate`` - capturing values from last iteration
- ``reduction`` - parallel reductions (sums, products, etc.)
- ``threadprivate`` - privatizing global storage
- User-defined reductions (OpenMP 4.0+)

----

Private and Shared Data: Review
================================

In a parallel region, data can be either shared or private.

Shared Data
-----------

- Value unchanged on entry to parallel region
- Survives after end of parallel region

Private Data
------------

- Each thread has its own private copy
- Normally uninitialized at beginning of parallel region
- Contents typically lost when parallel region finishes
- **However:** Connection to values before/after is often needed

Memory Layout
-------------

.. code-block:: text

    Main Memory: [Shared data]
    
    Thread 0: [Private T0]
    Thread 1: [Private T1]
    Thread 2: [Private T2]
    Thread 3: [Private T3]

----

Clause: firstprivate
====================

Problem
-------

Private variables are not initialized by default.

Solution: firstprivate Clause
------------------------------

The ``firstprivate`` clause:

- Declares variable(s) as private
- Initializes each private copy with the value prior to the construct

Fortran Example
---------------

.. code-block:: fortran

    integer :: lsum = 10
    
    !$omp parallel &
    !$omp firstprivate(lsum)
        lsum = lsum + omp_get_thread_num()
        print *, lsum
    !$omp end parallel

C Example
---------

.. code-block:: c

    int lsum = 10;
    
    #pragma omp parallel \
        firstprivate(lsum)
    {
        lsum += omp_get_thread_num();
        printf("%i\n", lsum);
    }

Expected Output
---------------

With 4 threads:

.. code-block:: text

    Thread 0: 10
    Thread 1: 11
    Thread 2: 12
    Thread 3: 13

----

Example: Vector Norm with private
==================================

Fortran Version
---------------

.. code-block:: fortran

    norm = 0.0
    
    !$omp parallel default(none) &
    !$omp shared(vect, norm) private(i, lNorm)
        lNorm = 0.0
        
        !$omp do
        do i = 0, vleng
            lNorm = lNorm + vect(i)**2
        enddo
        
        !$omp atomic update
        norm = norm + lNorm
    !$omp end parallel
    
    norm = sqrt(norm)

C Version
---------

.. code-block:: c

    norm = 0.0;
    
    #pragma omp parallel default(none) \
        shared(vect, norm) private(i, lNorm)
    {
        lNorm = 0.0;
        
        #pragma omp for
        for (i = 0; i < vleng; i++)
            lNorm += vect[i] * vect[i];
        
        #pragma omp atomic update
        norm += lNorm;
    }
    
    norm = sqrt(norm);

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

.. note::
   ``lNorm`` must be explicitly initialized to 0.0 inside the parallel region.

----

Example: Vector Norm with firstprivate
=======================================

Fortran Version
---------------

.. code-block:: fortran

    norm = 0.0
    lNorm = 0.0
    
    !$omp parallel default(none) &
    !$omp shared(vect, norm) private(i) firstprivate(lNorm)
        !$omp do
        do i = 0, vleng
            lNorm = lNorm + vect(i)**2
        enddo
        
        !$omp atomic update
        norm = norm + lNorm
    !$omp end parallel
    
    norm = sqrt(norm)

C Version
---------

.. code-block:: c

    norm = 0.0;
    lNorm = 0.0;
    
    #pragma omp parallel default(none) \
        shared(vect, norm) private(i) firstprivate(lNorm)
    {
        #pragma omp for
        for (i = 0; i < vleng; i++)
            lNorm += vect[i] * vect[i];
        
        #pragma omp atomic update
        norm += lNorm;
    }
    
    norm = sqrt(norm);

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

.. important::
   With ``firstprivate``, ``lNorm`` is automatically initialized to 0.0 from the master thread's value.

----

Clause: lastprivate
===================

Purpose
-------

The ``lastprivate`` clause:

- Used with loop and sections constructs
- Variable is private during execution
- At the end: assigns value from last iteration or section
- Undefined if not set in last iteration/section

Combined Usage
--------------

Variables can be both ``firstprivate`` **and** ``lastprivate``.

Fortran Example
---------------

.. code-block:: fortran

    integer :: i, a
    
    !$omp parallel do &
    !$omp lastprivate(a)
    do i = 1, 100
        a = i + 1
        call func(a)
    enddo
    
    print *, "a=", a
    ! This prints: a=101

C Example
---------

.. code-block:: c

    int i, a;
    
    #pragma omp parallel for \
        lastprivate(a)
    for (i = 0; i < 100; i++)
    {
        a = i + 1;
        func(a);
    }
    
    printf("a=%i\n", a);
    // This prints: a=100

.. note::
   The value from the sequentially last iteration is assigned back to the original variable.

----

Reduction Variables
===================

Common Need
-----------

Reductions of private variables are frequently needed:

- Averages of array values
- Scalar products
- Sum, product, minimum, maximum operations

Previous Approach
-----------------

We've done this before (e.g., vector norm example) using ``atomic`` to protect the update.

Better Approach: Reduction Clause
----------------------------------

For a reduction, we specify:

- **Operation:** e.g., addition, multiplication, OR, AND, etc.
- **One or more variables**
- A construct can have more than one reduction

----

Behavior of Reduction
=====================

Basic Syntax
------------

.. code-block:: fortran

    reduction(operator : variable_list)

How It Works
------------

**Variables specified in reduction:**

1. Each thread gets a private copy
2. Private copies are initialized with default values matching the operator
3. At the end of the construct (e.g., parallel region):
   
   - Value prior to construct is combined with private copies
   - Using the specified operator for combining values
   - New combined value is available after the construct

----

Example: Memory Movements for Reduction (C)
============================================

.. code-block:: c

    int b;
    b = 5;
    
    #pragma omp parallel \
        reduction(+:b)
    {
        b += omp_get_thread_num();
    }
    
    printf("%i\n", b);

Memory Behavior
---------------

.. code-block:: text

    Main Memory: b = 5
    
    Thread 0: b = 0 → b = 0
    Thread 1: b = 0 → b = 1
    Thread 2: b = 0 → b = 2
    Thread 3: b = 0 → b = 3
    
    Final: 5 + 0 + 1 + 2 + 3 = 11

Output: ``11``

.. note::
   Each thread's private copy is initialized to 0 (identity for addition), then combined at the end.

----

Example: Memory Movements for Reduction (Fortran)
==================================================

.. code-block:: fortran

    integer :: b
    
    b = 5
    
    !$omp parallel &
    !$omp reduction(+:b)
        b = b + omp_get_thread_num()
    !$omp end parallel
    
    print *, b

Memory Behavior
---------------

.. code-block:: text

    Main Memory: b = 5
    
    Thread 0: b = 0 → b = 0
    Thread 1: b = 0 → b = 1
    Thread 2: b = 0 → b = 2
    Thread 3: b = 0 → b = 3
    
    Final: 5 + 0 + 1 + 2 + 3 = 11

Output: ``11``

.. note::
   Each thread's private copy is initialized to 0 (identity for addition), then combined at the end.

----

Example: Vector Norm with atomic update
========================================

Fortran Version
---------------

.. code-block:: fortran

    norm = 0.0
    lNorm = 0.0
    
    !$omp parallel default(none) &
    !$omp shared(vect, norm) private(i) firstprivate(lNorm)
        !$omp do
        do i = 1, vleng
            lNorm = lNorm + vect(i)**2  ! private copy
        enddo
        
        !$omp atomic update
        norm = norm + lNorm
    !$omp end parallel  ! combine copies
    
    norm = sqrt(norm)  ! master copy

C Version
---------

.. code-block:: c

    norm = 0.0;
    lNorm = 0.0;
    
    #pragma omp parallel default(none) \
        shared(vect, norm) private(i) firstprivate(lNorm)
    {
        #pragma omp for
        for (i = 0; i < vleng; i++)
            lNorm += vect[i] * vect[i];
        
        #pragma omp atomic update
        norm += lNorm;
    }
    
    norm = sqrt(norm);

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

----

Example: Vector Norm with reduction
====================================

Fortran Version
---------------

.. code-block:: fortran

    norm = 0.0  ! master copy
    ! lNorm gone
    
    !$omp parallel default(none) &
    !$omp shared(vect) reduction(+:norm) private(i)
        !$omp do  ! private copy = 0
        do i = 1, vleng
            norm = norm + vect(i)**2  ! private copy
        enddo
    !$omp end parallel  ! combine copies
    
    norm = sqrt(norm)  ! master copy

C Version
---------

.. code-block:: c

    norm = 0.0;  // master copy
    // lNorm gone!
    
    #pragma omp parallel default(none) \
        shared(vect) reduction(+:norm) private(i)
    {  // private copy: 0
        #pragma omp for
        for (i = 0; i < vleng; i++)
            norm += vect[i] * vect[i];  // private copy
    }  // combine copies
    
    norm = sqrt(norm);  // master copy

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

.. important::
   No need for ``lNorm`` variable or ``atomic`` directive. The reduction clause handles everything automatically.

----

Example: Vector Norm with reduction (Simplified)
=================================================

Fortran Version
---------------

.. code-block:: fortran

    norm = 0.0  ! master copy
    
    !$omp parallel do default(none) &
    !$omp shared(vect) reduction(+:norm)
    do i = 1, vleng
        norm = norm + vect(i)**2  ! private copy
    enddo
    !$omp end parallel do
    
    norm = sqrt(norm)  ! master copy

C Version
---------

.. code-block:: c

    norm = 0.0;  // master copy
    
    #pragma omp parallel for default(none) \
        shared(vect) reduction(+:norm)
    for (i = 0; i < vleng; i++)
        norm += vect[i] * vect[i];  // private copy
    
    norm = sqrt(norm);  // master copy

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

.. note::
   Using ``parallel do``/``parallel for`` makes the code even more concise.

----

Supported Operators: Fortran (OpenMP 3.0)
==========================================

.. list-table::
   :header-rows: 1
   :widths: 20 15 35

   * - Name
     - Symbol
     - Initial Value of Local Copy
   * - add
     - ``+``
     - 0
   * - multiply
     - ``*``
     - 1
   * - subtract
     - ``-``
     - 0
   * - logical AND
     - ``.and.``
     - ``.true.``
   * - logical OR
     - ``.or.``
     - ``.false.``
   * - EQUIVALENCE
     - ``.eqv.``
     - ``.true.``
   * - NON-EQUIV.
     - ``.neqv.``
     - ``.false.``
   * - maximum
     - ``max``
     - smallest representable number
   * - minimum
     - ``min``
     - largest representable number
   * - bitwise AND
     - ``iand``
     - all bits on
   * - bitwise OR
     - ``ior``
     - 0
   * - bitwise XOR
     - ``ieor``
     - 0

----

Supported Operators: C (OpenMP 3.0)
====================================

.. list-table::
   :header-rows: 1
   :widths: 20 15 35

   * - Name
     - Symbol
     - Initial Value of Local Copy
   * - add
     - ``+``
     - 0
   * - multiply
     - ``*``
     - 1
   * - subtract
     - ``-``
     - 0
   * - bitwise AND
     - ``&``
     - ``~0``
   * - bitwise OR
     - ``|``
     - 0
   * - bitwise XOR
     - ``^``
     - 0
   * - logical AND
     - ``&&``
     - 1
   * - logical OR
     - ``||``
     - 0

----

Restrictions on Reduction
==========================

Important Limitations
---------------------

**C/C++:**

- Arrays are **unsupported** as reduction variables
- No pointer or reference types

**Fortran:**

- ``ALLOCATABLE`` arrays must be allocated at the beginning of construct
- Must not be deallocated during construct
- No Fortran pointers or assumed-size arrays

Order of Execution
------------------

.. warning::
   No order of threads is specified!
   
   - Repeated runs are typically **not bit-identical**
   - This is common in parallel computing
   - This is technically a race condition, which is typically tolerated

OpenMP 4.0 Enhancement
----------------------

OpenMP 4.0 allows you to declare your own custom reductions.

----

User-Defined Reductions
=======================

Purpose
-------

Allows definition of custom reduction operations.

Use Cases
---------

Particularly useful with derived data types:

- **C/C++:** ``struct``
- **Fortran:** ``type``

Requirements
------------

You need to provide:

1. **Combiner:** Combines thread-private results to final result
2. **Initializer:** Initializes private contributions at outset

----

Case Study: Maximum Value and Its Position
===========================================

Problem Statement
-----------------

Given a large array:

- Determine the maximum value
- Find the location (index) of the maximum in the array

Parallelization Strategy
------------------------

1. Assign a portion of array to each thread
2. Each thread determines maximum and position in its part
3. Use user-defined reduction to determine final result

----

User-Defined Reduction in Fortran
==================================

Step 1: Define the Data Type
-----------------------------

.. code-block:: fortran

    type :: mx_s
        real :: value
        integer :: index
    end type

Step 2: Declare the Reduction
------------------------------

.. code-block:: fortran

    !$omp declare reduction(maxloc: mx_s: &
    !$omp mx_combine(omp_out, omp_in)) &
    !$omp initializer(mx_init(omp_priv, omp_orig))

Properties
----------

- The operation can be triggered by the name ``maxloc``
- Utilizes subroutine ``mx_combine`` and ``mx_init``
- Acts on objects of type ``mx_s``

----

The Initializer in Fortran
===========================

Purpose
-------

Can be a subroutine or assignment statement (here: subroutine).

Special Variables
-----------------

- ``omp_priv``: reference to variable to be initialized
- ``omp_orig``: reference to original variable prior to construct

Example Implementation
----------------------

Initialize from value prior to construct:

.. code-block:: fortran

    subroutine mx_init(priv, orig)
        type(mx_s), intent(out) :: priv
        type(mx_s), intent(in) :: orig
        
        priv%value = orig%value
        priv%index = orig%index
    end subroutine mx_init

----

The Combiner in Fortran
========================

Purpose
-------

Can be a subroutine or assignment statement (here: subroutine).

Special Variables
-----------------

- ``omp_in``: reference to contribution from thread
- ``omp_out``: reference to combined result

Example Implementation
----------------------

Replace if contribution is larger:

.. code-block:: fortran

    subroutine mx_combine(out, in)
        type(mx_s), intent(inout) :: out
        type(mx_s), intent(in) :: in
        
        if (out%value < in%value) then
            out%value = in%value
            out%index = in%index
        endif
    end subroutine mx_combine

----

Using User-Defined Reduction in Fortran
========================================

.. code-block:: fortran

    mx%value = val(1)
    mx%index = 1
    
    !$omp parallel do reduction(maxloc: mx)
    do i = 2, count
        if (mx%value < val(i)) then
            mx%value = val(i)
            mx%index = i
        endif
    enddo

Benefits
--------

- Easily readable code
- Similar to what one would do in serial programming
- Abstracts away the parallel complexity

----

User-Defined Reduction in C
============================

Step 1: Define the Data Type
-----------------------------

.. code-block:: c

    struct mx_s {
        float value;
        int index;
    };

Step 2: Declare the Reduction
------------------------------

.. code-block:: c

    #pragma omp declare reduction(maxloc: \
        struct mx_s: mx_combine(&omp_out, &omp_in)) \
        initializer(mx_init(&omp_priv, &omp_orig))

Properties
----------

- The operation can be triggered by the name ``maxloc``
- Utilizes functions ``mx_combine`` and ``mx_init``
- Acts on objects of type ``struct mx_s``

----

The Initializer in C
=====================

Purpose
-------

An expression (here: implemented with a function).

Special Variables
-----------------

- ``omp_priv``: reference to variable to be initialized
- ``omp_orig``: reference to original variable prior to construct

Example Implementation
----------------------

Initialize from value prior to construct:

.. code-block:: c

    void mx_init(struct mx_s *priv, struct mx_s *orig)
    {
        priv->value = orig->value;
        priv->index = orig->index;
    }

----

The Combiner in C
=================

Purpose
-------

An expression (here: implemented with a function).

Special Variables
-----------------

- ``omp_in``: reference to contribution from thread
- ``omp_out``: reference to combined result

Example Implementation
----------------------

Replace if contribution is larger:

.. code-block:: c

    void mx_combine(struct mx_s *out, struct mx_s *in)
    {
        if (out->value < in->value) {
            out->value = in->value;
            out->index = in->index;
        }
    }

----

Using User-Defined Reduction in C
==================================

.. code-block:: c

    mx->value = val[0];
    mx->index = 0;
    
    #pragma omp parallel for reduction(maxloc: mx)
    for (i = 1; i < count; i++) {
        if (mx.value < val[i])
        {
            mx.value = val[i];
            mx.index = i;
        }
    }

Benefits
--------

- Easily readable code
- Similar to what one would do in serial programming
- Abstracts away the parallel complexity

----

Declaring a Reduction Operation: Syntax Summary
================================================

C Syntax
--------

.. code-block:: c

    #pragma omp declare reduction (reduction-identifier : \
        typename-list : combiner) [initializer-clause] new-line

Fortran Syntax
--------------

.. code-block:: fortran

    !$omp declare reduction(reduction-identifier : &
    !$omp type-list : combiner) [initializer-clause]

Components
----------

- **reduction-identifier:** Name for your reduction
- **typename-list/type-list:** Data types the reduction applies to
- **combiner:** Function/subroutine to combine values
- **initializer-clause:** Optional initialization specification

----

Dealing with Global Storage
============================

Default Behavior
----------------

By default, global storage is shared among all threads.

Examples of Global Storage
---------------------------

**C/C++:**

- File scope variables
- ``static`` variables

**Fortran:**

- ``COMMON`` blocks
- Module data
- Variables with ``save`` attribute

Problem
-------

This default behavior is not always what is needed.

----

Directive: threadprivate in C
==============================

Purpose
-------

The ``threadprivate`` directive makes global storage private to each thread.

Syntax
------

.. code-block:: c

    int g_var = 1;
    #pragma omp threadprivate(g_var)
    
    int main()
    {
        g_var = 4;
        
        #pragma omp parallel
        {
            printf("%d\n", g_var);
        }
        
        return 0;
    }

Behavior
--------

- Each thread gets a private copy
- Outside parallel region: modifications affect master's copy

Example Output
--------------

With 4 threads:

.. code-block:: text

    Thread 0 (master): 4
    Thread 1: 1
    Thread 2: 1
    Thread 3: 1

----

Directive: threadprivate in Fortran
====================================

Purpose
-------

The ``threadprivate`` directive makes global storage private to each thread.

Syntax
------

.. code-block:: fortran

    module gmod
        integer :: g_var = 1
        !$omp threadprivate(g_var)
    end module gmod
    
    program example
        use gmod
        
        g_var = 4
        
        !$omp parallel
        print *, g_var
        !$omp end parallel
    end program example

Behavior
--------

- Each thread gets a private copy
- Outside parallel region: modifications affect master's copy

Example Output
--------------

With 4 threads:

.. code-block:: text

    Thread 0 (master): 4
    Thread 1: 1
    Thread 2: 1
    Thread 3: 1

----

Clause: copyin
==============

Purpose
-------

The ``copyin`` clause initializes threadprivate data from the master thread.

C Example
---------

.. code-block:: c

    int g_var = 1;
    #pragma omp threadprivate(g_var)
    
    int main()
    {
        g_var = 4;
        
        #pragma omp parallel \
            copyin(g_var)
        {
            printf("%d\n", g_var);
        }
        
        return 0;
    }

Fortran Example
---------------

.. code-block:: fortran

    module gmod
        integer :: g_var = 1
        !$omp threadprivate(g_var)
    end module gmod
    
    program example
        use gmod
        
        g_var = 4
        
        !$omp parallel copyin(g_var)
        print *, g_var
        !$omp end parallel
    end program example

Output
------

With 4 threads, **all** threads print: ``4``

----

More on threadprivate
=====================

Data Persistence
----------------

``threadprivate`` data remains unchanged between parallel regions if:

1. Neither region is nested inside another parallel region
2. Both regions have the same thread count
3. Internal variable ``dyn-var`` is false in both regions
   
   - Use function ``omp_set_dynamic`` to control this

Fortran COMMON Blocks
----------------------

In Fortran, you can make a ``COMMON`` block threadprivate:

.. code-block:: fortran

    integer :: a, b, c
    COMMON /abccom/ a, b, c
    !$OMP threadprivate(/abccom/)

----

Summary
=======

This guide covered special private variables in OpenMP:

Special Private Variable Types
-------------------------------

- **firstprivate:** Initialization of private variables from master thread
- **lastprivate:** Set value of private variable to value of last loop iteration or last section at end of construct
- **reduction:** Calculating sums, products, etc. in parallel
- **threadprivate:** Privatize global storage

User-Defined Reductions
------------------------

- Available in OpenMP 4.0+
- Useful for complex data types
- Requires combiner and initializer functions

When to Use Standard Constructs
--------------------------------

The above constructs handle standard situations. For special cases, use:

- Explicit initialization of private variables from shared variables
- ``atomic``/``critical`` for writes to shared variables