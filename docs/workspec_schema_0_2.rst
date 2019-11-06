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

.. rubric:: Option

Add an additional *shape* attribute to the serialized record.


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

Reference implementation
========================

A reference implementation in Python can heavily rely on the ``json`` module,
supplemented through the *object_hook* and *object_pairs_hook* to the
``json.JSONDecoder``.