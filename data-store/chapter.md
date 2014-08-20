DBDB
====

Dog Bed Database (it's like a couch, but not as nice).


Memory
------

I remember the first time I had a bug I was really stuck on. When I finished
typing in my BASIC program and ran it, weird sparkly pixels showed up on the
screen, and the program aborted early. When I went back to look at the code,
the last few lines of the program were gone! One of my mom's friends was a
P.Eng and she knew how to program, so my mom arranged a phone call so I could
explain the problem and get some feedback. Within a few minutes of talking, I
had figured out the problem: the program was too big, and had encroached into
video memory. Clearing the screen truncated the program, and the sparkles were
artifacts of Applesoft BASIC's behaviour of storing program state in RAM
just beyond the end of the program.

I learned to care about memory allocation.

I remember learning about pointers and how to allocate memory with malloc, to
stick records of arbitrary-length strings into a struct so I could sort them
by latitude and longitude.

I learned how data strutures are laid out in memory.

I understood that Erlang didn't have to copy data between processes, even
though it was "strictly message-passing", because everything was immutable.
I'm not sure that the utility of immutable data structures really sank in until
I read about Clojure's in 2009.
Who'd have thought that learning about CouchDB in 2013 would just make me smile
and nod?

I learned that you can can design systems built around immutable data.

Then I agreed to write a book chapter, and in trying to write a binary tree
algorithm that mutated the tree in place, I got frustrated with how complicated
things were getting, between edge cases and trying to see how changes in one
part of the tree affected others. I took a peek at a recursive algorithm for
updating immutable binary trees (INSERT LINK), and it turned out to be
relatively straightforward.

I learned that it's easier to reason about things that don't change.

So starts the story. ``:)``


What does it do?
----------------

Most projects require a data store of some kind.
You really shouldn't write your own;
there are many edge-cases that will bite you in the end,
even if you're just writing JSON to disk:

* What happens if your filesystem runs out of space?
* What happens if your laptop battery dies while saving?
* What if your data size exceeds available memory?
  (unlikely for most applications on modern desktop computers&hellip;but
  not unlikely for a mobile device or server-side web application)

However,
it's incredibly useful to understand
how data stores work.
It will inform your use
from a performance
as well as crash recovery
and durability perspectives.

DBDB is a (too) simple example
of how save key/value data
in an atomic and durable way,
as well as how a persistence layer can be structured
in code.


Simplifications
---------------

DBDB is not <http://en.wikipedia.org/wiki/ACID_(database)>[ACID].
While updates are atomic and durable,
consistency is not covered
(there are no constraints on the data stored)
and isolation is not guaranteed(1).
Application code can of course impose its own consistency guarantees,
but proper isolation requires a transaction manager
(which could probably be its own 500 lines;
maybe you should write it for the 2nd edition!).

1. Given key:values ``{a:1, b:1}``,
   and two transactions ``a = b + 1`` and ``b = a + 1``,
   full isolation requires that there's a way to run them
   one at a time and get the same effect as running them concurrently.
   A result of ``{a:2, b:3}`` or ``{a:3, b:2}`` are both possible with isolation.
   Because DBDB doesn't provide this "serializable" isolation property,
   you could end up with ``{a:2, b:2}`` instead.

Stale data is not reclaimed in this implementation,
so repeated updates
(even to the same key)
will eventually consume all disk space.
Postgres calls this reclamation "vacuuming"
(which makes old row space available for re-use),
and CouchDB calls it "compaction"
(by rewriting the "live" parts of the data store into a new file,
and atomically moving it over the old one).
DBDB can easily be enhanced to add a compaction feature,
but it is left as an exercise for the reader.


Intro to the toolchain
----------------------

The code is written in polyglot Python 2/3.

I recommended using ``virtualenv``
when installing dependencies:

```bash
500lines/data-store$ virtualenv env
...
500lines/data-store$ source env/bin/activate
(env)500lines/data-store$ pip install -r requirements.txt
...
```

Tests can be run using ``nosetests``:

```bash
(env)500lines/data-store$ nosetests
.....................
----------------------------------------------------------------------
Ran 21 tests in 0.102s

OK

```


Exploring the code
------------------

DBDB separates the concerns of "put this on disk somewhere"
(how data are laid out in a file; the physical layer)
from the logical structure of the data
(a binary tree in this example; the logical layer)
from the contents of the key/value store
(the association of key "a" to value "foo"; the public API).


### Organizational units

* ``tool.py`` defines
    a CLI tool
    for exploring a database
    from a terminal window.

* ``interface.py`` defines
    a class (``DBDB``)
    which implements the Python dictionary API
    using the concrete ``BinaryTree`` implementation.
    This is how you'd use DBDB inside a Python program.

* ``tree.py`` defines
    the logical layer.
    It's an abstract interface to a key/value store.

    - ``Tree`` is not tree-specific,
        and defers to a concrete sub-class to implement updates.
        It manages storage locking and dereferencing internal nodes.

    - ``ValueRef`` is a Python object that refers to
        a binary blob stored in the database.
        The indirection lets us avoid loading
        the entire data store into memory at once.

* ``binary_tree.py`` defines
    a concrete binary tree algorithm
    underneath the tree interface.

    - ``BinaryTree`` provides a concrete implementation
        of a binary tree, with methods for
        getting, inserting, and deleting key/value pairs.
        ``BinaryTree`` represents an immutable tree;
        updates are performed by returning a new tree
        which shares common structure with the old one.

    - ``BinaryNode`` implements a node in the binary tree.

    - ``BinaryNodeRef`` is a specialised ``ValueRef``
        which knows how to serialize and deserialize
        a ``BinaryNode``.

* ``storage.py`` defines
    physical layer.
    The ``Storage`` class
    provides persistent, append-only record storage.
    The only exception to the append-only policy
    is the atomic "commit" operation
    which updates the first few bytes of the file to point
    at a new "root" record.
    This is atomic for all block devices.
    A record is a length-delimited series of bytes.

These modules grew from attempting
to give each class a single responsibility.
In other words,
each class should have only one reason to change.


### How it works

DBDB's data structures are not strictly immutable internally,
but they are effectively immutable from the user's perspective.
Tree nodes are created
with an associated key and value,
and left and right children.
Those associates never change.
Updating a key/value pair involves creating new nodes
from the inserted/modified/deleted node
all the way back up to the new root.
Internally,
a node's private parts mutate only
to remember where its data was written (writing a node as part of a commit),
or to remember the data when it's read from disk (after the node was read from disk).

![Immutable tree](immutable.svg)

![Immutable tree with updates](immutable2.svg)

The insertion function returns a new root node,
and the old one is garbage collected if it's no longer referenced.
When it's time to commit the changes to disk,
the tree is walked from the bottom-up
("postfix" or "depth-first" traversal),
new nodes are serialised to disk,
and the disk address of the new root node is written atomically
(because single-block disk writes are atomic).

![Tree nodes on disk before update](nodes_on_disk_1.svg)

![Tree nodes on disk after update](nodes_on_disk_2.svg)

This also means that readers get lock-free access to a consistent view of the tree.

To avoid keeping the entire tree structure in memory concurrently,
when a logical node is read in from disk,
the disk address of its left and right children
(as well as its value)
are loaded into memory.
Accessing children and values
requires one extra function call to `NodeRef.get()`
to "really get" the thing.

[INSERT PIC OF TREE WALK]

When changes to the tree are not committed,
they exist in memory
with strong references from the root down to the changed leaves.
Additional updates can be made before issuing a commit
because `NodeRef.get()` will return the uncommitted value if it has one.


### Points of extensibility

The algorithm used to update the data store
can be completely changed out
by replacing the string ``BinaryTree`` in ``interface.py``.
In particular,
data stores of this nature tend to use B-trees
not binary trees
to improve the nodes-per-byte ratio
of the tree.

By default, values are stored by ``ValueRef``
which expects bytes as values
(to be passed directly to ``Storage``).
The binary tree nodes themselves
are just a sublcass of ``ValueRef``.
Storing richer data
(via ``json``, ``msgpack``, ``pickle``, or your own invention)
is just a matter of writing your own
and setting it as the ``value_ref_class``.


### Tradeoffs (time/space, performance/readability)

The binary tree is much easier to write
and hopefully easier to read
than other tree structures would have been.
Structures like B-tree, B+ tree, B\*-tree
[and others](http://en.wikipedia.org/wiki/Tree_%28data_structure%29)
provide superior performance,
particularly for on-disk structures like this.
While a balanced binary tree
(and this one isn't)
needs to do $O(log_2(n))$ random node reads to find a value,
a B+tree needs many fewer, e.g. $O(log_32(n))$
because each node splits 32 ways instead of just 2.
This makes a huge different in practise,
since looking through 4 billion entries would go from
$log_2(2^32) = 32$ to $log_32(2^32) \approx 6.4$ lookups.


### Patterns or principles that can be used elsewhere

Test interfaces, not implementation.
I wrote my first tests against an in-memory version of the database,
then extended it to persist to disk,
then added the concept of NodeRefs.
Most of the tests didn't have to change,
which gave me confidence that things were still working.

Single Responsibility Principle.
Classes should have at most one reason to change.
That's not strictly the case with DBDB,
but there are multiple avenues of extension
with only localised changes required.


Conclusion
----------

* Futher extensions to make:

    - Full and proper multi-client, non-clobbering support.
        Concurrent dirty readers already "just work",
        but concurrent writers,
        and readers-who-then-write
        could cause problems.
        Beyond making the implementation "safe",
        it's important to provide a useful
        and hard-to-use-incorrectly
        interface to users.

    - Database compaction.
        Compacting should be as simple as
        doing an infix-of-median traversal of the tree
        writing things out as you go.
        It's probably best if the tree nodes all go together,
        since they're what have to be traversed
        to find any piece of data.
        Packing as many intermediate nodes as possible
        into a disk sector
        should improve read performance
        at least right after compaction.

    - If compaction is enabled,
        it's probably not useful to truncate uncommitted writes
        on crash recovery.

   
* Similar real-world projects to explore:

    - CouchDB