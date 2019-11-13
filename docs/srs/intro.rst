Introduction
============

Purpose
-------

The SCALE-MS project defines a framework in which computational molecular
science achieves new scalability by mapping complex research protocols to
large scale computing resources.

Document conventions
--------------------

This document uses a mixture of Python syntax and JSON syntax to represent
data structures.

Abstract syntax grammar may be described with
`BNF notation <https://www.w3.org/Notation.html>`__.

Concepts and terminology may overlap with existing technologies,
whereas the requirements specified here are intended to be agnostic to specific solutions.
Refer to the :doc:`glossary` for key terms and their intended meanings in the
context of this document.

Description
===========

Product perspective
-------------------

The scopes of several related efforts are still being clarified.

Initial data flow driven molecular simulation and analysis management tools
have been developed under the `gmxapi <http://gmxapi.org>`_ project,
now maintained with the `GROMACS <http://www.gromacs.org>`_ molecular simulation
package.

The `RADICAL <https://radical-cybertools.github.io/>`_ project has developed a
software stack that we hope to leverage.

Implementation details for SCALE-MS API functionality may require extension of
external tools, such as RADICAL or `Parsl <http://parsl-project.org/>`_.

Product functions/features
--------------------------

The API under development supports arbitrary complex and adaptive
computational molecular science research protocols on large and varied
computing resources.

Through its implementation library or integrations, the API must support

* Scriptability of multi-step multi-tool variably parallel computation and analysis.
* Data flow driven work, compartmentalized as nodes on a directed acyclic graph.
* Work specifications that are trivially portable to different computing environments.
* Full workflow checkpointing and optimal recoverability.
* Data abstractions that minimize the specification of artifact storage or transfer.
* Data locality optimizations that allow data transfers to be optimized or
  eliminated according to data dependencies.
* Data locality optimizations that allow work scheduling decisions to be made
  in terms of data dependencies.

User classes and characteristics
--------------------------------

Users are assumed to be molecular science researchers using Python scripts to
express and execute simulation and analysis work consisting of multiple
simulation and analysis tasks, using software tools from multiple packages.

Software tools are individually accessible as Python modules or as command line
tools.

Computational work may require multiple invocations (multiple HPC jobs) to complete.

The following classes of user are not necessarily mutually exclusive.

.. glossary::

    Basic user
        A researcher writing a Python script to control standard software.

    Advanced user
        A researcher who needs to integrate custom code into the scripted work.

    Pure Python user
        All software necessary for the work is importable as Python modules.

    Mixed command line user
        Some software is only accessible to Python by wrapping a command line driven tool.

    Compiled extension user
        Some software necessary for the work requires compilation and/or installation
        on the computing resource.

    Direct user
    Local user
        Work is executed directly in the process(es) launched by the user.
        Examples include a Python interpreter launched on the user's desktop or
        a script launched with :command:`mpiexec` in a terminal window.

    Indirect user
    Remote user
        A script run by the user dispatches serialized work through API-enabled
        middleware for deserialization and execution outside of the user's
        Python interpreter. In addition to remote execution systems, this class
        may include adapters to *container* systems or job queuing systems,
        whether or not the execution occurs on the same machine as the initial
        Python interpreter.

Operating environment
---------------------

Design and implementation constraints
-------------------------------------

User documentation
------------------

Assumptions and Dependencies
----------------------------
