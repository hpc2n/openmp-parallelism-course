OpenMP parallelism in scientific computing
==============================================

:Abstract: The purpose of the course is to learn when a code could benefit from task-based parallelism, and how to apply it. A task-based algorithm comprises of a set of self-contained tasks that have well-defined inputs and outputs. This differs from the common practice of organizing an implementation into subroutines in that a task-based implementation does not call the associated computation kernels directly, instead it is the role of a runtime system to schedule the task to various computational resources, such as CPU cores and GPUs. One of the main benefits of this approach is that the underlying parallelism is exposed automatically as the runtime system gradually traverses the resulting task graph.

:Content: The course mainly focuses on the task-pragmas implemented in the newer incarnations of OpenMP. Other task-based runtime systems, e.g., StarPU, and GPU offloading are briefly discussed.

:Format: The course will be three half-days and comprises of lectures and hands-on sessions. This is an online-only course (Zoom).

:Audience: ...

:Date and Time: 2021-05-{10,11,12}, 9:00-12:00

:Location: Online through Zoom

:Instructors: Pedro Ojeda-May (pedro.ojeda-may@umu.se)

:Original author: Mirko Myllykoski.

:Recordings: ...

:Registration: ...

:Acknowledgment: This material was orignally created by Mirko Myllykoski (mirkom@cs.umu.se), and later modified.

.. prereq::

 - Basic knowledge of C programming language.
 - Basic knowledge of parallel programming.
 - Basic Linux skills.
 - Basic knowledge of OpenMP is beneficial but not required.

.. toctree::
   :caption: Day 1
   :maxdepth: 1

   hpc2n-intro
   setup
   openmp-basics1
   openmp-basics2

.. toctree::
   :caption: Day 2
   :maxdepth: 1
   
   motivation
   task-basics-1
   task-basics-2
   task-basics-lu

.. toctree::
   :caption: Day 3
   :maxdepth: 1
   
   two-sided-algorithms
   starpu1
   starpu2
   starpu-lu

.. toctree::
   :caption: Miscellanous
   :maxdepth: 1
   
   quick-reference
   guide
