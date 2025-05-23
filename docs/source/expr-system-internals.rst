Query planning with Expression system
=====================================

.. note::

    This document is intended for Dask developers and contributors. It is not
    intended for end-users.
    For a high level user guide, see :doc:`/dataframe-optimizer`.

.. toctree::
    :maxdepth: 2

The expression system was originally developed for Dask DataFrames, as implemented in the `dask-expr <https://github.com/dask/dask-expr>`_ project.

Expr objects
------------

The expression system is built around the `Expr` class. This class is used to represent a computation that can be performed on a Dask DataFrame. The `Expr` class is designed to be subclassed, and each subclass represents a specific type of computation. For example, there are subclasses for arithmetic operations, logical operations, and so on.

Construction
^^^^^^^^^^^^

The expression system centers around the Expr class, which represents a computation on a Dask DataFrame. This class is designed for subclassing; each subclass corresponds to a specific computation type (e.g., arithmetic, logical operations, filtering, joins).

Notably, custom initializers (``__init__``) are disallowed, both in the base class and its subclasses. This design decision reflects concerns around performance, as expression objects may be created and recreated frequently, and custom logic in constructors could introduce unnecessary overhead.

Instead, expression classes use a dataclass-like interface defined by two attributes:

* ``_parameters``: List of parameter names

* ``_defaults``: Dictionary of default values for optional parameters

Arguments passed to the constructor are stored in the operands attribute, with minimal input validation. Here's an example:

.. code-block:: python

    >>> class MyExpr(Expr):
            _parameters = ["param1", "param2"]
            _defaults = {"param2": None}

    >>> expr = MyExpr(1, 2, 3)
    >>> expr.param1
    1

    >>> expr.param2
    2

    >>> expr.operands
    [1, 2, 3]

Names and Tokens
^^^^^^^^^^^^^^^^

Every expression is uniquely identified by a name, composed of:

* A prefix (typically the class name or a variant)

* A token generated by hashing its operands via :func:`^dask.base.tokenize`

This tokenization enables:

* Deduplication of equivalent expressions in the graph

* Detection of changes across optimization steps

* Singleton enforcement for certain expression types

Expressions that subclass :class:`^dask.dataframe._expr.SingletonExpr` (the default for most DataFrame expressions) are guaranteed to be unique by name. However, this tokenization system introduces several challenges:

* Performance: Tokenization is slow due to recursive traversal and Dask’s dispatch mechanisms.

* Determinism: Objects without a registered __dask_tokenize__ fallback to (cloud)pickling, which can be slow and non-deterministic.

* Cross-interpreter behavior: Tokens are not consistent across interpreters or machines, complicating client-scheduler interactions.

To address this, each expression computes and caches its name and token upon construction. These values are stored and serialized to ensure pickle roundtrip stability. The token is accessible via ``_determ_token`` or ``deterministic_token``.

Caching and Singletons
^^^^^^^^^^^^^^^^^^^^^^

Despite efforts to keep expressions stateless, in practice many attributes are computed on demand and cached via ``functools.cached_property``. This defers computation but complicates reasoning about when and how state is evaluated.

Cached properties are typically serialized with the expression, unless ``_pickle_functools_cache`` is set to False.

To preserve cache values during repeated optimization (which recreates expressions), most classes subclass :class:`^dask.dataframe._expr.SingletonExpr`. This ensures that instances with the same name return a previously created, cached version. This makes expressions effectively immutable singletons — and must not be mutated in-place.

Optimization Procedure
----------------------

Expressions form a directed graph structure: when one expression is passed as an operand to another, it becomes a dependency. While this starts as a tree, deduplication by name quickly transforms it into a directed acyclic graph (DAG) — a key property for optimization.

The optimizer currently performs the following five steps:

* Simplify

* Rewrite / Tune

* Lower

* Simplify (again)

* Fuse

Simplify
^^^^^^^^

Simplification rewrites expressions into more optimal but semantically equivalent forms. A common example is pushing projections or filters down the graph to reduce computation earlier.

Constraints for simplification (not enforced at runtime):

* The number of partitions (npartitions) must not increase.

* No computations with side effects (e.g., computing divisions) may occur.

Rewrite / Tune
^^^^^^^^^^^^^^

This step implements performance tuning based on heuristics. Typically this targets a more efficient intermediate partitioning.

Two examples:

* Fuse I/O operations based on column projection (e.g., ``FusedIO``)

* Choose an appropriate ``split_out`` to balance partitioning

This step does not alter the logical meaning of expressions but adjusts execution-related parameters.

Lowering
^^^^^^^^

At this stage, abstract operations are transformed into concrete execution strategies.

For example:

A logical ``Merge`` might become a ``BlockwiseMerge`` if the input DataFrames are already partitioned appropriately.

In less favorable cases, a general ``HashJoinP2P`` might be selected instead.

This marks the transition from logical to physical plan, akin to traditional query planners.

After lowering, a second simplification step is applied to the resulting expressions.

Fuse
^^^^

Linear chains of blockwise tasks are combined into a single task, minimizing scheduler overhead.

Walking the Graph During Optimization
-------------------------------------

Each optimizer step traverses the expression graph until no further changes are found. The traversal typically follows this pattern (using Expr.simplify as an example):

#. Call ``simplify_once``, which:

#. Calls ``_simplify_down`` on self (the current expression). This downward pass:
    * Only has access to the current node and its operands
    * May return a new (optimized) expression or None

#. If a new expression is returned, check whether its name has changed (as unchanged names imply no effective change).

#. Then call ``_simplify_up`` on each dependency, passing in the parent node and a map of dependents. This upward pass:
    * Has access to context across branches (e.g., siblings, shared parents)
    * Returns a replacement for the parent expression

Finally, the traversal recurses into dependencies by calling ``simplify_once`` on each.

.. note:: Convergence and Memoization

  Without safeguards, this recursive traversal could loop indefinitely or cause exponential blowups. Protections include:

  * Memoization by expression name

  * Detection of repeated subgraphs

  Despite these, pathological cases occasionally arise (e.g. `dask-expr#835 <https://github.com/dask/dask-expr/issues/835>`_).

Expressions as the Client-Scheduler Interface
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Instead of transmitting low-level task graphs, we submit expressions directly to the scheduler. This reduces overhead but introduces complications:

* The distributed.Client requires final task keys before submission.

* Tokenization is non-deterministic across interpreters.

Optimization changes keys — so it must be run before submission to lock in key names and populate caches.

Some expressions (e.g., ReadParquet) require I/O to gather metadata like partition statistics. These steps must occur client-side, not on the scheduler, and are currently handled during lowering.

Legacy HighLevelGraph (HLG) Support
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
HighLevelGraph is a legacy representation still used by Dask Arrays, Bags, and Delayed objects. Despite its goal of deferring graph materialization, many code paths trigger unintended conversion to low-level graphs.

Key issues:

* Low-level optimization often forces premature materialization.

* HLG lacks knowledge of the collection type and required optimizations.

* HLG does not encode postcompute behavior (e.g., how to combine partitions).

To bridge this gap, the ``HLGExpr`` class wraps an HLG and implements the full expression interface. Materialization is delayed until the scheduler explicitly calls ``__dask_graph__``, at which point low-level optimization occurs. This ensures materialization and execution stay decoupled.


Custom Expressions and Collections
----------------------------------
