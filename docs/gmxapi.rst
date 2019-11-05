gmxapi 0.1 specification
========================

``gmxapi`` operations
---------------------

Operation namespace: gmxapi


.. rubric:: operation: make_input

.. versionadded:: gmxapi_graph_0_2

Produced by :py:func:`gmx.make_input`

* ``input`` ports

  - ``params``
  - ``structure``
  - ``topology``
  - ``state``

* ``output`` ports

  - ``params``
  - ``structure``
  - ``topology``
  - ``state``


.. rubric:: operation: md

.. versionadded:: gmxapi_workspec_0_1

.. deprecated:: gmxapi_graph_0_2

Produced by :py:func:`gmx.workflow.from_tpr`

Ports:

* ``params``
* ``depends``


.. rubric:: operation: modify_input

.. versionadded:: gmxapi_graph_0_2

Produced by :py:func:`gmx.modify_input`

* ``input`` ports

  - ``params``
  - ``structure``
  - ``topology``
  - ``state``

* ``output`` ports

  - ``params``
  - ``structure``
  - ``topology``
  - ``state``


``gromacs`` operations
----------------------

Operation namespace: gromacs


.. rubric:: operation: load_tpr

.. versionadded:: gmxapi_workspec_0_1

.. deprecated:: gmxapi_graph_0_2

Produced by :py:func:`gmx.workflow.from_tpr`


.. rubric:: operation: mdrun

.. versionadded:: gmxapi_graph_0_2

Produced by :py:func:`gmx.mdrun`

* ``input`` ports

  - ``params``
  - ``structure``
  - ``topology``
  - ``state``

* ``output`` ports

  - ``trajectory``
  - ``conformation``
  - ``state``

* ``interface`` ports

  - ``potential``


.. rubric:: operation: read_tpr

.. versionadded:: gmxapi_graph_0_2

Produced by :py:func:`gmx.read_tpr`

* ``input`` ports

  - ``params`` takes a list of filenames

* ``output`` ports

  - ``params``
  - ``structure``
  - ``topology``
  - ``state``


Extension API
=============

Extension modules provide a high-level interface to gmxapi operations with functions
that produce Operation objects. Operation objects maintain a weak reference to the
context and work graph to which they have been added so that they can provide a
consistent proxy interface to operation data. Several object properties provide
accessors that are forwarded to the context.

.. These may seem like redundant scoping while operation instances are essentially
   immutable, but with more graph manipulation functionality, we can make future
   operation proxies more mutable. Also, we might add extra utilities or protocols
   at some point, so we include the scoping from the beginning.

``input`` contains the input ports of the operation. Allows a typed graph edge. Can
contain static information or a reference to another gmxapi object in the work graph.

``output`` contains the output ports of the operation. Allows a typed graph edge. Can
contain static information or a reference to another gmxapi object in the work graph.

``interface`` allows operation objects to bind lower-level interfaces at run time.

Connections between ``input`` and ``output`` ports define graph edges that can be
checkpointed by the library with additional metadata.

Python interface
================


:py:func:`gmx.read_tpr` creates a node for a ``gromacs.read_tpr`` operation implemented
with :py:func:`gmx.fileio.read_tpr`

:py:func:`gmx.mdrun` creates a node for a ``gromacs.mdrun`` operation, implemented
with :py:func:`gmx.context._mdrun`

:py:func:`gmx.init_subgraph`

:py:func:`gmx.while_loop` creates a node for a ``gmxapi.while_loop``


Work graph procedural interface
-------------------------------

Python syntax available in the imported ``gmx`` module.

..  py:function:: gmx.commandline_operation(executable, arguments=[], input=[], output=[])

    .. versionadded:: 0.0.8

    lorem ipsum

..  py:function:: gmx.get_context(work=None)
    :noindex:

    .. versionadded:: 0.0.4

    Get a handle to an execution context that can be used to launch a session
    (for the given work graph, if provided).

..  py:function:: gmx.logical_not

    .. versionadded:: 0.1

    Create a work graph operation that negates a boolean input value on its
    output port.

..  py:function:: gmx.make_input()
    :noindex:

    .. versionadded:: 0.1

..  py:function:: gmx.mdrun()

    .. versionadded:: 0.0.8

    Creates a node for a ``gromacs.mdrun`` operation, implemented
    with :py:func:`gmx.context._mdrun`

..  py:function:: gmx.modify_input()

    .. versionadded:: 0.0.8

    Creates a node for a ``gmxapi.modify_input`` operation. Initial implementation
    uses ``gmx.fileio.read_tpr`` and ``gmx.fileio.write_tpr``

..  py:function:: gmx.read_tpr()

    .. versionadded:: 0.0.8

    Creates a node for a ``gromacs.read_tpr`` operation implemented
    with :py:func:`gmx.fileio.read_tpr`

..  py:function:: gmx.gather()

    .. versionadded:: 0.0.8

..  py:function:: gmx.reduce()

    .. versionadded:: 0.1

    Previously only available as an ensemble operation with implicit reducing
    mode of ``mean``.

..  py:function:: gmx.run(work=None, **kwargs)
    :noindex:

    Run the current work graph, or the work provided as an argument.

    .. versionchanged:: 0.0.8

    ``**kwargs`` are passed to the gmxapi execution context. Refer to the
    documentation for the Context for usage. (E.g. see :py:class:`gmx.context.Context`)

..  py:function:: gmx.init_subgraph()

    .. versionadded:: 0.1

    Prepare a subgraph. Alternative name: ``gmx.subgraph``

..  py:function:: gmx.tool

    .. versionadded:: 0.1

    Add a graph operation for one of the built-in tools, such as a GROMACS
    analysis tool that would typically be invoked with a ``gmx toolname <args>``
    command line syntax. Improves interoperability of tools previously accessible
    only through :py:func:`gmx.commandline_operation`

..  py:function:: gmx.while_loop()

    .. versionadded:: 0.1

    Creates a node for a ``gmxapi.while_loop``
