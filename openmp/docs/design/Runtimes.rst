.. _openmp_runtimes:

LLVM/OpenMP Runtimes
====================

There are four distinct types of LLVM/OpenMP runtimes: the host runtime
:ref:`libomp`, the target offloading runtime :ref:`libomptarget`, the target
offloading plugin :ref:`libomptarget_plugin`, and finally the target device
runtime :ref:`libomptarget_device`.

For general information on debugging OpenMP target offloading applications, see
:ref:`libomptarget_info` and :ref:`libomptarget_device_debugging`

.. _libomp:

LLVM/OpenMP Host Runtime (``libomp``)
-------------------------------------

An `early (2015) design document
<https://raw.githubusercontent.com/llvm/llvm-project/main/openmp/runtime/doc/Reference.pdf>`_
for the LLVM/OpenMP host runtime, aka.  `libomp.so`, is available as a `pdf
<https://raw.githubusercontent.com/llvm/llvm-project/main/openmp/runtime/doc/Reference.pdf>`_.

.. _libomp_environment_vars:

Environment Variables
^^^^^^^^^^^^^^^^^^^^^

OMP_CANCELLATION
""""""""""""""""

Enables cancellation of the innermost enclosing region of the type specified.
If set to ``true``, the effects of the cancel construct and of cancellation
points are enabled and cancellation is activated. If set to ``false``,
cancellation is disabled and the cancel construct and cancellation points are
effectively ignored.

.. note::
   Internal barrier code will work differently depending on whether cancellation
   is enabled. Barrier code should repeatedly check the global flag to figure
   out if cancellation has been triggered. If a thread observes cancellation, it
   should leave the barrier prematurely with the return value 1 (and may wake up
   other threads). Otherwise, it should leave the barrier with the return value 0.

Enables (``true``) or disables (``false``) cancellation of the innermost
enclosing region of the type specified.

**Default:** ``false``


OMP_DISPLAY_ENV
"""""""""""""""

Enables (``true``) or disables (``false``) the printing to ``stderr`` of
the OpenMP version number and the values associated with the OpenMP
environment variables.

Possible values are: ``true``, ``false``, or ``verbose``.

**Default:** ``false``

OMP_DEFAULT_DEVICE
""""""""""""""""""

Sets the device that will be used in a target region. The OpenMP routine
``omp_set_default_device`` or a device clause in a parallel pragma can override
this variable. If no device with the specified device number exists, the code is
executed on the host. If this environment variable is not set, device number 0
is used.

OMP_DYNAMIC
"""""""""""

Enables (``true``) or disables (``false``) the dynamic adjustment of the
number of threads.

| **Default:** ``false``

OMP_MAX_ACTIVE_LEVELS
"""""""""""""""""""""

The maximum number of levels of parallel nesting for the program.

| **Default:** ``1``

OMP_NESTED
""""""""""

.. warning::
    Deprecated. Please use ``OMP_MAX_ACTIVE_LEVELS`` to control nested parallelism

Enables (``true``) or disables (``false``) nested parallelism.

| **Default:** ``false``

OMP_NUM_THREADS
"""""""""""""""

Sets the maximum number of threads to use for OpenMP parallel regions if no
other value is specified in the application.

The value can be a single integer, in which case it specifies the number of threads
for all parallel regions. The value can also be a comma-separated list of integers,
in which case each integer specifies the number of threads for a parallel
region at that particular nesting level.

The first position in the list represents the outer-most parallel nesting level,
the second position represents the next-inner parallel nesting level, and so on.
At any level, the integer can be left out of the list. If the first integer in a
list is left out, it implies the normal default value for threads is used at the
outer-most level. If the integer is left out of any other level, the number of
threads for that level is inherited from the previous level.

| **Default:** The number of processors visible to the operating system on which the program is executed.
| **Syntax:** ``OMP_NUM_THREADS=value[,value]*``
| **Example:** ``OMP_NUM_THREADS=4,3``

OMP_PLACES
""""""""""

Specifies an explicit ordered list of places, either as an abstract name
describing a set of places or as an explicit list of places described by
non-negative numbers. An exclusion operator, ``!``, can also be used to exclude
the number or place immediately following the operator.

For **explicit lists**, an ordered list of places is specified with each place
represented as a set of non-negative numbers. The non-negative numbers represent
operating system logical processor numbers and can be thought of as an OS affinity mask.

Individual places can be specified through two methods.
Both the **examples** below represent the same place.

* An explicit list of comma-separated non-negatives numbers **Example:** ``{0,2,4,6}``
* An interval with notation ``<lower-bound>:<length>[:<stride>]``.  **Example:** ``{0:4:2}``. When ``<stride>`` is omitted, a unit stride is assumed.
  The interval notation represents this set of numbers:

::

    <lower-bound>, <lower-bound> + <stride>, ..., <lower-bound> + (<length> - 1) * <stride>


A place list can also be specified using the same interval
notation: ``{place}:<length>[:<stride>]``.
This represents the list of length ``<length>`` places determined by the following:

.. code-block:: c

    {place}, {place} + <stride>, ..., {place} + (<length>-1)*<stride>
    Where given {place} and integer N, {place} + N = {place with every number offset by N}
    Example: {0,3,6}:4:1 represents {0,3,6}, {1,4,7}, {2,5,8}, {3,6,9}

**Examples of explicit lists:**
These all represent the same set of places

::

     OMP_PLACES="{0,1,2,3},{4,5,6,7},{8,9,10,11},{12,13,14,15}"
     OMP_PLACES="{0:4},{4:4},{8:4},{12:4}"
     OMP_PLACES="{0:4}:4:4"

.. note::
    When specifying a place using a set of numbers, if any number cannot be
    mapped to a processor on the target platform, then that number is
    ignored within the place, but the rest of the place is kept intact.
    If all numbers within a place are invalid, then the entire place is removed
    from the place list, but the rest of place list is kept intact.

The **abstract names** listed below are understood by the run-time environment:

* ``threads:`` Each place corresponds to a single hardware thread.
* ``cores:`` Each place corresponds to a single core (having one or more hardware threads).
* ``sockets:`` Each place corresponds to a single socket (consisting of one or more cores).
* ``numa_domains:`` Each place corresponds to a single NUMA domain (consisting of one or more cores).
* ``ll_caches:`` Each place corresponds to a last-level cache (consisting of one or more cores).

The abstract name may be appended by a positive number in parentheses to
denote the length of the place list to be created, that is ``abstract_name(num-places)``.
If the optional number isn't specified, then the runtime will use all available
resources of type ``abstract_name``. When requesting fewer places than available
on the system, the first available resources as determined by ``abstract_name``
are used. When requesting more places than available on the system, only the
available resources are used.

**Examples of abstract names:**
::

    OMP_PLACES=threads
    OMP_PLACES=threads(4)

OMP_PROC_BIND (Windows, Linux)
""""""""""""""""""""""""""""""
Sets the thread affinity policy to be used for parallel regions at the
corresponding nested level. Enables (``true``) or disables (``false``)
the binding of threads to processor contexts. If enabled, this is the
same as specifying ``KMP_AFFINITY=scatter``. If disabled, this is the
same as specifying ``KMP_AFFINITY=none``.

**Acceptable values:** ``true``, ``false``, or a comma separated list, each
element of which is one of the following values: ``master``, ``close``, ``spread``, or ``primary``.

**Default:** ``false``

.. warning::
    ``master`` is deprecated. The semantics of ``master`` are the same as ``primary``.

If set to ``false``, the execution environment may move OpenMP threads between
OpenMP places, thread affinity is disabled, and ``proc_bind`` clauses on
parallel constructs are ignored. Otherwise, the execution environment should
not move OpenMP threads between OpenMP places, thread affinity is enabled, and
the initial thread is bound to the first place in the OpenMP place list.

If set to ``primary``, all threads are bound to the same place as the primary
thread.

If set to ``close``, threads are bound to successive places, near where the
primary thread is bound.

If set to ``spread``, the primary thread's partition is subdivided and threads
are bound to single place successive sub-partitions.

| **Related environment variables:** ``KMP_AFFINITY`` (overrides ``OMP_PROC_BIND``).

OMP_SCHEDULE
""""""""""""
Sets the run-time schedule type and an optional chunk size.

| **Default:** ``static``, no chunk size specified
| **Syntax:** ``OMP_SCHEDULE="kind[,chunk_size]"``

OMP_STACKSIZE
"""""""""""""

Sets the number of bytes to allocate for each OpenMP thread to use as the
private stack for the thread. Recommended size is 16M.

Use the optional suffixes to specify byte units: ``B`` (bytes), ``K`` (Kilobytes),
``M`` (Megabytes), ``G`` (Gigabytes), or ``T`` (Terabytes) to specify the units.
If you specify a value without a suffix, the byte unit
is assumed to be ``K`` (Kilobytes).

This variable does not affect the native operating system threads created by the
user program, or the thread executing the sequential part of an OpenMP program.

The ``kmp_{set,get}_stacksize_s()`` routines set/retrieve the value.
The ``kmp_set_stacksize_s()`` routine must be called from sequential part, before
first parallel region is created. Otherwise, calling ``kmp_set_stacksize_s()``
has no effect.

| **Default:**

* 32-bit architecture: ``2M``
* 64-bit architecture: ``4M``

| **Related environment variables:** ``KMP_STACKSIZE`` (overrides ``OMP_STACKSIZE``).
| **Example:** ``OMP_STACKSIZE=8M``

OMP_THREAD_LIMIT
""""""""""""""""

Limits the number of simultaneously-executing threads in an OpenMP program.

If this limit is reached and another native operating system thread encounters
OpenMP API calls or constructs, the program can abort with an error message.
If this limit is reached when an OpenMP parallel region begins, a one-time
warning message might be generated indicating that the number of threads in
the team was reduced, but the program will continue.

The ``omp_get_thread_limit()`` routine returns the value of the limit.

| **Default:** No enforced limit
| **Related environment variable:** ``KMP_ALL_THREADS`` (overrides ``OMP_THREAD_LIMIT``).

OMP_WAIT_POLICY
"""""""""""""""

Decides whether threads spin (active) or yield (passive) while they are waiting.
``OMP_WAIT_POLICY=active`` is an alias for ``KMP_LIBRARY=turnaround``, and
``OMP_WAIT_POLICY=passive`` is an alias for ``KMP_LIBRARY=throughput``.

| **Default:** ``passive``

.. note::
    Although the default is ``passive``, unless the user has explicitly set
    ``OMP_WAIT_POLICY``, there is a small period of active spinning determined
    by ``KMP_BLOCKTIME``.

KMP_AFFINITY (Windows, Linux)
"""""""""""""""""""""""""""""

Enables run-time library to bind threads to physical processing units.

You must set this environment variable before the first parallel region, or
certain API calls including ``omp_get_max_threads()``, ``omp_get_num_procs()``
and any affinity API calls.

**Syntax:** ``KMP_AFFINITY=[<modifier>,...]<type>[,<permute>][,<offset>]``

``modifiers`` are optional strings consisting of a keyword and possibly a specifier

* ``respect`` (default) and ``norespect`` - determine whether to respect the original process affinity mask.
* ``verbose`` and ``noverbose`` (default) - determine whether to display affinity information.
* ``warnings`` (default) and ``nowarnings`` - determine whether to display warnings during affinity detection.
* ``reset`` and ``noreset`` (default) - determine whether to reset primary thread's affinity after outermost parallel region(s)
* ``granularity=<specifier>`` - takes the following specifiers ``thread``, ``core`` (default), ``tile``,
  ``socket``, ``die``, ``group`` (Windows only).
  The granularity describes the lowest topology levels that OpenMP threads are allowed to float within a topology map.
  For example, if ``granularity=core``, then the OpenMP threads will be allowed to move between logical processors within
  a single core. If ``granularity=thread``, then the OpenMP threads will be restricted to a single logical processor.
* ``proclist=[<proc_list>]`` - The ``proc_list`` is specified by

+--------------------+----------------------------------------+
| Value              |         Description                    |
+====================+========================================+
|   <proc_list> :=   |   <proc_id> | { <id_list> }            |
+--------------------+----------------------------------------+
|   <id_list> :=     |   <proc_id> | <proc_id>,<id_list>      |
+--------------------+----------------------------------------+

Where each ``proc_id`` represents an operating system logical processor ID.
For example, ``proclist=[3,0,{1,2},{0,3}]`` with ``OMP_NUM_THREADS=4`` would place thread 0 on
OS logical processor 3, thread 1 on OS logical processor 0, thread 2 on both OS logical
processors 1 & 2, and thread 3 on OS logical processors 0 & 3.

``type`` is the thread affinity policy to choose.
Valid choices are ``none``, ``balanced``, ``compact``, ``scatter``, ``explicit``, ``disabled``

* type ``none`` (default) - Does not bind OpenMP threads to particular thread contexts;
  however, if the operating system supports affinity, the compiler still uses the
  OpenMP thread affinity interface to determine machine topology.
  Specify ``KMP_AFFINITY=verbose,none`` to list a machine topology map.
* type ``compact`` - Specifying compact assigns the OpenMP thread <n>+1 to a free thread
  context as close as possible to the thread context where the <n> OpenMP thread was
  placed. For example, in a topology map, the nearer a node is to the root, the more
  significance the node has when sorting the threads.
* type ``scatter`` - Specifying scatter distributes the threads as evenly as
  possible across the entire system. ``scatter`` is the opposite of ``compact``; so the
  leaves of the node are most significant when sorting through the machine topology map.
* type ``balanced`` - Places threads on separate cores until all cores have at least one thread,
  similar to the ``scatter`` type. However, when the runtime must use multiple hardware thread
  contexts on the same core, the balanced type ensures that the OpenMP thread numbers are close
  to each other, which scatter does not do. This affinity type is supported on the CPU only for
  single socket systems.
* type ``explicit`` - Specifying explicit assigns OpenMP threads to a list of OS proc IDs that
  have been explicitly specified by using the ``proclist`` modifier, which is required
  for this affinity type.
* type ``disabled`` - Specifying disabled completely disables the thread affinity interfaces.
  This forces the OpenMP run-time library to behave as if the affinity interface was not
  supported by the operating system. This includes the low-level API interfaces such
  as ``kmp_set_affinity`` and ``kmp_get_affinity``, which have no effect and will return
  a nonzero error code.

For both ``compact`` and ``scatter``, ``permute`` and ``offset`` are allowed;
however, if you specify only one integer, the runtime interprets the value as
a permute specifier. **Both permute and offset default to 0.**

The ``permute`` specifier controls which levels are most significant when sorting
the machine topology map. A value for ``permute`` forces the mappings to make the
specified number of most significant levels of the sort the least significant,
and it inverts the order of significance. The root node of the tree is not
considered a separate level for the sort operations.

The ``offset`` specifier indicates the starting position for thread assignment.

| **Default:** ``noverbose,warnings,respect,granularity=core,none``
| **Related environment variable:** ``OMP_PROC_BIND`` (``KMP_AFFINITY`` takes precedence)

.. note::
    On Windows with multiple processor groups, the norespect affinity modifier
    is assumed when the process affinity mask equals a single processor group
    (which is default on Windows). Otherwise, the respect affinity modifier is used.

.. note::
    On Windows with multiple processor groups, if the granularity is too coarse, it
    will be set to ``granularity=group``. For example, if two processor groups exist
    across one socket, and ``granularity=socket`` the runtime will shift the
    granularity down to group since that is the largest granularity allowed by the OS.

KMP_HIDDEN_HELPER_AFFINITY (Windows, Linux)
"""""""""""""""""""""""""""""""""""""""""""

Enables run-time library to bind hidden helper threads to physical processing units.
This environment variable has the same syntax and semantics as ``KMP_AFFINIY`` but only
applies to the hidden helper team.

You must set this environment variable before the first parallel region, or
certain API calls including ``omp_get_max_threads()``, ``omp_get_num_procs()``
and any affinity API calls.

**Syntax:** Same as ``KMP_AFFINITY``

The following ``modifiers`` are ignored in ``KMP_HIDDEN_HELPER_AFFINITY`` and are only valid
for ``KMP_AFFINITY``:
* ``respect`` and ``norespect``
* ``reset`` and ``noreset``

KMP_ALL_THREADS
"""""""""""""""

Limits the number of simultaneously-executing threads in an OpenMP program.
If this limit is reached and another native operating system thread encounters
OpenMP API calls or constructs, then the program may abort with an error
message. If this limit is reached at the time an OpenMP parallel region begins,
a one-time warning message may be generated indicating that the number of
threads in the team was reduced, but the program will continue execution.

| **Default:** No enforced limit.
| **Related environment variable:** ``OMP_THREAD_LIMIT`` (``KMP_ALL_THREADS`` takes precedence)

KMP_BLOCKTIME
"""""""""""""

Sets the time that a thread should wait, after completing the
execution of a parallel region, before sleeping.

Use the optional suffixes: ``ms`` (milliseconds), or ``us`` (microseconds) to
specify/change the units. Defaults units is milliseconds.

Specify ``infinite`` for an unlimited wait time.

| **Default:** 200 milliseconds
| **Related Environment Variable:** ``KMP_LIBRARY``
| **Example:** ``KMP_BLOCKTIME=1ms``

KMP_CPUINFO_FILE
""""""""""""""""

Specifies an alternate file name for a file containing the machine topology
description. The file must be in the same format as :file:`/proc/cpuinfo`.

**Default:** None

KMP_DETERMINISTIC_REDUCTION
"""""""""""""""""""""""""""

Enables (``true``) or disables (``false``) the use of a specific ordering of
the reduction operations for implementing the reduction clause for an OpenMP
parallel region. This has the effect that, for a given number of threads, in
a given parallel region, for a given data set and reduction operation, a
floating point reduction done for an OpenMP reduction clause has a consistent
floating point result from run to run, since round-off errors are identical.

| **Default:** ``false``
| **Example:** ``KMP_DETERMINISTIC_REDUCTION=true``

KMP_DYNAMIC_MODE
""""""""""""""""

Selects the method used to determine the number of threads to use for a parallel
region when ``OMP_DYNAMIC=true``. Possible values: (``load_balance`` | ``thread_limit``), where,

* ``load_balance``: tries to avoid using more threads than available execution units on the machine;
* ``thread_limit``: tries to avoid using more threads than total execution units on the machine.

**Default:** ``load_balance`` (on all supported platforms)

KMP_HOT_TEAMS_MAX_LEVEL
"""""""""""""""""""""""
Sets the maximum nested level to which teams of threads will be hot.

.. note::
    A hot team is a team of threads optimized for faster reuse by subsequent
    parallel regions. In a hot team, threads are kept ready for execution of
    the next parallel region, in contrast to the cold team, which is freed
    after each parallel region, with its threads going into a common pool
    of threads.

For values of 2 and above, nested parallelism should be enabled.

**Default:** 1

KMP_HOT_TEAMS_MODE
""""""""""""""""""

Specifies the run-time behavior when the number of threads in a hot team is reduced.
Possible values:

* ``0`` - Extra threads are freed and put into a common pool of threads.
* ``1`` - Extra threads are kept in the team in reserve, for faster reuse
  in subsequent parallel regions.

**Default:** 0

KMP_HW_SUBSET
"""""""""""""

Specifies the subset of available hardware resources for the hardware topology
hierarchy. The subset is specified in terms of number of units per upper layer
unit starting from top layer downwards. E.g. the number of sockets (top layer
units), cores per socket, and the threads per core, to use with an OpenMP
application, as an alternative to writing complicated explicit affinity settings
or a limiting process affinity mask. You can also specify an offset value to set
which resources to use. When available, you can specify attributes to select
different subsets of resources.

An extended syntax is available when ``KMP_TOPOLOGY_METHOD=hwloc``. Depending on what
resources are detected, you may be able to specify additional resources, such as
NUMA domains and groups of hardware resources that share certain cache levels.

**Basic syntax:** ``[:][num_units|*]ID[@offset][:attribute] [,[num_units|*]ID[@offset][:attribute]...]``

An optional colon (:) can be specified at the beginning of the syntax to specify an explicit hardware subset. The default is an implicit hardware subset.

Supported unit IDs are not case-insensitive.

| ``S`` - socket
| ``num_units`` specifies the requested number of sockets.

| ``D`` - die
| ``num_units`` specifies the requested number of dies per socket.

| ``C`` - core
| ``num_units`` specifies the requested number of cores per die - if any - otherwise, per socket.

| ``T`` - thread
| ``num_units`` specifies the requested number of HW threads per core.

.. note::
    ``num_units`` can be left out or explicitly specified as ``*`` instead of a positive integer
    meaning use all specified resources at that level.
    e.g., ``1s,*c`` means use 1 socket and all the cores on that socket

``offset`` - (Optional) The number of units to skip.

``attribute`` - (Optional) An attribute differentiating resources at a particular level. The attributes available to users are:

* **Core type** - On Intel architectures, this can be ``intel_atom`` or ``intel_core``
* **Core efficiency** - This is specified as ``eff``:emphasis:`num` where :emphasis:`num` is a number from 0
  to the number of core efficiencies detected in the machine topology minus one.
  E.g., ``eff0``. The greater the efficiency number the more performant the core. There may be
  more core efficiencies than core types and can be viewed by setting ``KMP_AFFINITY=verbose``

.. note::
    The hardware cache can be specified as a unit, e.g. L2 for L2 cache,
    or LL for last level cache.

**Extended syntax when KMP_TOPOLOGY_METHOD=hwloc:**

Additional IDs can be specified if detected. For example:

``N`` - numa
``num_units`` specifies the requested number of NUMA nodes per upper layer
unit, e.g. per socket.

``TI`` - tile
num_units specifies the requested number of tiles to use per upper layer
unit, e.g. per NUMA node.

When any numa or tile units are specified in ``KMP_HW_SUBSET`` and the hwloc
topology method is available, the ``KMP_TOPOLOGY_METHOD`` will be automatically
set to hwloc, so there is no need to set it explicitly.

For an **explicit hardware subset**, if one or more topology layers detected by the
runtime are omitted from the subset, then those topology layers are ignored.
Only explicitly specified topology layers are used in the subset.

For an **implicit hardware subset**, it is implied that the socket, core, and thread
topology types should be included in the subset. Other topology layers are not
implicitly included and are ignored if they are not specified in the subset.
Because the socket, core and thread topology types are always included in
implicit hardware subsets, when they are omitted, it is assumed that all
available resources of that type should be used. Implicit hardware subsets are
the default.

If you don't specify one or more types of resource, such as socket or thread,
all available resources of that type are used.

The run-time library prints a warning, and the setting of
``KMP_HW_SUBSET`` is ignored if:

* a resource is specified, but detection of that resource is not supported
  by the chosen topology detection method and/or
* a resource is specified twice. An exception to this condition is if attributes
  differentiate the resource.
* attributes are used when not detected in the machine topology or conflict with
  each other.

This variable does not work if ``KMP_AFFINITY=disabled``.

**Default:** If omitted, the default value is to use all the
available hardware resources.

**Implicit Hardware Subset Examples:**

* ``2s,4c,2t``: Use the first 2 sockets (s0 and s1), the first 4 cores on each
  socket (c0 - c3), and 2 threads per core.
* ``2s@2,4c@8,2t``: Skip the first 2 sockets (s0 and s1) and use 2 sockets
  (s2-s3), skip the first 8 cores (c0-c7) and use 4 cores on each socket
  (c8-c11), and use 2 threads per core.
* ``5C@1,3T``: Use all available sockets, skip the first core and use 5 cores,
  and use 3 threads per core.
* ``1T``: Use all cores on all sockets, 1 thread per core.
* ``1s, 1d, 1n, 1c, 1t``: Use 1 socket, 1 die, 1 NUMA node, 1 core, 1 thread
  - use HW thread as a result.
* ``4c:intel_atom,5c:intel_core``: Use all available sockets and use 4
  Intel Atom(R) processor cores and 5 Intel(R) Core(TM) processor cores per socket.
* ``2c:eff0@1,3c:eff1``: Use all available sockets, skip the first core with efficiency 0
  and use the next 2 cores with efficiency 0 and 3 cores with efficiency 1 per socket.
* ``1s, 1c, 1t``: Use 1 socket, 1 core, 1 thread. This may result in using
  single thread on a 3-layer topology architecture, or multiple threads on
  4-layer or 5-layer architecture. Result may even be different on the same
  architecture, depending on ``KMP_TOPOLOGY_METHOD`` specified, as hwloc can
  often detect more topology layers than the default method used by the OpenMP
  run-time library.
* ``*c:eff1@3``: Use all available sockets, skip the first three cores of
  efficiency 1, and then use the rest of the available cores of efficiency 1.

Explicit Hardware Subset Examples:

* ``:2s,6t`` Use exactly the first two sockets and 6 threads per socket.
* ``:1t@7`` Skip the first 7 threads (t0-t6) and use exactly one thread (t7).
* ``:5c,1t`` Use exactly the first 5 cores (c0-c4) and the first thread on each core.

To see the result of the setting, you can specify ``verbose`` modifier in
``KMP_AFFINITY`` environment variable. The OpenMP run-time library will output
to ``stderr`` the information about the discovered hardware topology before and
after the ``KMP_HW_SUBSET`` setting was applied.

KMP_INHERIT_FP_CONTROL
""""""""""""""""""""""

Enables (``true``) or disables (``false``) the copying of the floating-point
control settings of the primary thread to the floating-point control settings
of the OpenMP worker threads at the start of each parallel region.

**Default:** ``true``

KMP_LIBRARY
"""""""""""

Selects the OpenMP run-time library execution mode. The values for this variable
are ``serial``, ``turnaround``, or ``throughput``.

| **Default:** ``throughput``
| **Related environment variable:** ``KMP_BLOCKTIME`` and ``OMP_WAIT_POLICY``

KMP_SETTINGS
""""""""""""

Enables (``true``) or disables (``false``) the printing of OpenMP run-time library
environment variables during program execution. Two lists of variables are printed:
user-defined environment variables settings and effective values of variables used
by OpenMP run-time library.

**Default:** ``false``

KMP_STACKSIZE
"""""""""""""

Sets the number of bytes to allocate for each OpenMP thread to use as its private stack.

Recommended size is ``16M``.

Use the optional suffixes to specify byte units: ``B`` (bytes), ``K`` (Kilobytes),
``M`` (Megabytes), ``G`` (Gigabytes), or ``T`` (Terabytes) to specify the units.
If you specify a value without a suffix, the byte unit is assumed to be K (Kilobytes).

**Related environment variable:** ``KMP_STACKSIZE`` overrides ``GOMP_STACKSIZE``, which
overrides ``OMP_STACKSIZE``.

**Default:**

* 32-bit architectures: ``2M``
* 64-bit architectures: ``4M``

KMP_TOPOLOGY_METHOD
"""""""""""""""""""

Forces OpenMP to use a particular machine topology modeling method.

Possible values are:

* ``all`` - Let OpenMP choose which topology method is most appropriate
  based on the platform and possibly other environment variable settings.
* ``cpuid_leaf31`` (x86 only) - Decodes the APIC identifiers as specified by leaf 31 of the
  cpuid instruction. The runtime will produce an error if the machine does not support leaf 31.
* ``cpuid_leaf11`` (x86 only) - Decodes the APIC identifiers as specified by leaf 11 of the
  cpuid instruction. The runtime will produce an error if the machine does not support leaf 11.
* ``cpuid_leaf4`` (x86 only) - Decodes the APIC identifiers as specified in leaf 4
  of the cpuid instruction. The runtime will produce an error if the machine does not support leaf 4.
* ``cpuinfo`` - If ``KMP_CPUINFO_FILE`` is not specified, forces OpenMP to
  parse :file:`/proc/cpuinfo` to determine the topology (Linux only).
  If ``KMP_CPUINFO_FILE`` is specified as described above, uses it (Windows or Linux).
* ``group`` - Models the machine as a 2-level map, with level 0 specifying the
  different processors in a group, and level 1 specifying the different
  groups (Windows 64-bit only).

.. note::
    Support for group is now deprecated and will be removed in a future release. Use all instead.

* ``flat`` - Models the machine as a flat (linear) list of processors.
* ``hwloc`` - Models the machine as the Portable Hardware Locality (hwloc) library does.
  This model is the most detailed and includes, but is not limited to: numa domains,
  packages, cores, hardware threads, caches, and Windows processor groups. This method is
  only available if you have configured libomp to use hwloc during CMake configuration.

**Default:** all

KMP_VERSION
"""""""""""

Enables (``true``) or disables (``false``) the printing of OpenMP run-time
library version information during program execution.

**Default:** ``false``

KMP_WARNINGS
""""""""""""

Enables (``true``) or disables (``false``) displaying warnings from the
OpenMP run-time library during program execution.

**Default:** ``true``

.. _libomptarget:

LLVM/OpenMP Target Host Runtime (``libomptarget``)
--------------------------------------------------

.. _libopenmptarget_environment_vars:

Environment Variables
^^^^^^^^^^^^^^^^^^^^^

``libomptarget`` uses environment variables to control different features of the
library at runtime. This allows the user to obtain useful runtime information as
well as enable or disable certain features. A full list of supported environment
variables is defined below.

    * ``LIBOMPTARGET_DEBUG=<Num>``
    * ``LIBOMPTARGET_PROFILE=<Filename>``
    * ``LIBOMPTARGET_PROFILE_GRANULARITY=<Num> (default 500, in us)``
    * ``LIBOMPTARGET_MEMORY_MANAGER_THRESHOLD=<Num>``
    * ``LIBOMPTARGET_INFO=<Num>``
    * ``LIBOMPTARGET_HEAP_SIZE=<Num>``
    * ``LIBOMPTARGET_STACK_SIZE=<Num>``
    * ``LIBOMPTARGET_SHARED_MEMORY_SIZE=<Num>``
    * ``LIBOMPTARGET_MAP_FORCE_ATOMIC=[TRUE/FALSE] (default TRUE)``
    * ``LIBOMPTARGET_JIT_OPT_LEVEL={0,1,2,3} (default 3)``
    * ``LIBOMPTARGET_JIT_SKIP_OPT=[TRUE/FALSE] (default FALSE)``
    * ``LIBOMPTARGET_JIT_REPLACEMENT_OBJECT=<in:Filename> (object file)``
    * ``LIBOMPTARGET_JIT_REPLACEMENT_MODULE=<in:Filename> (LLVM-IR file)``
    * ``LIBOMPTARGET_JIT_PRE_OPT_IR_MODULE=<out:Filename> (LLVM-IR file)``
    * ``LIBOMPTARGET_JIT_POST_OPT_IR_MODULE=<out:Filename> (LLVM-IR file)``
    * ``LIBOMPTARGET_MIN_THREADS_FOR_LOW_TRIP_COUNT=<Num> (default: 32)``
    * ``LIBOMPTARGET_REUSE_BLOCKS_FOR_HIGH_TRIP_COUNT=[TRUE/FALSE] (default TRUE)``
    * ``OFFLOAD_TRACK_ALLOCATION_TRACES=[TRUE/FALSE] (default FALSE)``
    * ``OFFLOAD_TRACK_NUM_KERNEL_LAUNCH_TRACES=<Num> (default 0)``

LIBOMPTARGET_DEBUG
""""""""""""""""""

``LIBOMPTARGET_DEBUG`` controls whether or not debugging information will be
displayed. This feature is only available if ``libomptarget`` was built with
``-DOMPTARGET_DEBUG``. The debugging output provided is intended for use by
``libomptarget`` developers. More user-friendly output is presented when using
``LIBOMPTARGET_INFO``.

LIBOMPTARGET_PROFILE
""""""""""""""""""""

``LIBOMPTARGET_PROFILE`` allows ``libomptarget`` to generate time profile output
similar to Clang's ``-ftime-trace`` option. This generates a JSON file based on
`Chrome Tracing`_ that can be viewed with ``chrome://tracing`` or the
`Speedscope App`_. The output will be saved to the filename specified by the
environment variable. For multi-threaded applications, profiling in ``libomp``
is also needed. Setting the CMake option ``OPENMP_ENABLE_LIBOMP_PROFILING=ON``
to enable the feature. This feature depends on the `LLVM Support Library`_
for time trace output. Note that this will turn ``libomp`` into a C++ library.

.. _`Chrome Tracing`: https://www.chromium.org/developers/how-tos/trace-event-profiling-tool

.. _`Speedscope App`: https://www.speedscope.app/

.. _`LLVM Support Library`: https://llvm.org/docs/SupportLibrary.html

LIBOMPTARGET_PROFILE_GRANULARITY
""""""""""""""""""""""""""""""""

``LIBOMPTARGET_PROFILE_GRANULARITY`` allows to change the time profile
granularity measured in `us`. Default is 500 (`us`).

LIBOMPTARGET_MEMORY_MANAGER_THRESHOLD
"""""""""""""""""""""""""""""""""""""

``LIBOMPTARGET_MEMORY_MANAGER_THRESHOLD`` sets the threshold size for which the
``libomptarget`` memory manager will handle the allocation. Any allocations
larger than this threshold will not use the memory manager and be freed after
the device kernel exits. The default threshold value is ``8KB``. If
``LIBOMPTARGET_MEMORY_MANAGER_THRESHOLD`` is set to ``0`` the memory manager
will be completely disabled.

.. _libomptarget_info:

LIBOMPTARGET_INFO
"""""""""""""""""

``LIBOMPTARGET_INFO`` allows the user to request different types of runtime
information from ``libomptarget``. ``LIBOMPTARGET_INFO`` uses a 32-bit field to
enable or disable different types of information. This includes information
about data-mappings and kernel execution. It is recommended to build your
application with debugging information enabled, this will enable filenames and
variable declarations in the information messages. OpenMP Debugging information
is enabled at any level of debugging so a full debug runtime is not required.
For minimal debugging information compile with `-gline-tables-only`, or compile
with `-g` for full debug information. A full list of flags supported by
``LIBOMPTARGET_INFO`` is given below.

    * Print all data arguments upon entering an OpenMP device kernel: ``0x01``
    * Indicate when a mapped address already exists in the device mapping table:
      ``0x02``
    * Dump the contents of the device pointer map at kernel exit: ``0x04``
    * Indicate when an entry is changed in the device mapping table: ``0x08``
    * Print OpenMP kernel information from device plugins: ``0x10``
    * Indicate when data is copied to and from the device: ``0x20``

Any combination of these flags can be used by setting the appropriate bits. For
example, to enable printing all data active in an OpenMP target region along
with ``CUDA`` information, run the following ``bash`` command.

.. code-block:: console

   $ env LIBOMPTARGET_INFO=$((0x1 | 0x10)) ./your-application

Or, to enable every flag run with every bit set.

.. code-block:: console

   $ env LIBOMPTARGET_INFO=-1 ./your-application

For example, given a small application implementing the ``ZAXPY`` BLAS routine,
``Libomptarget`` can provide useful information about data mappings and thread
usages.

.. code-block:: c++

    #include <complex>

    using complex = std::complex<double>;

    void zaxpy(complex *X, complex *Y, complex D, std::size_t N) {
    #pragma omp target teams distribute parallel for
      for (std::size_t i = 0; i < N; ++i)
        Y[i] = D * X[i] + Y[i];
    }

    int main() {
      const std::size_t N = 1024;
      complex X[N], Y[N], D;
    #pragma omp target data map(to:X[0 : N]) map(tofrom:Y[0 : N])
      zaxpy(X, Y, D, N);
    }

Compiling this code targeting ``nvptx64`` with all information enabled will
provide the following output from the runtime library.

.. code-block:: console

    $ clang++ -fopenmp -fopenmp-targets=nvptx64 -O3 -gline-tables-only zaxpy.cpp -o zaxpy
    $ env LIBOMPTARGET_INFO=-1 ./zaxpy

.. code-block:: text

    Info: Entering OpenMP data region at zaxpy.cpp:14:1 with 2 arguments:
    Info: to(X[0:N])[16384]
    Info: tofrom(Y[0:N])[16384]
    Info: Creating new map entry with HstPtrBegin=0x00007fff0d259a40,
          TgtPtrBegin=0x00007fdba5800000, Size=16384, RefCount=1, Name=X[0:N]
    Info: Copying data from host to device, HstPtr=0x00007fff0d259a40,
          TgtPtr=0x00007fdba5800000, Size=16384, Name=X[0:N]
    Info: Creating new map entry with HstPtrBegin=0x00007fff0d255a40,
          TgtPtrBegin=0x00007fdba5804000, Size=16384, RefCount=1, Name=Y[0:N]
    Info: Copying data from host to device, HstPtr=0x00007fff0d255a40,
          TgtPtr=0x00007fdba5804000, Size=16384, Name=Y[0:N]
    Info: OpenMP Host-Device pointer mappings after block at zaxpy.cpp:14:1:
    Info: Host Ptr           Target Ptr         Size (B) RefCount Declaration
    Info: 0x00007fff0d255a40 0x00007fdba5804000 16384    1        Y[0:N] at zaxpy.cpp:13:17
    Info: 0x00007fff0d259a40 0x00007fdba5800000 16384    1        X[0:N] at zaxpy.cpp:13:11
    Info: Entering OpenMP kernel at zaxpy.cpp:6:1 with 4 arguments:
    Info: firstprivate(N)[8] (implicit)
    Info: use_address(Y)[0] (implicit)
    Info: tofrom(D)[16] (implicit)
    Info: use_address(X)[0] (implicit)
    Info: Mapping exists (implicit) with HstPtrBegin=0x00007fff0d255a40,
          TgtPtrBegin=0x00007fdba5804000, Size=0, RefCount=2 (incremented), Name=Y
    Info: Creating new map entry with HstPtrBegin=0x00007fff0d2559f0,
          TgtPtrBegin=0x00007fdba5808000, Size=16, RefCount=1, Name=D
    Info: Copying data from host to device, HstPtr=0x00007fff0d2559f0,
          TgtPtr=0x00007fdba5808000, Size=16, Name=D
    Info: Mapping exists (implicit) with HstPtrBegin=0x00007fff0d259a40,
          TgtPtrBegin=0x00007fdba5800000, Size=0, RefCount=2 (incremented), Name=X
    Info: Mapping exists with HstPtrBegin=0x00007fff0d255a40,
          TgtPtrBegin=0x00007fdba5804000, Size=0, RefCount=2 (update suppressed)
    Info: Mapping exists with HstPtrBegin=0x00007fff0d2559f0,
          TgtPtrBegin=0x00007fdba5808000, Size=16, RefCount=1 (update suppressed)
    Info: Mapping exists with HstPtrBegin=0x00007fff0d259a40,
          TgtPtrBegin=0x00007fdba5800000, Size=0, RefCount=2 (update suppressed)
    Info: Launching kernel __omp_offloading_10305_c08c86__Z5zaxpyPSt7complexIdES1_S0_m_l6
          with 8 blocks and 128 threads in SPMD mode
    Info: Mapping exists with HstPtrBegin=0x00007fff0d259a40,
          TgtPtrBegin=0x00007fdba5800000, Size=0, RefCount=1 (decremented)
    Info: Mapping exists with HstPtrBegin=0x00007fff0d2559f0,
          TgtPtrBegin=0x00007fdba5808000, Size=16, RefCount=1 (deferred final decrement)
    Info: Copying data from device to host, TgtPtr=0x00007fdba5808000,
          HstPtr=0x00007fff0d2559f0, Size=16, Name=D
    Info: Mapping exists with HstPtrBegin=0x00007fff0d255a40,
          TgtPtrBegin=0x00007fdba5804000, Size=0, RefCount=1 (decremented)
    Info: Removing map entry with HstPtrBegin=0x00007fff0d2559f0,
          TgtPtrBegin=0x00007fdba5808000, Size=16, Name=D
    Info: OpenMP Host-Device pointer mappings after block at zaxpy.cpp:6:1:
    Info: Host Ptr           Target Ptr         Size (B) RefCount Declaration
    Info: 0x00007fff0d255a40 0x00007fdba5804000 16384    1        Y[0:N] at zaxpy.cpp:13:17
    Info: 0x00007fff0d259a40 0x00007fdba5800000 16384    1        X[0:N] at zaxpy.cpp:13:11
    Info: Exiting OpenMP data region at zaxpy.cpp:14:1 with 2 arguments:
    Info: to(X[0:N])[16384]
    Info: tofrom(Y[0:N])[16384]
    Info: Mapping exists with HstPtrBegin=0x00007fff0d255a40,
          TgtPtrBegin=0x00007fdba5804000, Size=16384, RefCount=1 (deferred final decrement)
    Info: Copying data from device to host, TgtPtr=0x00007fdba5804000,
          HstPtr=0x00007fff0d255a40, Size=16384, Name=Y[0:N]
    Info: Mapping exists with HstPtrBegin=0x00007fff0d259a40,
          TgtPtrBegin=0x00007fdba5800000, Size=16384, RefCount=1 (deferred final decrement)
    Info: Removing map entry with HstPtrBegin=0x00007fff0d255a40,
          TgtPtrBegin=0x00007fdba5804000, Size=16384, Name=Y[0:N]
    Info: Removing map entry with HstPtrBegin=0x00007fff0d259a40,
          TgtPtrBegin=0x00007fdba5800000, Size=16384, Name=X[0:N]

From this information, we can see the OpenMP kernel being launched on the CUDA
device with enough threads and blocks for all ``1024`` iterations of the loop in
simplified :doc:`SPMD Mode <Offloading>`. The information from the OpenMP data
region shows the two arrays ``X`` and ``Y`` being copied from the host to the
device. This creates an entry in the host-device mapping table associating the
host pointers to the newly created device data. The data mappings in the OpenMP
device kernel show the default mappings being used for all the variables used
implicitly on the device. Because ``X`` and ``Y`` are already mapped in the
device's table, no new entries are created. Additionally, the default mapping
shows that ``D`` will be copied back from the device once the OpenMP device
kernel region ends even though it isn't written to. Finally, at the end of the
OpenMP data region the entries for ``X`` and ``Y`` are removed from the table.

The information level can be controlled at runtime using an internal
libomptarget library call ``__tgt_set_info_flag``. This allows for different
levels of information to be enabled or disabled for certain regions of code.
Using this requires declaring the function signature as an external function so
it can be linked with the runtime library.

.. code-block:: c++

    extern "C" void __tgt_set_info_flag(uint32_t);

    extern foo();

    int main() {
      __tgt_set_info_flag(0x10);
    #pragma omp target
      foo();
    }

.. _libopenmptarget_errors:

Errors:
^^^^^^^

``libomptarget`` provides error messages when the program fails inside the
OpenMP target region. Common causes of failure could be an invalid pointer
access, running out of device memory, or trying to offload when the device is
busy. If the application was built with debugging symbols the error messages
will additionally provide the source location of the OpenMP target region.

For example, consider the following code that implements a simple parallel
reduction on the GPU. This code has a bug that causes it to fail in the
offloading region.

.. code-block:: c++

    #include <cstdio>

    double sum(double *A, std::size_t N) {
      double sum = 0.0;
    #pragma omp target teams distribute parallel for reduction(+:sum)
      for (int i = 0; i < N; ++i)
        sum += A[i];

      return sum;
    }

    int main() {
      const int N = 1024;
      double A[N];
      sum(A, N);
    }

If this code is compiled and run, there will be an error message indicating what is
going wrong.

.. code-block:: console

    $ clang++ -fopenmp -fopenmp-targets=nvptx64 -O3 -gline-tables-only sum.cpp -o sum
    $ ./sum

.. code-block:: text

    CUDA error: an illegal memory access was encountered
    Libomptarget error: Copying data from device failed.
    Libomptarget error: Call to targetDataEnd failed, abort target.
    Libomptarget error: Failed to process data after launching the kernel.
    Libomptarget error: Consult https://openmp.llvm.org/design/Runtimes.html for debugging options.
    sum.cpp:5:1: Libomptarget error 1: failure of target construct while offloading is mandatory

This shows that there is an illegal memory access occurring inside the OpenMP
target region once execution has moved to the CUDA device, suggesting a
segmentation fault. This then causes a chain reaction of failures in
``libomptarget``. Another message suggests using the ``LIBOMPTARGET_INFO``
environment variable as described in :ref:`libopenmptarget_environment_vars`. If
we do this it will print the sate of the host-target pointer mappings at the
time of failure.

.. code-block:: console

    $ clang++ -fopenmp -fopenmp-targets=nvptx64 -O3 -gline-tables-only sum.cpp -o sum
    $ env LIBOMPTARGET_INFO=4 ./sum

.. code-block:: text

    info: OpenMP Host-Device pointer mappings after block at sum.cpp:5:1:
    info: Host Ptr           Target Ptr         Size (B) RefCount Declaration
    info: 0x00007ffc058280f8 0x00007f4186600000 8        1        sum at sum.cpp:4:10

This tells us that the only data mapped between the host and the device is the
``sum`` variable that will be copied back from the device once the reduction has
ended. There is no entry mapping the host array ``A`` to the device. In this
situation, the compiler cannot determine the size of the array at compile time
so it will simply assume that the pointer is mapped on the device already by
default. The solution is to add an explicit map clause in the target region.

.. code-block:: c++

    double sum(double *A, std::size_t N) {
      double sum = 0.0;
    #pragma omp target teams distribute parallel for reduction(+:sum) map(to:A[0 : N])
      for (int i = 0; i < N; ++i)
        sum += A[i];

      return sum;
    }

LIBOMPTARGET_STACK_SIZE
"""""""""""""""""""""""

This environment variable sets the stack size in bytes for the AMDGPU and CUDA
plugins. This can be used to increase or decrease the standard amount of memory
reserved for each thread's stack.

LIBOMPTARGET_HEAP_SIZE
"""""""""""""""""""""""

This environment variable sets the amount of memory in bytes that can be
allocated using ``malloc`` and ``free`` for the CUDA plugin. This is necessary
for some applications that allocate too much memory either through the user or
globalization.

LIBOMPTARGET_SHARED_MEMORY_SIZE
"""""""""""""""""""""""""""""""

This environment variable sets the amount of dynamic shared memory in bytes used
by the kernel once it is launched. A pointer to the dynamic memory buffer can be
accessed using the ``llvm_omp_target_dynamic_shared_alloc`` function. An example
is shown in :ref:`libomptarget_dynamic_shared`.

.. toctree::
   :hidden:
   :maxdepth: 1

   Offloading


LIBOMPTARGET_MAP_FORCE_ATOMIC
"""""""""""""""""""""""""""""

The OpenMP standard guarantees that map clauses are atomic. However, the this
can have a drastic performance impact. Users that do not require atomic map
clauses can disable them to potentially recover lost performance. As a
consequence, users have to guarantee themselves that no two map clauses will
concurrently map the same memory. If the memory is already mapped and the
map clauses will only modify the reference counter from a non-zero count to
another non-zero count, concurrent map clauses are supported regardless of
this option. To disable forced atomic map clauses use "false"/"FALSE" as the
value of the ``LIBOMPTARGET_MAP_FORCE_ATOMIC`` environment variable.
The default behavior of LLVM 14 is to force atomic maps clauses, prior versions
of LLVM did not.

.. _libomptarget_jit_opt_level:

LIBOMPTARGET_JIT_OPT_LEVEL
""""""""""""""""""""""""""

This environment variable can be used to change the optimization pipeline used
to optimize the embedded device code as part of the device JIT. The value is
corresponds to the ``-O{0,1,2,3}`` command line argument passed to ``clang``.

LIBOMPTARGET_JIT_SKIP_OPT
""""""""""""""""""""""""""

This environment variable can be used to skip the optimization pipeline during
JIT compilation. If set, the image will only be passed through the backend. The
backend is invoked with the ``LIBOMPTARGET_JIT_OPT_LEVEL`` flag.

LIBOMPTARGET_JIT_REPLACEMENT_OBJECT
"""""""""""""""""""""""""""""""""""

This environment variable can be used to replace the embedded device code
before the device JIT finishes compilation for the target. The value is
expected to be a filename to an object file, thus containing the output of the
assembler in object format for the respective target. The JIT optimization
pipeline and backend are skipped and only target specific post-processing is
performed on the object file before it is loaded onto the device.

.. _libomptarget_jit_replacement_module:

LIBOMPTARGET_JIT_REPLACEMENT_MODULE
"""""""""""""""""""""""""""""""""""

This environment variable can be used to replace the embedded device code
before the device JIT finishes compilation for the target. The value is
expected to be a filename to an LLVM-IR file, thus containing an LLVM-IR module
for the respective target. To obtain a device code image compatible with the
embedded one it is recommended to extract the embedded one either before or
after IR optimization. This can be done at compile time, after compile time via
llvm tools (llvm-objdump), or, simply, by setting the
:ref:`LIBOMPTARGET_JIT_PRE_OPT_IR_MODULE` or
:ref:`LIBOMPTARGET_JIT_POST_OPT_IR_MODULE` environment variables.

.. _libomptarget_jit_pre_opt_ir_module:

LIBOMPTARGET_JIT_PRE_OPT_IR_MODULE
""""""""""""""""""""""""""""""""""

This environment variable can be used to extract the embedded device code
before the device JIT runs additional IR optimizations on it (see
:ref:`LIBOMPTARGET_JIT_OPT_LEVEL`). The value is expected to be a filename into
which the LLVM-IR module is written. The module can be the analyzed, and
transformed and loaded back into the JIT pipeline via
:ref:`LIBOMPTARGET_JIT_REPLACEMENT_MODULE`.

.. _libomptarget_jit_post_opt_ir_module:

LIBOMPTARGET_JIT_POST_OPT_IR_MODULE
"""""""""""""""""""""""""""""""""""

This environment variable can be used to extract the embedded device code after
the device JIT runs additional IR optimizations on it (see
:ref:`LIBOMPTARGET_JIT_OPT_LEVEL`). The value is expected to be a filename into
which the LLVM-IR module is written. The module can be the analyzed, and
transformed and loaded back into the JIT pipeline via
:ref:`LIBOMPTARGET_JIT_REPLACEMENT_MODULE`.


LIBOMPTARGET_MIN_THREADS_FOR_LOW_TRIP_COUNT
"""""""""""""""""""""""""""""""""""""""""""

This environment variable defines a lower bound for the number of threads if a
combined kernel, e.g., `target teams distribute parallel for`, has insufficient
parallelism. Especially if the trip count of the loops is lower than the number
of threads possible times the number of teams (aka. blocks) the device prefers
(see also :ref:`LIBOMPTARGET_AMDGPU_TEAMS_PER_CU`), we will reduce the thread
count to increase outer (team/block) parallelism. The thread count will never
be reduced below the value passed for this environment variable though.

LIBOMPTARGET_REUSE_BLOCKS_FOR_HIGH_TRIP_COUNT
"""""""""""""""""""""""""""""""""""""""""""""

This environment variable can be used to control how the OpenMP runtime assigns
blocks to loops with high trip counts. By default we reuse existing blocks
rather than spawning new blocks.

OFFLOAD_TRACK_ALLOCATION_TRACES
"""""""""""""""""""""""""""""""

This environment variable determines if the stack traces of allocations and
deallocations are tracked to aid in error reporting, e.g., in case of
double-free.

OFFLOAD_TRACK_KERNEL_LAUNCH_TRACES
""""""""""""""""""""""""""""""""""

This environment variable determines how manytstack traces of kernel launches
are tracked to aid in error reporting, e.g., what asynchronous kernel failed.

.. _libomptarget_plugin:

LLVM/OpenMP Target Host Runtime Plugins (``libomptarget.rtl.XXXX``)
-------------------------------------------------------------------

The LLVM/OpenMP target host runtime plugins were recently re-implemented,
temporarily renamed as the NextGen plugins, and set as the default and only
plugins' implementation. Currently, these plugins have support for the NVIDIA
and AMDGPU devices as well as the GenericELF64bit host-simulated device.

The source code of the common infrastructure and the vendor-specific plugins is
in the ``openmp/libomptarget/nextgen-plugins`` directory in the LLVM project
repository. The plugin infrastructure aims at unifying the plugin code and logic
into a generic interface using object-oriented C++. There is a plugin interface
composed by multiple generic C++ classes which implement the common logic that
every vendor-specific plugin should provide. In turn, the specific plugins
inherit from those generic classes and implement the required functions that
depend on the specific vendor API. As an example, some generic classes that the
plugin interface define are for representing a device, a device image, an
efficient resource manager, etc.

With this common plugin infrastructure, several tasks have been simplified:
adding a new vendor-specific plugin, adding generic features or optimizations
to all plugins, debugging plugins, etc.

Environment Variables
^^^^^^^^^^^^^^^^^^^^^

There are several environment variables to change the behavior of the plugins:

* ``LIBOMPTARGET_SHARED_MEMORY_SIZE``
* ``LIBOMPTARGET_STACK_SIZE``
* ``LIBOMPTARGET_HEAP_SIZE``
* ``LIBOMPTARGET_NUM_INITIAL_STREAMS``
* ``LIBOMPTARGET_NUM_INITIAL_EVENTS``
* ``LIBOMPTARGET_LOCK_MAPPED_HOST_BUFFERS``
* ``LIBOMPTARGET_AMDGPU_NUM_HSA_QUEUES``
* ``LIBOMPTARGET_AMDGPU_HSA_QUEUE_SIZE``
* ``LIBOMPTARGET_AMDGPU_HSA_QUEUE_BUSY_TRACKING``
* ``LIBOMPTARGET_AMDGPU_TEAMS_PER_CU``
* ``LIBOMPTARGET_AMDGPU_MAX_ASYNC_COPY_BYTES``
* ``LIBOMPTARGET_AMDGPU_NUM_INITIAL_HSA_SIGNALS``
* ``LIBOMPTARGET_AMDGPU_STREAM_BUSYWAIT``

The environment variables ``LIBOMPTARGET_SHARED_MEMORY_SIZE``,
``LIBOMPTARGET_STACK_SIZE`` and ``LIBOMPTARGET_HEAP_SIZE`` are described in
:ref:`libopenmptarget_environment_vars`.

LIBOMPTARGET_NUM_INITIAL_STREAMS
""""""""""""""""""""""""""""""""

This environment variable sets the number of pre-created streams in the plugin
(if supported) at initialization. More streams will be created dynamically
throughout the execution if needed. A stream is a queue of asynchronous
operations (e.g., kernel launches and memory copies) that are executed
sequentially. Parallelism is achieved by featuring multiple streams. The
``libomptarget`` leverages streams to exploit parallelism between plugin
operations. The default value is ``1``, more streams are created as needed.

LIBOMPTARGET_NUM_INITIAL_EVENTS
"""""""""""""""""""""""""""""""

This environment variable sets the number of pre-created events in the
plugin (if supported) at initialization. More events will be created
dynamically throughout the execution if needed. An event is used to synchronize
a stream with another efficiently. The default value is ``1``, more events are
created as needed.

LIBOMPTARGET_LOCK_MAPPED_HOST_BUFFERS
"""""""""""""""""""""""""""""""""""""

This environment variable indicates whether the host buffers mapped by the user
should be automatically locked/pinned by the plugin. Pinned host buffers allow
true asynchronous copies between the host and devices. Enabling this feature can
increase the performance of applications that are intensive in host-device
memory transfers. The default value is ``false``.

LIBOMPTARGET_AMDGPU_NUM_HSA_QUEUES
""""""""""""""""""""""""""""""""""

This environment variable controls the number of HSA queues per device in the
AMDGPU plugin. An HSA queue is a runtime-allocated resource that contains an
AQL (Architected Queuing Language) packet buffer and is associated with an AQL
packet processor. HSA queues are used for inserting kernel packets to launching
kernel executions. A high number of HSA queues may degrade the performance. The
default value is ``4``.

LIBOMPTARGET_AMDGPU_HSA_QUEUE_SIZE
""""""""""""""""""""""""""""""""""

This environment variable controls the size of each HSA queue in the AMDGPU
plugin. The size is the number of AQL packets an HSA queue is expected to hold.
It is also the number of AQL packets that can be pushed into each queue without
waiting the driver to process them. The default value is ``512``.

LIBOMPTARGET_AMDGPU_HSA_QUEUE_BUSY_TRACKING
"""""""""""""""""""""""""""""""""""""""""""

This environment variable controls if idle HSA queues will be preferentially
assigned to streams, for example when they are requested for a kernel launch.
Should all queues be considered busy, a new queue is initialized and returned,
until we reach the set maximum. Otherwise, we will select the least utilized
queue. If this is disabled, each time a stream is requested a new HSA queue
will be initialized, regardless of their utilization. Additionally, queues will
be selected using round robin selection. The default value is ``true``.

.. _libomptarget_amdgpu_teams_per_cu:

LIBOMPTARGET_AMDGPU_TEAMS_PER_CU
""""""""""""""""""""""""""""""""

This environment variable controls the default number of teams relative to the
number of compute units (CUs) of the AMDGPU device. The default number of teams
is ``#default_teams = #teams_per_CU * #CUs``. The default value of teams per CU
is ``4``.

LIBOMPTARGET_AMDGPU_MAX_ASYNC_COPY_BYTES
""""""""""""""""""""""""""""""""""""""""

This environment variable specifies the maximum size in bytes where the memory
copies are asynchronous operations in the AMDGPU plugin. Up to this transfer
size, the memory copies are asynchronous operations pushed to the corresponding
stream. For larger transfers, they are synchronous transfers. Memory copies
involving already locked/pinned host buffers are always asynchronous. The default
value is ``1*1024*1024`` bytes (1 MB).

LIBOMPTARGET_AMDGPU_NUM_INITIAL_HSA_SIGNALS
"""""""""""""""""""""""""""""""""""""""""""

This environment variable controls the initial number of HSA signals per device
in the AMDGPU plugin. There is one resource manager of signals per device
managing several pre-created signals. These signals are mainly used by AMDGPU
streams. More HSA signals will be created dynamically throughout the execution
if needed. The default value is ``64``.

LIBOMPTARGET_AMDGPU_STREAM_BUSYWAIT
"""""""""""""""""""""""""""""""""""

This environment variable controls the timeout hint in microseconds for the
HSA wait state within the AMDGPU plugin. For the duration of this value
the HSA runtime may busy wait. This can reduce overall latency.
The default value is ``2000000``.

.. _remote_offloading_plugin:

Remote Offloading Plugin:
^^^^^^^^^^^^^^^^^^^^^^^^^

The remote offloading plugin permits the execution of OpenMP target regions
on devices in remote hosts in addition to the devices connected to the local
host. All target devices on the remote host will be exposed to the
application as if they were local devices, that is, the remote host CPU or
its GPUs can be offloaded to with the appropriate device number. If the
server is running on the same host, each device may be identified twice:
once through the device plugins and once through the device plugins that the
server application has access to.

This plugin consists of ``libomptarget.rtl.rpc.so`` and
``openmp-offloading-server`` which should be running on the (remote) host. The
server application does not have to be running on a remote host, and can
instead be used on the same host in order to debug memory mapping during offloading.
These are implemented via gRPC/protobuf so these libraries are required to
build and use this plugin. The server must also have access to the necessary
target-specific plugins in order to perform the offloading.

Due to the experimental nature of this plugin, the CMake variable
``LIBOMPTARGET_ENABLE_EXPERIMENTAL_REMOTE_PLUGIN`` must be set in order to
build this plugin. For example, the rpc plugin is not designed to be
thread-safe, the server cannot concurrently handle offloading from multiple
applications at once (it is synchronous) and will terminate after a single
execution. Note that ``openmp-offloading-server`` is unable to
remote offload onto a remote host itself and will error out if this is attempted.

Remote offloading is configured via environment variables at runtime of the OpenMP application:
    * ``LIBOMPTARGET_RPC_ADDRESS=<Address>:<Port>``
    * ``LIBOMPTARGET_RPC_ALLOCATOR_MAX=<NumBytes>``
    * ``LIBOMPTARGET_BLOCK_SIZE=<NumBytes>``
    * ``LIBOMPTARGET_RPC_LATENCY=<Seconds>``

LIBOMPTARGET_RPC_ADDRESS
""""""""""""""""""""""""
The address and port at which the server is running. This needs to be set for
the server and the application, the default is ``0.0.0.0:50051``. A single
OpenMP executable can offload onto multiple remote hosts by setting this to
comma-separated values of the addresses.

LIBOMPTARGET_RPC_ALLOCATOR_MAX
""""""""""""""""""""""""""""""
After allocating this size, the protobuf allocator will clear. This can be set for both endpoints.

LIBOMPTARGET_BLOCK_SIZE
"""""""""""""""""""""""
This is the maximum size of a single message while streaming data transfers between the two endpoints and can be set for both endpoints.

LIBOMPTARGET_RPC_LATENCY
""""""""""""""""""""""""
This is the maximum amount of time the client will wait for a response from the server.


.. _libomptarget_libc:

LLVM/OpenMP support for C library routines
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Support for calling standard C library routines on GPU targets is provided by
the `LLVM C Library <https://libc.llvm.org/gpu/>`_. This project provides two
static libraries, ``libcgpu.a`` and ``libllvmlibc_rpc_server.a``, which are used
by the OpenMP runtime to provide ``libc`` support. The ``libcgpu.a`` library
contains the GPU device code, while ``libllvmlibc_rpc_server.a`` provides the
interface to the RPC interface. More information on the RPC construction can be
found in the `associated documentation <https://libc.llvm.org/gpu/rpc.html>`_.

To provide host services, we run an RPC server inside of the runtime. This
allows the host to respond to requests made from the GPU asynchronously. For
``libc`` calls that require an RPC server, such as printing, an external handle
to the RPC client running on the GPU will be present in the GPU executable. If
we find this symbol, we will initialize a client and server and run it in the
background while the kernel is executing.

For example, consider the following simple OpenMP offloading code. Here we will
simply print a string to the user from the GPU.

.. code-block:: c++

   #include <stdio.h>

   int main() {
    #pragma omp target
      { fputs("Hello World!\n", stderr); }
   }

We can compile this using the ``libcgpu.a`` library to resolve the symbols.
Because this function requires RPC support, this will also pull in an externally
visible symbol called ``__llvm_libc_rpc_client`` into the device image. When
loading the device image, the runtime will check for this symbol and initialize
an RPC interface if it is found. The following example shows the RPC server
being used.

.. code-block:: console

    $ clang++ hello.c -fopenmp --offload-arch=gfx90a -lcgpu
    $ env LIBOMPTARGET_DEBUG=1 ./a.out
    PluginInterface --> Running an RPC server on device 0
    ...
    Hello World!

.. _libomptarget_device:

LLVM/OpenMP Target Device Runtime (``libomptarget-ARCH-SUBARCH.bc``)
--------------------------------------------------------------------

The target device runtime is an LLVM bitcode library that implements OpenMP
runtime functions on the target device. It is linked with the device code's LLVM
IR during compilation.

.. _libomptarget_dynamic_shared:

Dynamic Shared Memory
^^^^^^^^^^^^^^^^^^^^^

The target device runtime contains a pointer to the dynamic shared memory
buffer. This pointer can be obtained using the
``llvm_omp_target_dynamic_shared_alloc`` extension. If this function is called
from the host it will simply return a null pointer. In order to use this buffer
the kernel must be launched with an adequate amount of dynamic shared memory
allocated. This can be done using the ``LIBOMPTARGET_SHARED_MEMORY_SIZE``
environment variable or the ``ompx_dyn_cgroup_mem(<N>)`` target directive
clause. Examples for both are given below.

.. code-block:: c++

    void foo() {
      int x;
    #pragma omp target parallel map(from : x)
      {
        int *buf = llvm_omp_target_dynamic_shared_alloc();
        if (omp_get_thread_num() == 0)
          *buf = 1;
    #pragma omp barrier
        if (omp_get_thread_num() == 1)
          x = *buf;
      }
      assert(x == 1);
    }

.. code-block:: console

    $ clang++ -fopenmp --offload-arch=sm_80 -O3 shared.c
    $ env LIBOMPTARGET_SHARED_MEMORY_SIZE=256 ./shared

.. code-block:: c++

    void foo(int N) {
      int x;
    #pragma omp target parallel map(from : x) ompx_dyn_cgroup_mem(N * sizeof(int))
      {
        int *buf = llvm_omp_target_dynamic_shared_alloc();
        if (omp_get_thread_num() == 0)
          buf[N - 1] = 1;
    #pragma omp barrier
        if (omp_get_thread_num() == 1)
          x = buf[N - 1];
      }
      assert(x == 1);
    }

.. code-block:: console

    $ clang++ -fopenmp --offload-arch=gfx90a -O3 shared.c
    $ env ./shared

.. _libomptarget_device_allocator:

Device Allocation
^^^^^^^^^^^^^^^^^

The device runtime supports basic runtime allocation via the ``omp_alloc`` 
function. Currently, this allocates global memory for all default traits. Access 
modifiers are currently not supported and return a null pointer.

.. _libomptarget_device_debugging:

Debugging
^^^^^^^^^

The device runtime supports debugging in the runtime itself. This is configured
at compile-time using the flag ``-fopenmp-target-debug=<N>`` rather than using a
separate debugging build. If debugging is not enabled, the debugging paths will
be considered trivially dead and removed by the compiler with zero overhead.
Debugging is enabled at runtime by running with the environment variable
``LIBOMPTARGET_DEVICE_RTL_DEBUG=<N>`` set. The number set is a 32-bit field used
to selectively enable and disable different features.  Currently, the following
debugging features are supported.

    * Enable debugging assertions in the device. ``0x01``
    * Enable diagnosing common problems during offloading . ``0x4``
    * Enable device malloc statistics (amdgpu only). ``0x8``
    * Dump device PGO counters (only if PGO on GPU is enabled). ``0x10``
