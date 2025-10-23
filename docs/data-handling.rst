Data for Parallel Regions
=========================

.. objectives::
    
    - **Data:**

      - What is private data
      - What is shared data
      - How to control which is which

    - **Race conditions:**

      - Basic constructs to avoid data races



Shared & Private Data
^^^^^^^^^^^^^^^^^^^^^

**Private and Shared Data Concepts**


In a parallel region, data can be either shared or private.

*Shared Data*


- Every thread can access the same memory location (potential for conflict)
- Value remains unchanged on entry to parallel region
- Value survives after end of parallel region

*Private Data*


- Each thread has its own private copy
- Normally uninitialized at the beginning of parallel region
- Contents typically lost when parallel region finishes

.. note::
   In a shared memory architecture, shared data resides in the main memory accessible by all processors, while 
   each thread maintains its own private data copy.



*Controlling Data Sharing in Fortran*

For data declared before the start of a parallel region:

- Use clause ``shared`` to declare a data structure as shared
- Use clause ``private`` to declare a data structure as private


.. code-block:: fortran

    integer :: a=5, b
    
    !$omp parallel &
    !$omp shared(a) private(b)
        b = a + omp_get_thread_num()
        print *, "result=", b
    !$omp end parallel

In this example:

- ``a`` is shared among all threads
- ``b`` is private to each thread



*Controlling Data Sharing in C*


For data declared before the start of a parallel region:

- Use clause ``shared`` to declare a data structure as shared
- Use clause ``private`` to declare a data structure as private


.. code-block:: c

    int a, b;
    a = 5;
    
    #pragma omp parallel \
        shared(a) private(b)
    {
        b = a + omp_get_thread_num();
        printf("%i\n", b);
    }

In this example:

- ``a`` is shared among all threads
- ``b`` is private to each thread



Private Data
^^^^^^^^^^^^

Private data is typically used for control variables, including:

- Thread identification
- Loop indices
- Variables internal to the algorithm

*Default Private Variables*

Most variables declared inside a parallel region are private by default:

- Variables declared inside the block (C/C++)
- Variables in subroutine/function called from inside parallel region

*Exceptions*


The following are **NOT** private by default:

- ``static`` (C/C++) or ``save`` (Fortran) variables
- File scope variables (C/C++) or ``COMMON`` blocks
- Variables passed by reference inherit their data-sharing attribute

.. warning::
   In Fortran, special care is needed with ``COMMON`` and ``EQUIVALENCE`` statements.



*Example: Memory Movements for Private Data (Fortran)*

.. code-block:: fortran

    integer :: b
    
    b = 5
    
    !$OMP parallel &
    !$OMP private(b)
        b = omp_get_thread_num()
        b = b + 3
    !$OMP end parallel
    
    b = 7

*Memory Layout*

.. code-block:: text

    Main Memory: b = 5
    
    Thread 0: b = 0 → b = 3
    Thread 1: b = 1 → b = 4
    Thread 2: b = 2 → b = 5
    Thread 3: b = 3 → b = 6
    
    Main Memory: b = 7

.. note::
   Each thread has its own copy of ``b``, and changes do not affect the original value in main memory.



*Example: Memory Movements for Private Data (C)*

.. code-block:: c

    int b;
    
    b = 5;
    
    #pragma omp parallel \
        private(b)
    {
        b = omp_get_thread_num();
        b += 3;
    }
    
    b = 7;

*Memory Layout*

.. code-block:: text

    Main Memory: b = 5
    
    Thread 0: b = 0 → b = 3
    Thread 1: b = 1 → b = 4
    Thread 2: b = 2 → b = 5
    Thread 3: b = 3 → b = 6
    
    Main Memory: b = 7

.. note::
   Each thread has its own copy of ``b``, and changes do not affect the original value in main memory.



**Shared Data**


- Majority of the data in parallel programs
- Typically large data structures (e.g., arrays)

*Properties*

- Keeps its value on entry to parallel region
- Keeps its value on exit from parallel region
- Every thread can access (read and/or write) the data

*Safety Considerations*

**Safe scenario:**

- Multiple threads only read the data

**Dangerous scenario:**

- Multiple threads access the same memory location
- At least one of these is a write access
- This easily results in a **race condition**


*Example: Vector Initialization (Fortran)*


.. code-block:: fortran

    integer, parameter :: vleng = 120
    integer :: vect(vleng), myNum, start, fin, i
    
    !$omp parallel shared(vect) &
    !$omp private(myNum, start, fin, i)
        myNum = vleng / omp_get_num_threads()
        start = 1 + omp_get_thread_num() * myNum
        fin = (omp_get_thread_num() + 1) * myNum
        
        do i = start, fin
            vect(i) = 4 * i  ! threads write different elements
        enddo
    !$omp end parallel

Mathematical notation: :math:`v_i = 4i`

.. note::
   This is safe because each thread writes to different elements of the shared array.



*Example: Vector Initialization (C)*

.. code-block:: c

    const int vleng = 120;
    int vect[vleng], myNum, start, fin, i;
    
    #pragma omp parallel shared(vect) \
        private(myNum, start, fin, i)
    {
        myNum = vleng / omp_get_num_threads();
        start = omp_get_thread_num() * myNum;
        fin = start + myNum;
        
        for (i = start; i < fin; i++)
            vect[i] = 4 * i;  // threads write different elements
    }

Mathematical notation: :math:`v_i = 4i`

.. note::
   This is safe because each thread writes to different elements of the shared array.



*Example: Write Conflict for Shared Data (Fortran)*

.. code-block:: fortran

    integer :: a, b
    
    a = 5
    
    !$OMP parallel &
    !$OMP shared(a, b)
        b = a + omp_get_thread_num()
        print *, "updated"
        print *, "my b:", b
    !$OMP end parallel

*Memory Behavior*

.. code-block:: text

    Main Memory: a = 5
    
    All threads read: a = 5
    
    Thread 0: b = 5
    Thread 1: b = 6
    Thread 2: b = 7
    Thread 3: b = 8
    
    Final b value: RANDOM (could be 5, 6, 7, or 8)

.. warning::
   - Final ``b`` value is random/unpredictable
   - Individual threads might print ``b`` before it has its final value
   - This is a **race condition**


*Example: Write Conflict for Shared Data (C)*

.. code-block:: c

    int a, b;
    
    a = 5;
    
    #pragma omp parallel \
        shared(a, b)
    {
        b = a + omp_get_thread_num();
        printf("updated\n");
        printf("my b: %i\n", b);
    }

*Memory Behavior*

.. code-block:: text

    Main Memory: a = 5
    
    All threads read: a = 5
    
    Thread 0: b = 5
    Thread 1: b = 6
    Thread 2: b = 7
    Thread 3: b = 8
    
    Final b value: RANDOM (could be 5, 6, 7, or 8)

.. warning::
   - Final ``b`` value is random/unpredictable
   - Individual threads might print ``b`` before it has its final value
   - This is a **race condition**



Default Clause
^^^^^^^^^^^^^^

The ``default`` clause can be used on a parallel or task construct to determine data sharing of implicitly determined variables.


**In C:**

.. code-block:: c

    default(shared | none)

**In Fortran:**

.. code-block:: fortran

    default(shared | none | private | firstprivate)



For parallel constructs, if no ``default`` clause is supplied, ``default(shared)`` applies.

*Recommendation*

.. important::
   Using ``default(none)`` is typically a good idea!
   
   With ``default(none)``, all variables accessed in the parallel region must be explicitly declared as ``shared``, ``private``, etc.



Fixing Data Races
^^^^^^^^^^^^^^^^^


OpenMP provides several constructs to avoid data races:

- ``barrier`` - synchronization point
- ``critical`` - mutual exclusion region
- ``atomic`` - lightweight protection for simple operations

.. note::
   These constructs impact code performance, but we have no interest in "fast garbage"!



Barrier and Synchronization
^^^^^^^^^^^^^^^^^^^^^^^^^^^

**The Barrier Construct**

**Fortran:**

.. code-block:: fortran

    !$omp barrier

**C:**

.. code-block:: c

    #pragma omp barrier

*Behavior*

- All threads wait for the last one to arrive at the barrier
- Registers are flushed to the memory system
- All threads must have the barrier in their line of execution

.. warning::
   If not all threads reach the barrier, a **deadlock** will occur!

*Visual Representation*

.. code-block:: text

    Thread 0: A ──────┐
    Thread 1: A ──────┤
    Thread 2: A ──────┼── BARRIER ──┬──   B
    Thread 3: A ──────┘              ├──  B
                                      ├── B
                                      └── B
                Time ─────────────────────────>



*Example: Data Race in Matrix Transpose (Fortran)*

.. code-block:: fortran

    !$omp parallel default(none) &
    !$omp private(mysize, tid, i, j) shared(matrix, mtrans)
        tid = omp_get_thread_num()
        mysize = nsize / omp_get_num_threads()
        
        do j = 1 + tid*mysize, (tid+1)*mysize
            do i = 1, nsize
                matrix(i,j) = 1000.0 * j + i
            enddo
        enddo
        
        !$omp barrier
        
        do j = 1 + tid*mysize, (tid+1)*mysize
            do i = 1, nsize
                mtrans(i,j) = matrix(j,i)
            enddo
        enddo
    !$omp end parallel

.. note::
   The barrier ensures that all threads complete writing to ``matrix`` before any thread begins reading from it for the transpose operation.



*Example: Data Race in Matrix Transpose (C)*

.. code-block:: c

    #pragma omp parallel default(none) \
        private(mysize, tid, i, j) shared(matrix, mtrans)
    {
        tid = omp_get_thread_num();
        mysize = nsize / omp_get_num_threads();
        
        for (i = tid*mysize; i < (tid+1)*mysize; i++)
            for (j = 0; j < nsize; j++)
                matrix[i][j] = 1000.0 * j + i;
        
        #pragma omp barrier
        
        for (i = tid*mysize; i < (tid+1)*mysize; i++)
            for (j = 0; j < nsize; j++)
                mtrans[i][j] = matrix[j][i];
    }

.. note::
   The barrier ensures that all threads complete writing to ``matrix`` before any thread begins reading from it for the transpose operation.



**Critical Regions**



Critical regions protect updates of shared memory locations by ensuring only one thread executes the critical region at a time.

*Syntax in C*

.. code-block:: c

    #pragma omp critical (name)
    {
        code-block
    }

*Syntax in Fortran*


.. code-block:: fortran

    !$omp critical (name)
        code-block
    !$omp end critical (name)



- **Name is optional:**
  
  - If named: only one thread in all regions with the same name
  - If unnamed: only one thread in all unnamed regions

- Implies a register flush at entrance and exit
- Useful to execute non-thread-safe functions
- Performance penalty due to serialization



*Example: Use of Critical Region (Fortran)*

Computing a sum with critical section:

.. code-block:: fortran

    sum = 0.0_dpr
    
    !$omp parallel default(none) &
    !$omp shared(sum) private(tid, cont)
        tid = omp_get_thread_num()
        cont = func(tid)
        
        !$omp critical (exp_up)
            sum = sum + cont
            print *, tid, ": c=", cont, " s=", sum
        !$omp end critical (exp_up)
    !$omp end parallel

Mathematical notation: :math:`\sum_{k=0}^{n-1} e^k`

.. note::
   The critical region ensures that only one thread updates ``sum`` at a time, preventing race conditions.



*Example: Use of Critical Region (C)*

Computing a sum with critical section:

.. code-block:: c

    sum = 0.0;
    
    #pragma omp parallel default(none) \
        shared(sum) private(tid, cont)
    {
        tid = omp_get_thread_num();
        cont = func(tid);
        
        #pragma omp critical (exp_up)
        {
            sum += cont;
            printf("%i: c=%f s=%f\n", tid, cont, sum);
        }
    }

Mathematical notation: :math:`\sum_{k=0}^{n-1} e^k`

.. note::
   The critical region ensures that only one thread updates ``sum`` at a time, preventing race conditions.

----

**Atomic Operations**


``atomic`` is a lightweight alternative to ``critical`` for simple cases.



- Works with simple statements only
- Can use special hardware instructions if they exist
- Flushes the "protected" variable on entry and exit
- Much more efficient than ``critical`` for simple operations

*Versions (from OpenMP 3.1)*

Four different versions:

- ``read`` - atomic read operation
- ``write`` - atomic write operation
- ``update`` - atomic update operation
- ``capture`` - atomic update with capture of old/new value

*OpenMP 4.0 Enhancement*

Adding ``seq_cst`` to atomic flushes all variables:

- Important for controlling instruction reordering
- Example use case: implementing a lock



*Atomic Read*


Protects only the reading of a scalar intrinsic variable.

*Fortran Syntax*

.. code-block:: fortran

    !$omp atomic read
    v = x

*C Syntax*

.. code-block:: c

    #pragma omp atomic read
    v = x;



- Protects only the reading of scalar variable ``x``
- Flushes ``x`` on entry and exit



*Atomic Write*

Protects only the writing of a scalar intrinsic variable.

*Fortran Syntax*

.. code-block:: fortran

    !$omp atomic write
    x = expr

*C Syntax*

.. code-block:: c

    #pragma omp atomic write
    x = expr;

*Example Expressions*

.. code-block:: c

    x = 5;
    x = v;
    x = func(a);

.. warning::
   - Protects only the writing of ``x``
   - No protection for evaluation of ``expr`` on the right-hand side
   - Flushes ``x`` on entry and exit



*Atomic Update*


Protects the update of a variable in simple arithmetic operations.

.. note::
   ``atomic update`` was the only atomic operation prior to OpenMP 3.1. The ``update`` keyword is optional for backward compatibility.



- Only protects the update of the variable, not function calls on the right-hand side
- Works with simple statements only
- Can use special hardware instructions if available
- Flushes the updated variable on entry and exit

*Example*

.. code-block:: c

    x += func(a);
    x = x + func(a);

.. warning::
   The evaluation of ``func(a)`` is NOT protected. Use ``critical`` if protection is needed!



*Atomic Update: Fortran Statements*

*Examples*

.. code-block:: fortran

    !$omp atomic update
    x = x + 1
    
    !$omp atomic update
    x = x + f(a)

.. warning::
   The evaluation of ``f(a)`` is NOT protected. Use ``critical`` if needed!

*Allowed Operations*

.. code-block:: fortran

    x = x operator expr
    x = expr operator x
    x = intr_proc(x, expr_list)
    x = intr_proc(expr_list, x)

Where:

- ``x`` is scalar, intrinsic type
- ``operator`` is one of: ``+``, ``*``, ``-``, ``/``, ``.AND.``, ``.OR.``, ``.EQV.``, ``.NEQV.``
- ``intr_proc`` is one of: ``MAX``, ``MIN``, ``IAND``, ``IOR``, ``IEOR``

.. note::
   The ``update`` keyword is optional for consistency with older OpenMP standards.



*Example: Vector Norm (Fortran)*

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
   Each thread computes a local sum (``lNorm``), then atomically adds it to the global ``norm``.



*Atomic Update: C Statements*

*Examples*

.. code-block:: c

    #pragma omp atomic update
    x++;
    
    #pragma omp atomic update
    x += f(a);

.. warning::
   The evaluation of ``f(a)`` is NOT protected. Use ``critical`` if needed!

*Allowed Operations*

.. code-block:: c

    x binop= expr;
    x++;
    ++x;
    x--;
    --x;
    x = x binop expr;

Where:

- ``x`` is lvalue, scalar
- ``binop`` is one of: ``+``, ``*``, ``-``, ``/``, ``&``, ``^``, ``|``, ``<<``, ``>>``

.. note::
   The ``update`` keyword is optional for consistency with older OpenMP standards.



*Example: Vector Norm (C)*

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
    }  // synchronize at end parallel
    
    norm = sqrt(norm);

Mathematical notation: :math:`\sqrt{\sum_i v(i) \cdot v(i)}`

.. note::
   Each thread computes a local sum (``lNorm``), then atomically adds it to the global ``norm``.



*Atomic Capture*

Atomic capture allows you to:

- Update a shared variable atomically
- Keep a thread-private copy of **either** (but not both):
  
  - The old value before update
  - The new value after update

Restrictions apply to the allowed statement forms.



*Atomic Capture: C Statements*


.. code-block:: c

    #pragma omp atomic capture
    statement_or_structured_block

*Allowed Statements (OpenMP 4.0)*

.. code-block:: c

    v = x++;
    v = x--;
    v = ++x;
    v = --x;
    v = x binop= expr;
    v = x = x binop expr;
    v = x = expr binop x;

*Allowed Structured Blocks*

.. code-block:: c

    {v = x; x binop= expr;}
    {x binop= expr; v = x;}
    {v = x; x = x binop expr;}
    {v = x; x = expr binop x;}
    {x = x binop expr; v = x;}
    {x = expr binop x; v = x;}
    {v = x; x = expr;}
    {v = x; x++;}
    {v = x; ++x;}
    {++x; v = x;}
    {x++; v = x;}
    {v = x; x--;}
    {v = x; --x;}
    {--x; v = x;}
    {x--; v = x;}



*Atomic Capture: Fortran Statements*

*Syntax Form 1*

.. code-block:: fortran

    !$omp atomic capture
        update-statement
        capture-statement
    !$omp end atomic

*Syntax Form 2*

.. code-block:: fortran

    !$omp atomic capture
        capture-statement
        update-statement
    !$omp end atomic

*Allowed Update Statements*

.. code-block:: fortran

    x = x operator expr
    x = expr operator x
    x = intr_proc(x, expr_list)
    x = intr_proc(expr_list, x)

*Allowed Capture Statements*

.. code-block:: fortran

    v = x



Summary
^^^^^^^

This guide covered the following OpenMP concepts:

**Data Management:**

- Private data: each thread has its own copy
- Shared data: accessible by all threads
- Controlling data attributes with clauses

**Preventing Race Conditions:**

- ``barrier``: synchronization point for all threads
- ``critical``: mutual exclusion for code regions
- ``atomic``: lightweight protection for simple operations
  
  - ``read``, ``write``, ``update``, ``capture``

**Parallelization Strategies:**

- Examples demonstrated various approaches to parallel data management
- Techniques for avoiding data races while maintaining performance