.. _creating-path-queries:

Creating path queries
#####################

You can create path queries to visualize the flow of information through a codebase.

Overview
========

Security researchers are particularly interested in the way that information flows in a program. Many vulnerabilities are caused by seemingly benign data flowing to unexpected locations, and being used in a malicious way.
Path queries written with CodeQL are particularly useful for analyzing data flow as they can be used to track the path taken by a variable from its possible starting points (``source``) to its possible end points (``sink``).
To model paths, your query must provide information about the ``source`` and the ``sink``, as well as the data flow steps that link them.

This topic provides information on how to structure a path query file so you can explore the paths associated with the results of data flow analysis.

.. pull-quote::

    Note

    The alerts generated by path queries are included in the results generated using the `CodeQL CLI <https://docs.github.com/en/code-security/codeql-cli>`__ and in `code scanning <https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/about-code-scanning-alerts#about-alert-details>`__. You can also view the path explanations generated by your path query in the :ref:`CodeQL extension for VS Code <codeql-for-visual-studio-code>`.


To learn more about modeling data flow with CodeQL, see ":doc:`About data flow analysis <about-data-flow-analysis>`."
For more language-specific information on analyzing data flow, see:

- ":ref:`Analyzing data flow in C/C++ <analyzing-data-flow-in-cpp>`"
- ":ref:`Analyzing data flow in C# <analyzing-data-flow-in-csharp>`"
- ":ref:`Analyzing data flow in Java <analyzing-data-flow-in-java>`"
- ":ref:`Analyzing data flow in JavaScript/TypeScript <analyzing-data-flow-in-javascript-and-typescript>`"
- ":ref:`Analyzing data flow in Python <analyzing-data-flow-in-python>`"
- ":ref:`Analyzing data flow in Ruby <analyzing-data-flow-in-ruby>`"
- ":ref:`Analyzing data flow in Swift <analyzing-data-flow-in-swift>`"

Path query examples
*******************

The easiest way to get started writing your own path query is to modify one of the existing queries. For more information, see the `CodeQL query help <https://codeql.github.com/codeql-query-help>`__.

The Security Lab researchers have used path queries to find security vulnerabilities in various open source projects. To see articles describing how these queries were written, as well as other posts describing other aspects of security research such as exploiting vulnerabilities, see the `GitHub Security Lab website <https://securitylab.github.com/research>`__.

Constructing a path query
=========================

Path queries require certain metadata, query predicates, and ``select`` statement structures.
Many of the built-in path queries included in CodeQL follow a simple structure, which depends on how the language you are analyzing is modeled with CodeQL.

You should use the following template:

.. code-block:: ql

    /**
     * ...
     * @kind path-problem
     * ...
     */

    import <language>
    // For some languages (Java/C++/Python/Swift) you need to explicitly import the data flow library, such as
    // import semmle.code.java.dataflow.DataFlow or import codeql.swift.dataflow.DataFlow
    ...

    module Flow = DataFlow::Global<MyConfiguration>;
    import Flow::PathGraph

    from Flow::PathNode source, Flow::PathNode sink
    where Flow::flowPath(source, sink)
    select sink.getNode(), source, sink, "<message>"

Where:

- ``MyConfiguration`` is a module containing the predicates that define how data may flow between the ``source`` and the ``sink``.
- ``Flow`` is the result of the data flow computation based on ``MyConfiguration``.
- ``Flow::Pathgraph`` is the resulting data flow graph module you need to import in order to include path explanations in the query.
- ``source`` and ``sink`` are nodes in the graph as defined in the configuration, and ``Flow::PathNode`` is their type.
- ``DataFlow::Global<..>`` is an invocation of data flow. ``TaintTracking::Global<..>`` can be used instead to include a default set of additional taint steps.


The following sections describe the main requirements for a valid path query.

Path query metadata
*******************

Path query metadata must contain the property ``@kind path-problem``–this ensures that query results are interpreted and displayed correctly.
The other metadata requirements depend on how you intend to run the query. For more information, see ":doc:`Metadata for CodeQL queries <metadata-for-codeql-queries>`."

Generating path explanations
****************************

In order to generate path explanations, your query needs to compute a graph.
To do this you need to define a :ref:`query predicate <query-predicates>` called ``edges`` in your query.
This predicate defines the edge relations of the graph you are computing, and it is used to compute the paths related to each result that your query generates.
You can import a predefined ``edges`` predicate from a path graph module in one of the standard data flow libraries. In addition to the path graph module, the data flow libraries contain the other ``classes``, ``predicates``, and ``modules`` that are commonly used in data flow analysis.

.. code-block:: ql

    import MyFlow::PathGraph

This statement imports the ``PathGraph`` module from the data flow library (``DataFlow.qll``), in which ``edges`` is defined.

You can also import libraries specifically designed to implement data flow analysis in various common frameworks and environments, and many additional libraries are included with CodeQL. To see examples of the different libraries used in data flow analysis, see the links to the built-in queries above or browse the `standard libraries <https://codeql.github.com/codeql-standard-libraries>`__.

For all languages, you can also optionally define a ``nodes`` query predicate, which specifies the nodes of the path graph that you are interested in. If ``nodes`` is defined, only edges with endpoints defined by these nodes are selected. If ``nodes`` is not defined, you select all possible endpoints of ``edges``.

Defining your own ``edges`` predicate
-------------------------------------

You can also define your own ``edges`` predicate in the body of your query. It should take the following form:

.. code-block:: ql

    query predicate edges(PathNode a, PathNode b) {
      /* Logical conditions which hold if `(a,b)` is an edge in the data flow graph */
    }

For more examples of how to define an ``edges`` predicate, visit the `standard CodeQL libraries <https://codeql.github.com/codeql-standard-libraries>`__ and search for ``edges``.

Declaring sources and sinks
***************************

You must provide information about the ``source`` and ``sink`` in your path query. These are objects that correspond to the nodes of the paths that you are exploring.
The name and the type of the ``source`` and the ``sink`` must be declared in the ``from`` statement of the query, and the types must be compatible with the nodes of the graph computed by the ``edges`` predicate.

If you are querying C/C++, C#, Go, Java, JavaScript, Python, or Ruby code (and you have used ``import MyFlow::PathGraph`` in your query), the definitions of the ``source`` and ``sink`` are accessed via the module resulting from the application of the ``Global<..>`` module in the data flow library. You should declare both of these objects in the ``from`` statement.
For example:

.. code-block:: ql

    module MyFlow = DataFlow::Global<MyConfiguration>;

    from MyFlow::PathNode source, MyFlow::PathNode sink

The configuration module must be defined to include definitions of sources and sinks. For example:

.. code-block:: ql

    module MyConfiguration implements DataFlow::ConfigSig {
      predicate isSource(DataFlow::Node source) { ... }
      predicate isSink(DataFlow::Node source) { ... }
    }

- ``isSource()`` defines where data may flow from.
- ``isSink()`` defines where data may flow to.

For more information on using the configuration class in your analysis see the sections on global data flow in ":ref:`Analyzing data flow in C/C++ <analyzing-data-flow-in-cpp>`," ":ref:`Analyzing data flow in C# <analyzing-data-flow-in-csharp>`," and ":ref:`Analyzing data flow in Python <analyzing-data-flow-in-python>`."

You can also create a configuration for different frameworks and environments by extending the ``Configuration`` class. For more information, see ":ref:`Types <defining-a-class>`" in the QL language reference.

Defining flow conditions
************************

The ``where`` clause defines the logical conditions to apply to the variables declared in the ``from`` clause to generate your results.
This clause can use :ref:`aggregations <aggregations>`, :ref:`predicates <predicates>`, and logical :ref:`formulas <formulas>` to limit the variables of interest to a smaller set which meet the defined conditions.

When writing a path queries, you would typically include a predicate that holds only if data flows from the ``source`` to the ``sink``.

You can use the ``flowPath`` predicate to specify flow from the ``source`` to the ``sink`` for a given ``Configuration``:

.. code-block:: ql

    where MyFlow::flowPath(source, sink)


Select clause
*************

Select clauses for path queries consist of four 'columns', with the following structure::

    select element, source, sink, string

The ``element`` and ``string`` columns represent the location of the alert and the alert message respectively, as explained in ":doc:`About CodeQL queries <about-codeql-queries>`." The second and third columns, ``source`` and ``sink``, are nodes on the path graph selected by the query.
Each result generated by your query is displayed at a single location in the same way as an alert query. Additionally, each result also has an associated path, which can be viewed in the :ref:`CodeQL extension for VS Code <codeql-for-visual-studio-code>`.

The ``element`` that you select in the first column depends on the purpose of the query and the type of issue that it is designed to find. This is particularly important for security issues. For example, if you believe the ``source`` value to be globally invalid or malicious it may be best to display the alert at the ``source``. In contrast, you should consider displaying the alert at the ``sink`` if you believe it is the element that requires sanitization.

The alert message defined in the final column in the ``select`` statement can be developed to give more detail about the alert or path found by the query using links and placeholders. For more information, see ":doc:`Defining the results of a query <defining-the-results-of-a-query>`."

Further reading
***************

- ":ref:`Exploring data flow with path queries <exploring-data-flow-with-path-queries>`"

- `CodeQL repository <https://github.com/github/codeql>`__
