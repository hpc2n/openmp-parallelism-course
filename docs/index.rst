OpenMP parallelism in scientific computing
==============================================

:Abstract: The purpose of the course is to learn when a code could benefit from task-based parallelism, and how to apply it. A task-based algorithm comprises of a set of self-contained tasks that have well-defined inputs and outputs. This differs from the common practice of organizing an implementation into subroutines in that a task-based implementation does not call the associated computation kernels directly, instead it is the role of a runtime system to schedule the task to various computational resources, such as CPU cores and GPUs. One of the main benefits of this approach is that the underlying parallelism is exposed automatically as the runtime system gradually traverses the resulting task graph.

:Content: The course mainly focuses on the task-pragmas implemented in the newer incarnations of OpenMP. 

:Format: Onsite course.

:Audience: ...

:Date and Time: 2021-05-{10,11,12}, 9:00-12:00

:Location: Online through Zoom

:Instructors: Pedro Ojeda-May (pedro.ojeda-may@umu.se)

:Recordings: ...

:Registration: ...

:Acknowledgment: This material is based on the course created by Mirko Myllykoski (mirkom@cs.umu.se), and the OpenMP course by Joachim Hein (Lund University).

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
   first-steps

.. toctree::
   :caption: Day 2
   :maxdepth: 1
   
   data-handling
   worksharing

.. toctree::
   :caption: Day 3
   :maxdepth: 1

   data-handlingii
   worksharingii

.. toctree::
   :caption: Day 4
   :maxdepth: 1
   
   motivation
   tasks

.. toctree::
   :caption: Miscellanous
   :maxdepth: 1
   
   quick-reference
   guide
