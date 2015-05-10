A Guide to Fixing memory leaks with Valgrind and GDB
====================================================
:date: 2015-05-10 12:32
:slug: memory_leaks
:author: Ian Kronquist

Valgrind and GDB are some of a C or C++ programmer's most powerful tools for
fixing memory leaks and segfaults. Anyone interested in writing either of these
languages should master these tools. This code and exercise is based on some
buggy code I wrote just a couple days ago. The steps here are almost literally
the same steps I took to fix my program. This exercise should give you insight
into how to use Valgrind and GDB effectively.

This adapted from an exercise_ I wrote for the `DevOps BootCamp`_ program at
OSU.

.. _exercise: https://github.com/DevOpsBootcamp/Bootcamp-Exercises/tree/master/debuggers-and-dev-tools/memory_leaks

.. _DevOps Bootcamp: http://devopsbootcamp.osuosl.org

Before going further, you should be familiar with the following:

* You should know what a tree data structure is, and generally how one is implemented (don't worry, the implementation is given to you).
* What do `malloc` and `free` do? (Hint: the same thing as `new` and `delete` in C++)
* You should read the man page for `realloc`.
* You might have written something which used a pointer to a pointer (`int**`)

I'm writing an interpreter. An interpreter builds and tree data structure from
my new language's source code. `ast` stands for Abstact Syntax Tree.
In my code, a token is the data held by an individual node in the tree.

We are interested in two files, `ast.c`_ and `ast_tests.c`_. The first is the
implementation of the tree, and the second is a series of working tests for the
tree.

.. _ast.c: https://github.com/DevOpsBootcamp/Bootcamp-Exercises/blob/master/debuggers-and-dev-tools/memory_leaks/ast.c

.. _ast_tests.c: https://github.com/DevOpsBootcamp/Bootcamp-Exercises/blob/master/debuggers-and-dev-tools/memory_leaks/ast_tests.c

Even though the tests pass, there are two memory leaks. We're going to fix
them. You will need gcc, make, valgrind, and gdb installed.

First, compile the project:

.. code-block:: bash

	$ make
	gcc -c ast.c -o ast.o -O0 -Wall -g -std=c99
	ast.c: In function ‘append_child’:
	ast.c:24: warning: ignoring return value of ‘realloc’, declared with attribute
	warn_unused_result
	gcc ast_tests.c ast.o -o ast_tests -O0 -Wall -g -std=c99

Hmmm, a warning. Let's make note of that and return to it in a minute.

Then, run the tests:

.. code-block:: bash

	$ ./ast_tests

There seems to be an awful looking memory error.

Let's try running the tests with valgrind:

.. code-block:: bash

	$ valgrind ./ast_tests


You should see three types of errors:


.. code-block:: text

	==75081== Invalid write of size 8
	==75081== Invalid read of size 8
	==75081== Conditional jump or move depends on uninitialised value(s)


And this summary at the end:

.. code-block:: text

	==15534== HEAP SUMMARY:
	==15534==     in use at exit: 4 bytes in 1 blocks
	==15534==   total heap usage: 34 allocs, 34 frees, 756 bytes allocated
	==15534== 
	==15534== LEAK SUMMARY:
	==15534==    definitely lost: 4 bytes in 1 blocks
	==15534==    indirectly lost: 0 bytes in 0 blocks
	==15534==      possibly lost: 0 bytes in 0 blocks
	==15534==    still reachable: 0 bytes in 0 blocks
	==15534==         suppressed: 0 bytes in 0 blocks
	==15534== Rerun with --leak-check=full to see details of leaked memory


Let's look at that warning again:

.. code-block:: bash

	$ gcc -c ast.c -o ast.o -O0 -Wall -g -std=c99
	ast.c: In function ‘append_child’:
	ast.c:24: warning: ignoring return value of ‘realloc’, declared with attribute
	warn_unused_result

Let's remind ourselves what `realloc` does

.. code-block:: bash

	$ man realloc


So, line 24 should really look like this:

.. code-block:: c

	node->children = realloc(node->children, node->num_children * 2);


If we rerun valgrind we see that all of our memory is no successfully freed!
We don't have anymore memory leaks.
But, what about these invalid reads and writes?
Let's take a look at the first one:

.. code-block:: text

	==16938== Invalid write of size 8
	==16938==    at 0x401889: append_child (ast.c:26)
	==16938==    by 0x400F99: test_append_child (ast_tests.c:92)
	==16938==    by 0x4005BB: main (ast_tests.c:33)
	==16938==  Address 0x4c3b570 is 0 bytes after a block of size 0 alloc'd
	==16938==    at 0x4A06A2E: malloc (vg_replace_malloc.c:270)
	==16938==    by 0x40172E: make_node (ast.c:11)
	==16938==    by 0x400EF7: test_append_child (ast_tests.c:88)
	==16938==    by 0x4005BB: main (ast_tests.c:33)


It happens on line 26 of `ast.c`:

.. code-block:: c

    node->children[node->num_children] = val;


Okay, let's run fire up gdb and see what happens on that line.

.. code-block:: text

	$ gdb ./ast_tests
	(gdb) break ast.c:26
	Breakpoint 1 at 0x40186e: file ast.c, line 26.
	(gdb) run
	Breakpoint 1, append_child (node=0x6021c0, val=0x602140) at ast.c:26
	26	    node->children[node->num_children] = val;

We can print the fields of the node struct. First, lets check that `node->children` isn't
NULL, or something similarly silly, and then lets check that the number of children is less than the capacity of children.


.. code-block:: text

	(gdb) print node->children
	$1 = (struct ast_node **) 0x602070
	(gdb) print node->num_children
	$2 = 0
	(gdb) print node->children_cap
	$3 = 2

Everything seems fine, let's continue...


.. code-block:: text

	(gdb) continue
	Continuing.

We hit the breakpoint and continue a few more times before the program explodes in our face.
Okay then, that didn't pan out. Let's see what other lines might screw up our program's
memory, maybe, that realloc line we were looking at earlier:


.. code-block:: text

	(gdb) break ast.c:24
	Breakpoint 1 at 0x401844: file ast.c, line 24.
	(gdb) print node->num_children
	$4 = 2

It looks like the program has a problem after we hit that line.
We want that line to double the size of our space for the children nodes.
The first argument was our pointer, which seems fine, and the second argument was
the number of bytes we want to allocate. We want to double our allocation so we put
twice our number of children, which is four bytes.
Wait, how big is a single child node?


.. code-block:: text

	(gdb) print sizeof(struct ast_node*)
	$5 = 8

This looks like a problem. We want enough space for four pointers to our nodes, which are each
8 bytes long. We allocated 4 bytes total, not even enough for one node.
Then line 24 should look like:

.. code-block:: c

	node->children = realloc(node->children,
		sizeof(struct ast_node*) * node->num_children * 2);

Now let's rerun valgrind:

.. code-block:: bash

	$ make
	$ valgrind ./ast_tests
	...
	==19899== ERROR SUMMARY: 4 errors from 4 contexts (suppressed: 6 from 6)

It looks like we just halved the number of errors!

Well, our last error was from allocating the wrong amount of space
using realloc. Let's look at what our malloc  line does:

.. code-block:: c

	new_node->children = malloc(sizeof(struct ast_node*) * num_children);

Everything seems to be in order, we're making an array of ast_nodes which
is as long as `num_children`.

We resize our allocation when we run out of capacity for children pointers.
We allocate enough space for the number of children we're given, and set our
capacity to be two more than that. Wait a minute, we don't allocate enough space.
We need to allocate two extra slots:

.. code-block:: c

    new_node->children = malloc(sizeof(struct ast_node*) * (num_children + 2));


Rerun valgrind, and all of our problems disappear.
The program no longer has any memory leaks.
