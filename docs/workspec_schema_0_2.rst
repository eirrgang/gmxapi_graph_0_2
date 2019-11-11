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

State is not contained in the work specification (abstract graph), but state is attributable to a work specification.

If we can adequately normalize utf-8 Unicode string representation, we could checksum the full text (filtering
details that do not affect uniqueness),
but this may be more work than defining a scheme for hashing specific data or letting each operation define its own
comparator.

The present examples allow *uid* to consist of an arbitrary string that is verifiable with the help of operation
implementation details. By convention, the *uid* consists of a string of alphanumeric and underscore characters,
suffixed by a string-encoded hash. This allows some human readability of graph contents through examination of only
the element keys. It may seem inelegant to require string processing to extract the hash. The uid is only used for
look-up and equality testing, and does not need to be decoded. However, it may be preferable to assert that the uid
should be a valid SHA-256 hash, and allow serialization schemes individually to determine how the uid is serialized/deserialized,
allowing, for instance, the JSON scheme to encode a tuple of helpful prefix and uid hash with a safe delimiter
(a character not used in the string encoding scheme used for the SHA-256 hash).

.. admonition:: Question

    If an input value in a workflow is changed from a verifiably consistent result to an equivalent constant of a
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
    trajectory time.

Serialization conventions
-------------------------

The work graph has a basic grammar and structure that maps well to common basic data structures,
particularly in Python.
We use JSON for serialization of a Python dictionary.

Integers and floating point numbers are 64-bit.

The JSON data should be utf-8 compatible, but note that JSON codecs probably map Unicode string
objects on the program side to un-annotated strings in the serialized data
(encoding is at the level of the entire byte stream).

Names (labels and UIDs) in the work graph are strings from the ASCII / Latin-1 character set.
Periods (``.``) have special meaning as delimiters.

Some restrictions and special meanings are imposed on keys (object names or labels).

Object values represent a small number of structured data types with restrictions
noted below.

Data dimensionality and graph topology is unambiguous with minimal processing
apart from the underlying deserialization.

.. admonition:: TODO

    *Define the deterministic way to identify a work graph and its artifacts for
    persistence across interruptions and to avoid duplication of work. I.e. fingerprinting.*

.. _grammar:

Grammar
~~~~~~~

.. rubric:: Input values.

Inputs appear as key-value pairs (expressed in JSON format in this document) for
which the key is a string and the value is either literal data, a collection,
or a reference to another graph entity.
In `JSON <http://www.json.org>`_ serialized form, values are either *array* or
*object*.

JSON *objects* represent either "collections" or "meta" objects. "meta" objects have
a single member named "meta". Its value is an object with a single key that
determines how the meta object is to be processed, as documented below.
"Meta" objects are used to implement details that are otherwise not easily
represented in JSON form. "meta" is necessarily a reserved key word that may not
be used as an identifier for an *objectname*, *label*, or other user-facing entity.

Often, only one type of meta object makes sense in a particular situation, and
the nesting of a ``"meta": {...}`` member may seem superfluous. However, by
adopting this convention, we limit the growth in complexity of high-level parsing.
Parsers only need to look for a single key word ("meta") to dispatch handling
for standard or "meta-API" code paths.

Collections are mappings of keys to values. They are represented as JSON *objects*.
Keys must be strings, but are additionally subject to limitations described below.
A JSON *object* is treated as a collection if and only if it does not contain a
"meta" key.

Literal data is serialized as arrays of integers,
floating point numbers, strings, or other arrays.
The structures formed by
nested arrays must have regular shape and uniform type,
with the following caveat.

JSON *objects* may occur in arrays with special meaning.
Specifically, internal references can be made to other entities present in the
graph or known to the Context.

.. note:: All data has shape. There are no bare scalars, since they can be
   represented as arrays of shape ``(1,)``.

.. todo:: How should we optimize arrays of strings? We could let arrays contain
   references to long strings defined as separate 1-dimensional objects, but
   that would include expanding the schema to allow arrays of references, which
   we have avoided in the current document because of the challenges of
   disambiguating strings from references in the serialized form.

.. todo:: We should explore whether additional specification is warranted to
   describe a meta-API for light-weight operations, generalizing the internal
   reference scheme. Object key-value pairs are processed as meta-data for
   light-weight operations, such as to implement references to other entities
   present in the graph or known to the Context.

References are made using "meta" objects. An object with the key "meta"
holds an object with a member ``"key": "reference"`` and a member named "value"
containing the string form of the reference. The string will be processed in the
Context to resolve an internal reference according to the grammar below.
A reference may refer to another entity in the graph or to another resource
knowable by the Context.

Collections do not appear in arrays. Instead, data dimensionality occurs
exclusively in the collection member values. Collections are represented as
JSON *objects*. As noted, a collection may not use the special key, "meta".

Array values obtained through a generic JSON deserializer will require multiple
passes to convert to a native binary data structure, and so may not be suitable
for handling large data. In such cases, it will be appropriate to replace arrays
with references to codec operations (with string-encoded binary values) or to
entities obtainable by the Context from outside of the JSON document.

.. rubric:: Reference values

References occur as special objects, either contained within *arrays* (see above)
or as standalone values.

In the case of JSON serialization, a reference string is obtained from a "meta"
object with a "reference" member, whose value is a string.

The string representation of a reference to an entity resolvable by the Context
(such as through another graph entity) is represented and interpreted using the
following grammar.

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

    objectnamecharacters
        objectnamecharacter
        objectnamecharacter objectnamecharacters

    objectname
        letter
        letter objectnamecharacters

    subscript
        '[' integer ']'

    hyphen
        '-'

    underscore
        '_'

    labelcharacter
        hyphen
        underscore
        letter
        integer

    labelcharacters
        labelcharacter
        labelcharacter labelcharacters

    label
        labelcharacters
        label subscript
        label delimiter label

.. rubric:: Output values and interfaces

Operation nodes express ownership of resources by enumerating *ports*, which
may be nested.

In JSON, *ports* are expressed as object members. A port *name* is used as a
key, and the value is either a meta object the port resource,
or a collection of nested named *ports*.

The *name* should be user-friendly, but may be almost any sequence of
*labelcharacters* that is unique in the scope of the node outputs and suitable
for reference, as described above.

The "output" port of the node is reserved for immutable resources. It may
describe an immutable type or a collection of nested outputs.

The key word "meta" is reserved, and may not be used as an output name.

The "interface" port of the node is used (by convention) for mutable resources,
or interfaces that the interpreting Context will not be responsible for
resolving into directed acyclic flow of immutable data events. References to
"interface" or nested ports warrant either coscheduling or dispatch/delegation
to another Context implementation.

.. rubric:: Resource metadata

A meta object with the key "resource" provides metadata for a resource.
Resource meta objects have a string-valued member "type" and an array-valued
member "shape".

"type" is an *objectname* that the Context is able to resolve as an API entry
point providing the operation interface and, thus, the various API-specified
helpers for describing and instantiating graph nodes.

"shape" is a sequence giving the size of each dimension from the outside in.

Example: A single scalar integer output::

    "output": { "meta": { "resource": { "type": "gmxapi.Integer64", "shape": [1] } } }

Example: Output from an MD ensemble simulation with 10 members::

    "output":
    {
        "parameters":
        {
            "meta":
            {
                "resource":
                {
                    "type": "gmxapi.Mapping",
                    "shape": [10]
                }
            }
        },
        "trajectory":
        {
            "meta":
            {
                "resource":
                {
                    "type": "gmxapi.simulation.Trajectory",
                    "shape": [10]
                }
            }
        }
    }

.. todo::   Note that "mapping"s and "collection"s may often be interchangeable, but in the
            current specification we do not require that the keys and value types of a
            Mapping are known before run time. This may not be tenable in the long run.
            Similarly, we need to clarify the situations under which we may and may not know
            the dimensionality or dimension sizes of array data before run time.

.. todo:: Special meaning for bare string values? We have not specified an
          interpretation for input object members with bare string values. We
          could allow automatic treatment of such members as references.

.. todo:: Labels as references? We are currently requiring that references use
          the explicit object reference structured grammar. Since we do not
          allow periods (``.``) to be used in *labels*, we could treat reference
          strings that do not contain periods as *labels* that must resolve in
          the current graph. This would probably be a lot of parsing burden, so
          the benefit would need to be clearer.

Topology
~~~~~~~~

The topology of the graph data is well defined in the serialized record.
API handles may have implicit higher dimensions accommodating parallel computation,
but the graph data dimensions are explicitly represented in both operation
input and output.

Dimensionality of an input value is either the dimensionality of an input array
or the dimensionality of a referenced resource.

Dimensionality of a resource is determined by its *shape* value. Note that the
type may describe a schema in terms of another dimensioned type. Resolution of
such a resource to a simple higher dimensional object is an implementation
detail, but dimensions added by resolving references or types are considered
nested, and therefore inner dimensions. If other data shaping needs to occur or
to be represented in the graph, then helper operations may be used to consolidate
the data representation.

For example, a ``join_arrays`` operation may accept inputs of array compatible
references from different source types to establish an "output" port with a
single type and shape.

Graph and node Schema
---------------------

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

Discussion
~~~~~~~~~~

Note that there may be some unnecessary information (bloat) between *operation*,
*namespace*, and *uid*. The *namespace* may already contain nesting information
using period delimiters, so the operation and namespace could be combined.

They were originally kept separate to allow for semantics by which an operation
could be implemented in multiple namespaces. Such semantics have not been
developed and are inherently problematic due to the implied coordination.
The scenario is obviated by more recent semantics, in which an operation can
declare its output in terms of a *type* meta-object resolvable through the API.

By convention, the *uid* contains a short indication of the operation being
performed, which is potentially redundant. To avoid redundancy, we could either
encode the namespace and operation in the element key, or remove it from the
element key. Note that the *uid* is currently provided directly by the operation
implementation, while the namespace and operation values are mediated by the
Context. Expression of the *uid* and/or element key should be moved to the
responsibility of the Context and/or Serializer, using helper functions from the
operation implementation under the fingerprinting behavior.

Further note that (as a value, but not as a key) the combined namespace and
operation could reasonably be represented as an array of strings, rather than
as an internally delimited string.

.. _simulation input:

Simulation input
~~~~~~~~~~~~~~~~

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

.. versionadded:: 0.0.0

    For records with *version: gmxapi_workspec_0_1*,
    operation instantiation is mediated during Session launch by the *depends*
    field of each element. The binding protocol is unspecified, but a dependent
    node builder is *subscribed* to the builder of the dependency before the
    builders are called in topologically valid order, as determined by the DAG
    implied by the *depends* network.

.. seealso::

   `DOI 10.1093/bioinformatics/bty484 <https://doi.org/10.1093/bioinformatics/bty484>`_

.. versionchanged:: 0.1

    For records with *version: gmxapi_graph_0_2*
    inputs, outputs, and other interfaces are explicitly represented in the
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

.. admonition:: Question

    What do we want to say about the topology due to outputs that are
    arrays? Generally, it is hard to know the size and shape of an array before the
    operation executes. Can topology be dynamic? Should we insist that array
    dimensionality must asserted when the node is created? Or are we simply not able
    to scatter from arrays that are operation outputs?

Logical Schematics
==================

Serialization
-------------

For the following reasons, *elements* are serialized as an associative *object*
instead of as a sequence, or *array* of *objects*.

1. A directed acyclic graph may have multiple topologically valid sequences.
2. Node records are arbitrarily large, and do not lend themselves to a dense array
   data type.
3. In-memory representations likely use associative data structures to allow
   node look-ups or node deletions.
4. Access to graph sections, while possibly benefitting from monotonicity optimizations,
   do not necessarily access contiguous members of a sequence.

The serialized document must contain a *version* and *elements* member.
Object sequence is unspecified.

.. uml::

    start

    :GraphSerializer;
    fork
        :version: str;
    fork again
        partition "foreach element" {
            fork
                :label: str;
            fork again
                :namespace: str;
            fork again
                :operation: str;
            fork again
                partition "foreach input" {
                    if (is reference) then (encode reference)
                        :reference: str;
                        :meta: mapping;
                    elseif (is collection) then (encode collection)
                        :label: mapping;
                    else (typed data)
                        :label: sequence;
                    endif
                }
                :inputs: mapping;
            fork again
                partition "foreach resource" {
                }
                :outputs and interfaces: mapping;
            end fork
        }
        :elements: mapping;
    end fork
    :SerializedRecordEncoder;



Deserialization
---------------

1. Produce native associative data structure from JSON encoded document.
2. Check *version* member for version string ``gmxapi_graph_0_2``.
3. Parse *elements* object.
4. Validate directed acyclic graph topology.
5. Instantiate concrete graph.

Graph parsing
-------------

For each member of *elements*:
2. Validate *namespace* and *operation*.
3. Resolve *input* references.
4. Use operation helpers (API) to validate input type and shape.
5. Use operation helpers (API) to validate advertised resources in terms of input.
6. Use operation helpers to validate node fingerprint.

Native graph management
-----------------------

To allow early error detection, API implementations should impose some usage
requirements.

All references in an element must resolve at the time it is added to the graph.

Once an element is added to the graph, it is immutable. Otherwise, we would need
to define update propagation behavior that may trigger multiple errors.
An allowable exception would be to permit elements to be removed from a graph
if and only if there are no dependent elements already in the graph.

Possible optimizations or hooks
===============================

JSON deserialization in Python
------------------------------

A more refined implementation in Python could heavily rely on the ``json`` module,
supplemented through the *object_hook* and *object_pairs_hook* to the
``json.JSONDecoder``. ``raw_decode()`` may facilitate dispatching decoding logic
within the document to save memory on temporary structures, but these have not
been investigated.

Note that it is non-trivial to deserialize JSON arrays directly to native arrays
for several reasons related to the flexibility of allowed array data in the JSON
document (most notably, the dimensionality).

Graph decoding
--------------

The associative structure of *element* *objects* produced by the JSON deserializer
does not have a guaranteed sequence.

Multi-step implementations likely fall into two categories.

.. rubric:: Deserialize, sequence, construct.

1. Deserialize the *elements* object to an associative structure.
2. Sequence the *elements*.
3. Initialize a DAG in a topologically valid sequence, such that the graph is
   always valid and nodes may be verified as they are added.

.. rubric:: Deserialize, stage, validate, construct.

1. Deserialize the *elements* object to an associative structure.
2. Stage the element records into a graph-aware data structure.
3. Validate that the structure contains a single connected directed acyclic graph.
4. Instantiate a native representation of the graph, with API validation.

For our reference implementation, we use the latter approach to leverage existing
tools, separate levels of input validation, and avoid the lure of premature
optimization. Though potentially inefficient for small graphs, the memory usage
and performance is predictable, there is minimal branching, and the only code
that is not Order(N) is the native hash algorithm for looking up *node* and *edge*
identifiers in the DAG or native graph representation.

Nearly Order(N) solutions are plausible if arbitrary parallelism and memory
usage are available to perform the sequencing, but would require additional
checks. E.g. a parallel event queue (when node instantiation events trigger the
completeness of a staged node's inputs, it may add an instantiation event),
but invalid (cyclic or incomplete) input would cause the event queue to stall.

Reference implementation
========================

..
    Note that the plantuml output can be retreived from the web server.

    Alternatively, use the ``.. uml::`` directive and add the following notes to the README:

        Assumes plantuml is installed and that a wrapper script exists at
        `/usr/local/bin/plantuml` as described at
        https://pypi.org/project/sphinxcontrib-plantuml/

        Then,

            pip install sphinxcontrib-plantuml
            sphinx-build -b html -c docs docs build/html
            open build/html/index.html

.. uml::

    WorkGraph -> WorkDeserializer: from_json()
    WorkDeserializer -> JSONDeserializer: <<utf-8>>
    WorkDeserializer <- JSONDeserializer
    WorkGraph <- WorkDeserializer

..
    Edit the source by pasting the image URL at http://www.plantuml.com/plantuml/

.. .. image:: http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuL80Wl3yecpteiI230LTEp379RKujIWpCIUpAhN8IY6jA3ytFgiuFqz34wGSGmL8brUmln-gBXkRqf8qNGixE-nwR7GnzA2w1QW2GnUNGsfU2j3L0000

..
    As a further alternative, the source is embedded in the generated SVG or can
    be retrieved from the URL with `-decodeurl` using the command line tool. For
    PNG output, there is the `-metadata` CLI option, but who wants PNG?
    Ref: http://plantuml.com/command-line
