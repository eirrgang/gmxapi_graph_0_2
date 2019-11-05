=========================
Work specification schema
=========================

..  contents:: Draft version 4 (2 November 2019)
    :local:
    :depth: 2

Documentation by example
========================

.. rubric:: Example

Simple simulation using :ref:`data sources <simulation input>` produced by previously defined work::

   {
       "version": "gmxapi_graph_0_2",
       "elements":
       {
           "mdrun_<hash>":
           {
               "namespace": "gmxapi",
               "operation": "mdrun",
               "input":
               {
                  "parameters": "simulation_parameters_<hash>",
                  "simulation_state": "initial_state_<hash>",
                  "microstate": "initial_coordinates_<hash>"
               },
               "output":
               {
                  "trajectory": "gmxapi.Trajectory",
                  "parameters": "gmxapi.Mapping"
               },
               "depends": []
           }
   }

.. rubric:: Example

This example exercises more of the :ref:`grammar` in a complete, self-contained
graph.

Simulation reading inputs from the filesystem, with an attached restraint from a
pluggable extension module::

   {
       "version": "gmxapi_graph_0_2",
       "elements":
       {
           "read_tpr_<hash>":
           {
               "label": "tpr_input",
               "namespace": "gmxapi",
               "operation": "read_tpr",
               "input":
               {
                  "filename": ["topol.tpr"]
               },
               "output":
               {
                  "parameters": "gmxapi.Mapping",
                  "simulation_state": "gmxapi.Mapping",
                  "microstate": "gmxapi.NDArray"
               }
           }
           "mdrun_<hash>":
           {
               "label": "md_simulation_1",
               "namespace": "gmxapi",
               "operation": "mdrun",
               "input": "read_tpr_<hash>",
               "depends": ["ensemble_restraint_<hash>.interface.potential"]
           }
           "ensemble_restraint_<hash>":
           {
               "label": "ensemble_restraint_1",
               "namespace": "myplugin",
               "operation": "ensemble_restraint",
               "input":
               {
                   "params": {},
               },
               "interface": ["gromacs.restraint"]
           }
       }
   }

.. rubric:: Example

Illustrate the implementation of the command line wrapper.

The *gmxapi* Python package contains a helper :py:func:`gmxapi.commandline_operation`
that was implemented in terms of more strictly defined operations.
The :py:func:`gmxapi.commandline.cli` operation is aware only of an arbitrarily
long array of command line arguments. The wrapper script constructs the
necessary graph elements and data flow to give the user experience of files
being consumed and produced, though these files are handled in the framework
only as strings and string futures.

Graph node structure example::

    {
        "version": "gmxapi_graph_0_2",
        "elements":
        {
            "filemap_aaaaaa": {
                "namespace": "gmxapi",
                "operation": "make_map",
                "input": {
                    "-f": ["some_filename"],
                    "-t": ["filename1", "filename2"]
                },
                "output": {
                    "file": "gmxapi.Mapping"
                }
            },
            "cli_op_aaaaaa": {
                "label": "exe1",
                "namespace": "gmxapi",
                "operation": "cli",
                "input": {
                    "executable": ["some_executable"], # list length gives data edge width
                    "arguments": [[]], # Nested list allows disambiguation of array data within a single ensemble member.
                    "input_file_arguments": "filemap_aaaaaa",
                    # Complex values can use indirection to helper operations
                    # to reduce parsing complexity.
                    # Alternatively,
                    # we could make parsing recursive and allow arbitrary nesting
                    # with special semantics for dictionaries (as well as lists)
                },
                "output": {
                    "file": "gmxapi.Mapping"
                }
            },
            "filemap_bbbbbb: {
                "label": "exe1_output_files",
                "namespace": "gmxapi",
                "operation": "make_map",
                "input": {
                    "-in1": "cli_op_aaaaaa.output.file.-o",
                    "-in2": ["static_fileB"],
                    "-in3": ["arrayfile1", "arrayfile2"] # matches dimensionality of inputs
                }
            },
            "cli_op_bbbbbb": {
                "label": "exe2",
                "namespace": "gmxapi",
                "operation": "commandline",
                "input": {
                    "executable": [],
                    "arguments": [],
                    "input_file_arguments": "filemap_bbbbbb"
                },
            },

        }
    }

.. rubric:: Example

Subgraph specification and use. Illustrate the toy example of the subgraph test.

The *gmxapi.test* module contains the following code::

    import gmxapi as gmx

    @gmx.function_wrapper(output={'data': float})
    def add_float(a: float, b: float) -> float:
        return a + b

    @gmx.function_wrapper(output={'data': bool})
    def less_than(lhs: float, rhs: float) -> bool:
        return lhs < rhs

    def test_subgraph_function():
        subgraph = gmx.subgraph(variables={'float_with_default': 1.0, 'bool_data': True})
        with subgraph:
            # Define the update for float_with_default to come from an add_float operation.
            subgraph.float_with_default = add_float(subgraph.float_with_default, 1.).output.data
            subgraph.bool_data = less_than(lhs=subgraph.float_with_default, rhs=6.).output.data
        operation_instance = subgraph()
        operation_instance.run()
        assert operation_instance.values['float_with_default'] == 2.

        loop = gmx.while_loop(operation=subgraph, condition=subgraph.bool_data)
        handle = loop()
        assert handle.output.float_with_default.result() == 6

This could be serialized with something like the following, by separating the
concrete primary work graph from the abstract graph defining the data flow in
the subgraph. Note that a subgraph description is a special case of the
description of a fused operation, which we may need to explore when considering
how Context implementations may support dispatching between environments that
warrant different sorts of optimizations. We should also consider the Google
"protocol buffer" and gRPC syntax and semantics.

::

    {
        "concrete_graph_<hash>":
        {
            "version": "gmxapi_graph_0_2",
            "elements":
            {
                "while_loop_<hash>":
                {
                    "namespace": "gmxapi",
                    "operation": "while_loop",
                    "input":
                    {
                        "operation": ".abstact_graph_<hash>"
                    },
                    "depends": [".abstract_graph_<hash>.interface.bool_data"],
                    "output":
                    {
                        "float_with_default": "gmxapi.Float64",
                        "bool_data": "gmxapi.Bool"
                    }
                }
            }
        "abstract_graph_<hash>":
            {
                "input":
                {
                    "float_with_default": 1.0,
                    "bool_data": True
                },
                "output":
                {
                    "float_with_default": "add_float_<hash>.output.data",
                    "bool_data": "less_than_<hash>.output.data"
                },
                "elements":
                {
                    "less_than_<hash>":
                    {
                        "namespace": "gmxapi.test",
                        "operation": "less_than",
                        "input":
                        {
                            "lhs": "add_float_<hash>.output.data",
                            "rhs": [[6.]]
                        }
                        "output":
                        {
                            "data": "gmxapi.Bool"
                        }
                    }
                    "add_float_<hash>":
                    {
                        "namespace": "gmxapi.test",
                        "operation": "add_float",
                        "input":
                        {
                            "a": ".abstract_graph_<hash>.float_with_default",
                            "b": [[1.]]
                        }
                        "output":
                        {
                            "data": "gmxapi.Float64"
                        }
                    }
                }
            }
        }
    ]

Goals
=====

- Serializeable representation of a molecular simulation and analysis workflow
  that is

  - complete enough for abstractly specified work to be unambiguously translated to API calls, and
  - simple enough to be robust to API updates and uncoupled from implementation details.
- Facilitate easy integration between independent but compatible implementation code in Python or C++.
- Support verifiable compatibility with a given API level.
- Provide enough information to uniquely identify the "state" of deterministic inputs and outputs.

For the last point, the meaning of "deterministic" is explored in the following
discussions on uniqueness, deduplication, independent trials, and checkpointing.

Terms (more clarification needed)
=================================

These two terms are borrowed from TensorFlow:

Context
  Abstraction for the entity that maps work to a computing environment.

Session
  Abstraction for the entity representing work that is executing on resources
  allocated by an instance of a Context implementation.

The above terms roughly map to terms like *Executor* and *Task* in other frameworks.
Distinctions relate to the lifetime of the *Context* instance, and the fact that
it owns both the work specification (including operation and data handles)
and the computing resources.
The *Context* instance owns resources (on behalf of the client) that may
otherwise be owned directly by the client, and so its lifetime must span all
references to resources, operation handles, and data futures.

Operation
  A well defined computational element or data transformation that can be used
  to add computational work to a graph managed by a Context. Operation inputs
  are strongly specified, and behavior for a given set of inputs is deterministic
  (within numerical stability). Operation outputs may not be well specified
  until inputs are bound.

Operation instance / reference / handle
  A node in a work graph. Previously described as *WorkElement*.

Element
  Another term used to name work nodes or operation instances.

Operation factory / helper
  The syntax of UI-level functions that instantiate operations is specified by
  the API, but can extend the syntax implied by the serialized representation
  of a node for flexibility and user-friendliness.

port
  Generic term for a named source, sink, resource, or binding hook on a node.

resource
  Describes an API hook for an interaction mediated by a Context. Data flow
  is described as *immutable* resources (generally produced as Operation outputs)
  that can be consumed by binding to Operation inputs or by extracting as *result*s
  from the API. Some interactions cannot be represented in terms of producers
  and subscribers of immutable data events: *Mutable* resources cannot be
  managed by the Context as data events and require different work scheduling
  policies that either (a) allows arbitrary (unscheduled) call-back through the API framework,
  (b) dispatch the mutable resource collaboration to another Context, or (c)
  allow operations to bind and interact with an interface not specified by the
  API or not known to the responsible Context implementation. Examples include
  the Context-provided *ensemble_reduce* functionality, the ensemble simulation
  signaling facility (by which extension code can terminate a simulation early),
  and the binding mechanism by which MD extension code can be attached to an
  *MD* operation as a plugin. The nature of a resource is indicated by the
  namespace of its *port* in the work record.

Concrete graph definition and state of execution
================================================

Launch and relaunch: recoverability
-----------------------------------

To be able to recover the state of an executing graph after an interruption,
we need to be able to

1. identify whether or not work has been partially completed, and
#. reconcile checkpoint data for graph nodes and edges, which may not all (at least initially) be on the same computing
   host.

Discoverability of work graph state
-----------------------------------

We need to robustly discover and characterize data and checkpointing artifacts
to minimize unnecessary computation and data while supporting scientifically
relevant reproducible results (for an acceptable definition of "reproducible").

Due to numerical optimizations, molecular simulation results for the exact same
inputs and parameters may not produce output that is binary identical,
but which should be treated as scientifically equivalent.
We need to be able to identify equivalent rather than identical output.
Input that draws from the results of a previous operation should be able to verify whether
valid results for any identically specified operation exists, or at what state it is in progress.

Contrast this with the need to distinguish between similar results that represent independent trials.
Briefly, this means tracking data that users (and application developers) may not
be accustomed to tracking, such as pseudo-random initialization (PRNG seeds) or dynamic input.

Granularity versus abstraction
------------------------------

The degree of granularity in the work specification has ideological and practical
implications, affecting

* room for optimization,
* the amount of data in the work specification,
* its human-readability / editability, and
* the amount of additional metadata that needs to be stored in association with a Session.

Graph state versus work state
-----------------------------

If one element is added to the end of a work specification, results of the previous operations should not be
invalidated.

If an element at the beginning of a work specification is added or altered, "downstream" data should be easily
invalidated.

Names and Namespaces
====================

Python, work graph serialization spec, and extension modules
------------------------------------------------------------

I need to work on expressing it more clearly (maybe through Sphinx formatting),
but it is important to note that there are three different concepts implied by
the prefixes to names used here.

Names starting with ``gmx.`` are symbols in the Python ``gmxapi`` package.
Names starting with ``gmxapi.`` are not Python names, but work graph operations
defined for gmxapi and implemented by a gmxapi compatible execution Context.

Names starting with ``gromacs.`` are also work graph operations, but are implemented
through GROMACS library bindings (currently ``gmxapi._gmxapi``).
They are less firmly specified because they
are dependent on GROMACS terminology, conventions, and evolution.
Operations implemented by extension modules use a namespace equal to their importable module name.

The Context implementation in the Python package implements the runtime aspects
of gmxapi operations in submodules of *gmxapi*, named (hopefully conveniently) the
same as the work graph operation or ``gmx`` helper function.

The procedural interface in the :py:mod:`gmxapi` module provides helper functions that produce handles to work graph
operations and to simplify more involved API tasks.

Operation and Object References
-------------------------------

Entities in a work graph also have (somewhat) human readable names with nested
scope indicated by ``.`` delimiters. Within the scope of a work node, namespaces
distinguish several types of interaction behavior. (See :ref:`grammar`.)
Within those scopes, operation definitions specify named "ports" that are
available for nodes of a given operation.
Port names and object types are defined in the API spec (for operations in the ``gmxapi``
namespace) and expressed through the lower level API.

The ports for a work graph node are accessible by proxy in the Python interface,
using correspondingly named nested attributes of a Python reference to the node.

Note: we need a unique identifier and a well defined scheme for generating them so
that the API can determine data flow, tag artifacts, and detect complete or partially
complete work. We have separated work node *name* into *uid*
and *label*, where *label* is a non-canonical and non-required part of a work
graph representation.

Expressing interfaces
---------------------

Operation outputs and other interfaces are listed as key-value pairs of *port*
name and interface type. The interface type may be a data type specified by
gmxapi, or some other resource defined in the prefixed namespace. Interfaces
may represent mutable or immutable resources. The deserializer is responsible
for acquiring appropriate binding code from the source operation implementation,
sink operation implementation, and (if distinct) the module defining the
resource type.

By convention, immutable data interfaces are named with capital letters to be
consistent with naming conventions for data classes, but this is not (yet) a
requirement.

Middleware API: work record
===========================

The work specification record is a hierarchical associative data structure that is easily represented as an unordered
Python dictionary, or serialized to common structured text formats.
The canonical serialization format is valid JSON serialized data, restricted to the *latin-1* character set,
encoded in *utf-8*.

Uniqueness
----------

Goal: results should be clearly mappable to the work specification that led to them, such that the same work could be
repeated from scratch, interrupted, restarted, etcetera, in part or in whole, and verifiably produce the same results
(which can not be artificially attributed to a different work specification) without requiring recomputing intermediate
values that are available to the Context.

The entire record, as well as individual elements, have a well-defined hash that can be used to compare work for
functional equivalence.

State is not contained in the work specification, but state is attributable to a work specification.

If we can adequately normalize utf-8 Unicode string representation, we could checksum the full text,
but this may be more work than defining a scheme for hashing specific data or letting each operation define its own
comparator.

Question: If an input value in a workflow is changed from a verifiably consistent result to an equivalent constant of a
different "type", do we invalidate or preserve the downstream output validity? E.g. the work spec changes from
"operationB.input = operationA.output" to "operationB.input = final_value(operationA)"

The question is moot if we either only consider final values for terminating execution or if we know exactly how many
iterations of sequenced output we will have, but that is not generally true.

Maybe we can leave the answer to this question unspecified for now and prepare for implementation in either case by
recording more disambiguating information in the work specification (such as checksum of locally available files) and
recording initial, ongoing, and final state very granularly in the session metadata. It could be that this would be
an optimization that is optionally implemented by the Context.

It may be that we allow the user to decide what makes data unique. This would need to be very clearly documented, but
it could be that provided parameters always become part of the unique ID and are always not-equal to unprovided/default
values. Example: a ``load_tpr`` operation with a ``checksum`` parameter refers to a specific file and immediately
produces a ``final`` output, but a ``load_tpr`` operation with a missing ``checksum`` parameter produces non-final
output from whatever file is resolved for the operation at run time.

It may also be that some data occurs as a "stream" that does not make an operation unique, such as log file output or
trajectory output that the user wants to accumulate regardless of the data flow scheme; or as a "result" that indicates
a clear state transition and marks specific, uniquely produced output, such as a regular sequence of 1000 trajectory
frames over 1ns, or a converged observable. "result"s must be mapped to the representation of the
workflow that produced them. To change a workflow without invalidating results might be possible with changes that do
not affect the part of the workflow that fed those results, such as a change that only occurs after a certain point in
trajectory time. Other than the intentional ambiguity that could be introduced with parameter semantics in the previous
paragraph,

Serialization
-------------

The work graph has a basic grammar and structure that maps well to basic data structures.
We use JSON for serialization of a Python dictionary.

Integers and floating point numbers are 64-bit.

The JSON data should be utf-8 compatible, but note that JSON codecs probably map Unicode string
objects on the program side to un-annotated strings in the serialized data
(encoding is at the level of the entire byte stream).

Names (labels and UIDs) in the work graph are strings from the ASCII / Latin-1 character set.
Periods (``.``) have special meaning as delimiters.

Bare string values are interpreted as references to other work graph entities
or API facilities known to the Context implementation.
Strings in lists are interpreted as strings.

TODO:
*Define the deterministic way to identify a work graph and its artifacts for
persistence across interruptions and to avoid duplication of work...*

.. _grammar:

Grammar
~~~~~~~

.. rubric:: Input value.

Inputs appear as key-value pairs (expressed in JSON format in this document) for
which the key is a string and the value is either literal data, a collection,
or a reference to another graph entity.
In `JSON <http://www.json.org>`_ serialized form, the value is
either an *array*, an *object*, or a *string* with constraints described below.

Literal data is serialized as arrays of integers,
floating point numbers, strings, or other arrays.
The structures formed by
nested arrays must have regular shape and uniform type.

.. note:: All data has shape. There are no bare scalars, since they can be
   represented as arrays of shape ``(1,)``.

.. todo:: How should we optimize arrays of strings? We could let arrays contain
   references to long strings defined as separate 1-dimensional objects, but
   that would include expanding the schema to allow arrays of references, which
   we have avoided in the current document because of the challenges of
   disambiguating strings from references in the serialized form.

References to other entities in the graph are presented as bare string literals
with the following grammar constraints.

::

    reference
        nestedobject
        nestedobject delimiter label

    nestedobject
        objectname delimiter objectname
        nestedobject delimiter objectname

    delimiter
        '.'

The following
definitions clarify two forms of element used in string-based naming. *objectname*
strings have stricter requirements because they are likely to directly map to
coding constructs, whereas *label* strings are likely to appear only as keys to
associative mappings and may have more relaxed rules. Specifically, *objectname*
must begin with a letter and may not contain hyphens.
Some additional symbols are omitted for conciseness.
These are *string* (a sequence of characters from the *latin-1* character set),
*integer*, and *letter* (the 52 alphabetic characters from *latin-1* in the
contiguous blocks 'a' - 'z' and 'A' - 'Z').

::

    objectnamecharacter
        '_'
        letter
        integer
        ""

    objectnamecharacters
        objectnamecharacter objectnamecharacters

    objectname
        letter
        letter objectnamecharacters

    subscript
        '[' label ']'

    labelcharacter
        '-'
        '_'
        letter
        integer

    labelcharacters
        labelcharacter
        labelcharacter labelcharacters

    label
        labelcharacters
        label subscript


Embedding references in structured data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Initially, *gmxapi_graph_0_2* assumes that references are not contained within
serialized value structures. Where necessary, we can use helper operations to
create new references composed from combinations of lower-dimensional references
and literal data.

This section discusses future options for distinguishing string literals from
node references in nested structures.

Consider the following serialized object member::

    "value": ["mdrun1.output.trajectory", "mdrun2.output.trajectory"]

How do we determine whether this is a ``(2,)`` shaped reference to two trajectory
outputs, versus a list of two strings?

Is the operation definition considered when deserializing object members or
should the serialized data be unambiguous without context? This question is
relevant to the determination of the shape and dimensionality of resources.

.. rubric:: Option

Require all literals to have a final dimension of size unity. In order to avoid
ambiguity, empty dimensions beyond this requirement must be removed.

There are no bare scalar constants: all data has *shape*.
A single scalar value is the sole element of data with shape ``(1,)``
and therefore appears with at least one pair of enclosing list
delimiters, e.g. ``"value": [42]``

``"value": "mdrun1.output.trajectory"`` unambiguously refers to another graph
entity.

``"value": ["mdrun1.output.trajectory"]`` refers to a single string literal.

``"value": ["mdrun1.output.trajectory", "mdrun2.output.trajectory"]`` is a list
of two references.

``"value": [["mdrun1.output.trajectory"], ["mdrun2.output.trajectory"]]`` is a
list of two string literals.

``"value": ["mdrun1.output.trajectory", ["mdrun2.output.trajectory"]]`` is a
list of one reference and one string literal.

``"value": [["mdrun1.output.trajectory"]]`` is invalid.

``"value": [["mdrun1.output.trajectory", ["mdrun2.output.trajectory"]]]`` is invalid.

``"value": 42`` is invalid.

``"value": [42]`` is an integer with shape ``(1,)``.

``"value": [[42]]`` is invalid.

Caveats:

* The dimensionality of an input's serialized record cannot indicate the graph
  topology without details of the operation implementation.
* Scatter operations cannot be well defined until the consumer(s) are known.
  This likely means that ``scatter`` is not truly an operation, but, at best,
  an annotation.

.. rubric:: Option

Require all string literals to be enclosed in a 1-element list.


Additional cases:

Prevent broadcasting?

Force scatter with broadcast? (E.g. send each element of a (10,) array to 10
consumers, each of which consumes an array of 10 values.)

A data dimension must be populated

..
   It also simplifies uniqueness checks, where ``["string"]`` and ``[["string"]]``
   (or ``0``, ``[0]``, and ``[[0]]``) would otherwise need to be parsed as equivalent.

Topology
~~~~~~~~

The topology of the graph data is well defined in the serialized record.
API handles may have implicit higher dimensions accommodating parallel computation,
but the graph data dimensions are explicitly represented in both operation
input and output.

Schema
~~~~~~

When an element is being evaluated for deserialization / instantiation, the
*namespace* and *operation* are looked up in the API registry for a dispatching
factory function. If no registry entry is found, attempts to *import* an
operation implementation, attempting to treat *operation* as an importable
entity relative to a *namespace* module.

The work graph record contains two top-level keys.

version
  Schema version.

  .. versionchanged:: 0.1
     Second generation work specification schema denoted by the *version* string
     *gmxapi_graph_0_2*

elements
  Associative map of node specifications, keyed by *uid*.

Each *element* contains the following (required) keys.

namespace
  Scope of the operation implementation. Interpreted as an importable module in
  a Python Context.

operation
  Name of an Operation. Used to determine the registration key for an operation
  implementation, the name of the Operation helper function, and the *uid*
  prefix for nodes. For Python Contexts, assumed to be an importable entity from
  *namespace*

input
  .. versionadded:: 0.1

  Immutable data sources. Either a dictionary (keyed by the Operation's named
  inputs) or a string reference to another graph element with a compatible
  output interface.

depends
  .. versionchanged:: 0.1

  List of entities with which the operation director code will be given a chance
  to *bind* when launching work. Constrains the sequence with which nodes are
  processed.
  *TODO: deprecate?* This is left over from the first generation work
  specification. It may contain redundant information as we transition to
  explicit *input* and *output*, and is not particularly evocative with regard
  to binding mutable resources.

Each *element* may contain the following (optional) keys.

label
  .. versionadded:: 0.0.8

  A human-readable, user-provided node name that allows convenient look-up of
  context-managed resources.
  It must be unique in a Context,
  but does not affect the uniqueness of the node outputs.

Depending on the operation implementation and instance, an *element* may contain
the following keys.

output
  .. versionadded:: 0.1

  Names and types of the (immutable) data sources generated by the node. For
  various reasons, the exact names and types of operation outputs cannot always
  be known until the node is created (operation is instantiated). The output
  names and types can be used for validation when adding dependent operations
  to the graph.

interface
  .. versionadded:: 0.1

  List of named *ports* providing mutable resources. For instance, MD extension
  code may advertise itself as a pluggable force calculation with a
  *interface.potential* port.

gmxapi_workspec_0_1
"""""""""""""""""""

.. versionadded:: 0.0.0

    Operation instantiation is mediated during Session launch by the *depends*
    field of each element. The binding protocol is unspecified, but a dependent
    node builder is *subscribed* to the builder of the dependency before the
    builders are called in topologically valid order, as determined by the DAG
    implied by the *depends* network.

.. seealso::

   `DOI 10.1093/bioinformatics/bty484 <https://doi.org/10.1093/bioinformatics/bty484>`_

gmxapi_graph_0_2
""""""""""""""""

.. versionchanged:: 0.1

    Inputs, outputs, and other interfaces are explicitly represented in the
    data structure.
    Input ports names and types are specified by the API. Bound arguments are
    included in the record.
    Output ports are determined by querying the operation, so the available keys
    and types are included in the record.

Note that, in the examples, *element* keys are calculated deterministically
by the framework to uniquely identify a node (and its output) in terms of a
specified operation behavior and the inputs to the node.

Angle brackets and the names they enclose (e.g. *<symbol>*) are not literal,
representing variable data or values explained in this text.

*<hash>* indicates a MIME-like (latin-1 compatible, base-64 encoded) string
representation of the unique features of the operation node. This value is
calculated by the Context with help from the Operation definition.

.. _simulation input:

Simulation input
""""""""""""""""

The API conventions allow for specification of certain hierarchical data for
collaborating operations. For instance, we currently expect that a simulation
operation like *mdrun* accepts, as a complete input pack, the output of operations
such as *modify_input* or *read_tpr*. Such a standardized pack is defined by a
consistent set of data names and types.

Note that *simulation_state* is a mutable internal aspect of *mdrun* that must
be checkpointed, but that is a detail of the operation implementation in a
particular Context. Its exposure in the work graph indicates the immutable data
with which the operation is initialized when the initial work graph state is
established.

.. todo:: Revise definition of simulation input data wrt microstate vs. molecular force field.

   We had previously tentatively settled on the following components of the data
   represented by the pair of TPR file data and simulation checkpoint data.

   * parameters: simulation parameters that define the computational algorithm to apply
   * simulation_state: the stateful data of the MD implementation not usually
     provided as explicit inputs
   * structure or conformation: the atomic data and/or molecular primary structure configuration
   * topology: the molecular force field data

   The last two bullets are problematic because the data structures are generally
   coupled. It seems sensible to distinguish phase space data (microstate) from
   higher level model information,
   but it is not clear how best to divide information on atom typing, bonds,
   force field parameters, and additional force field metadata.

Deserialization heuristics
--------------------------

Deserialization requires at least two passes to produce a verifiably valid
in-memory work description.

First, elements must be individually processed from the associative data structure,
at which time the element dependencies can only be recorded.

Once all elements are read, a directed acyclic graph can be established using
the topology implied by the named inputs and outputs.

In the most naive implementation, we use a recursive search to pop elements from
the set of elements in topologically valid order. We can then apply the same
logic as is used when validating client input to build an always-valid DAG, one
element at a time. Specifically, nodes are not modifiable after addition, so
input dependencies must be resolvable when a node is added.

Immutable data resources are produced as outputs and consumed as inputs.
Additionally, some operations have interdependencies or data flow that cannot
be resolved at the level of the work graph. We refer to these interactions
collectively as *mutable* resources. For simplicity, we declare one operation
to be the provider of the resource, and other operations as subscribers.

This allows us to use the DAG topology to construct a graph of operation
Directors and subscription relationships.
Dependency order affects order of instantiation and the direction of binding
operations at session launch.

.. rubric:: Rules of thumb

* An element can not depend on another element that is not in the work specification.
  *Caveat: we probably need a special operation just to expose the results of a different work flow.*
* Dependency direction affects sequencing of Director calls when launching a session,
  but also may be used at some point to manage checkpoints or data flow state
  checks at a higher level than the execution graph.

Question: What do we want to say about the topology due to outputs that are
arrays? Generally, it is hard to know the size and shape of an array before the
operation executes. Can topology be dynamic? Should we insist that array
dimensionality must asserted when the node is created? Or are we simply not able
to scatter from arrays that are operation outputs?

Core API roles and collaborations
=================================

Interfaces and/or Base classes
------------------------------

OperationFactory
~~~~~~~~~~~~~~~~

An OperationFactory receives a Context and Input, and returns an OperationHandle to the caller.

Context
~~~~~~~

Variations include

* GraphContext that just builds a graph that can be serialized for deserialization by another context.
* LaunchContext that processes a graph to be run in appropriate OperationContexts. Produces a Session.
* OperationContext or ImmediateContext that immediately executes the implemented operation

NodeBuilder
~~~~~~~~~~~

``addResource()`` Configures inputs, outputs, framework requirements, and factory functions.
``build()`` returns an OperationHandle

Operation
~~~~~~~~~

The OperationHandle returned by a factory may be a an implementing object or some sort of wrapper or proxy object.
output() provides getters for Results.

Has a Runner behavior.

Result
~~~~~~

gmxapi-typed output data. May be implemented with futures and/or proxies. Provides
extract() method to convert to a valid local owning data handle.

Behaviors
---------

Launcher
~~~~~~~~

Callable that accepts a Session (Context) and produces a Runnable (Operation).
A special case of OperationDirector.

Runner
~~~~~~

Takes input, runs, and returns a Runner that can be called in the same way.

run() -> Runner
run(Runner::Resources) -> Runner

OperationDirector
~~~~~~~~~~~~~~~~~

Takes Context and Input to produce a Runner.

Use a Context to get one or more NodeBuilders to configure the operation in a new context.
Then return a properly configured OperationHandle for the context.

Graph
~~~~~

Can produce NodeBuilders, store directorFactories.

Can serialize / deserialize the workflow representation.

* ``serialize()``
* ``uid()``
* ``newOperation()``: get a NodeBuilder
* ``node(uid)``: getter
* ``get_node_by_label(str)``: find uid
* iterator

OutputProxy
~~~~~~~~~~~

Service requests for Results for an Operator's output nodes.

Input
~~~~~

Input is highly dependent on the implementation of the operation and the context in which
it is executing. The important thing is that it is something that can be interpreted by a DirectorFactory.

Arbitrary lists of arguments and keyword arguments can be accepted by a Python
module director to direct the construction of one or more graph nodes or to
get an immediately executed runner.

GraphNode or serialized Operation Input is accepted by a LaunchContext or
DispatchingContext.

A runner implementing the operation execution accepts Input in the form of
SessionResources.


Operations
----------

Each node in a work graph represents an instance of an Operation.
The API specifies operations that a gmxapi-compliant execution context *should* provide in
the ``gmxapi`` namespace.

All specified ``gmxapi`` operations are provided by the reference implementation in released
versions of the ``gmx`` package. ``gmx.context.Context`` also provides operations in the ``gromacs``
namespace. This support will probably move to a separate module, but the ``gromacs`` namespace
is reserved and should not be reimplemented in external software.

When translating a work graph for execution, the Context calls a factory function for each
operation to get a Director. A Python-based Context *should* consult an internal map for
factory functions for the ``gmxapi`` namespace. **TODO:** *How to handle errors?
We don't really want middleware clients to have to import ``gmx``, but how would a Python
script know what exception to catch? Errors need to be part of an API-specified result type
or protocol, and/or the exceptions need to be defined in the package implementing the context.*


*namespace* is imported (in Python: as a module).

operation is an attribute in namespace that

..  versionadded:: 0.0

    is callable with the signature ``operation(element)`` to get a Director

..  versionchanged:: 0.1

    has a ``_gmxapi_graph_director`` attribute to get a Director

Helper
~~~~~~

Add operation instance to work graph and return a proxy object.
If proxy object has ``input`` or ``output`` attributes, they should forward ``getattr``
calls to the context... *TBD*

The helper makes API calls to the default or provided Context and then asks the Context for
an object to return to the caller. Generally, this is a proxy Operation object, but when the
context is a local context in the process of launching a session, the object can be a
graph Director that can be used to finish configuring and launch the execution graph.

Signatures

``myplugin.myoperation(arg0: WorkElement) -> gmx.Operation``

..  versionchanged:: 0.1

    Operation helpers are no longer required to accept a ``gmx.workflow.WorkElement`` argument.

``myplugin.myoperation(*args, input: inputobject, output: outputobject, **kwargs)``

    inputobject : dict
        Map of named input ports to typed gmxapi data, implicitly mappable Python objects,
        or objects implementing the gmxapi Output interface.

Some operations (``gmx.commandline``) need to provide an ``output`` keyword argument to define
data types and/or placeholders (not represented in the work graph).

    outputobject : dict
        Map of named output ports to

Additional ``args`` and ``kwargs`` may be used by the helper function to set up the work
graph node. Note that the context will not use them when launching the operation, though,
so ....


.. todo::

   Maybe let ``input`` and ``output`` kwargs be interpreted by the helper function, too,
   and let the operation node input be completely specified by ``parameters``?

   ``myplugin.myoperation(arg0: graph_ref, *args, parameters: inputobject, **kwargs)``

.. todo::

   I think we can go ahead and let ``gmx.Operation.input`` and ``gmx.Operation.output``
   implement ``get_item``...

Implementation note: the input and output attributes can have common implementations,
provided with Python "Descriptor"s

Servicing the proxy
~~~~~~~~~~~~~~~~~~~

When the Python client added the operation to the work graph, it used a helper function
to get a reference to an Operation proxy object. This object holds a weak reference to
the context and work graph to which it was added.


Factory
~~~~~~~

get Director for session launch

Director
~~~~~~~~

subscribable to implement data dependencies

``build`` method adds ``launch`` and ``run`` objects to execution graph.

To do: change ``build`` to ``construct``

Session callable
~~~~~~~~~~~~~~~~

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

Types
-----

Python classes for gmxapi object and data types.

..  py:class:: gmx.InputFile

    Proxy for

..  py:class:: gmx.NDArray

    N-dimensional array of gmxapi data.

..  py:class:: gmx.OutputFile

    Proxy for

Additional classes and specified interfaces
-------------------------------------------

We support Python duck-typing where possible, in that objects do not need to
inherit from a gmxapi-provided base class to be compatible with specified
gmxapi behaviors. This section describes the important attributes of specified
gmxapi interfaces.

This section also notes

* classes in the reference implementation that implement specified interfaces
* utilities and helpers provided to support creating gmxapi compatible wrappers

.. rubric:: Operation

Utilities
---------

..  py:function:: gmx.operation.make_operation(class, input=[], output=[])

    Generate a function object that can be used to manipulate the work graph
    _and_ to launch the custom-defined work graph operation.

    Example: https://github.com/kassonlab/gmxapi-scripts/blob/master/analysis.py

Reference implementation
------------------------

The ``gmxapi`` Python package implements *gmxapi* operations in the ``gmxapi.operation.Context``
reference implementation to support top-level *gmxapi* functions using various
``gmxapi`` submodules.

:py:func:`gmx.fileio.read_tpr` Implements :py:func:`gromacs.read_tpr`

Specification module
====================

Documentation for the specification should be extracted from the package.
It will be migrated to :py:mod:`gmxapi._workspec_0_2`.

The module (eventually) provides helpers, with utilities to validate API implementations / data structures / interfaces,
and (possibly) wrappers or factories.

The remaining content in this document is automatically extracted from the
:py:mod:`gmxapi._workspec_0_2` module. The above content can be migrated into this
module shortly, but the intent is that the module will also contain syntax and
schema checkers.

Specification: :py:mod:`gmxapi._workspec_0_2`
---------------------------------------------

..  automodule:: gmxapi._workspec_0_2
    :members:

Helpers: :py:mod:`gmxapi._workspec_0_2.util`
--------------------------------------------

..  automodule:: gmxapi._workspec_0_2.util
    :members:

Changes in second version
=========================

gmxapi_workspec_0_1 is described in the supplementary information for
`DOI 10.1093/bioinformatics/bty484
<https://doi.org/10.1093/bioinformatics/bty484>`_

Summary of changes proposed to the next document schema and API:

- Node inputs, outputs, and mutable interactions are explicitly named, so that
  typing and binding protocols can be specified by the API without complexity or
  ambiguity in the serialized document.
- Use the term "work graph" instead of "work specification".
  ``gmx.workflow.WorkSpec`` is replaced with an interface for a view to a work graph owned
  by a Context instance.
- Schema version string changes from ``gmxapi_workspec_0_1`` to ``gmxapi_graph_0_2``.
- ``gmx.workflow.WorkElement`` class is replaced with an interface definition
  for an instance of an Operation. Users no longer create objects representing
  work directly.
- Proposed: ``elements`` map name changed to ``nodes``?
- Various API specified operations are moved or renamed.
- User-provided ``name`` properties are replaced with two new properties.

  - ``label`` optionally identifies the entity to the user.
  - ``uid`` is a unique identifier that is deterministically generated by the API.
    ``uid`` completely and verifiably characterize an entity
    (in terms of its inputs and behavior)
    to facilitate
    reproducibility, optimization, and flexibility in graph manipulation.

- Proposed: Serialized work graphs may be bundled, nested as a map of named graphs. By
  convention, the graph named ``default`` would be examined as an entry point.
