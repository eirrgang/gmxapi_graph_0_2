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
               "interface": {"potential": "gromacs.restraint"}
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