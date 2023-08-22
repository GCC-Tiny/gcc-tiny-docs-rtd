
*******************
Adding A Test Suite
*******************

You now have a full functioning compiler. Great job getting to this point.

Next step is to ensure the current code does not regress as GCC upstream 
is updated, or new features is added to the GNU Tiny programming language.
GCC comes with an elaborate testing facility based on DejaGNU (TODO: LINK)

In this blog we will add basic tests and validations of some of the sematic 
and lexical rules for the GNU Tiny lanaguage.

Prerequisites
=============

Try

.. code-block:: shell-session

    $ cd gcc-build
    $ make check


If you have all the prerequistites in place you will, after some time, see

.. code-block:: shell-session

    ..... success ....

you can skip to the next session (TODO: link)

Use the package manager that comes with you distribution to install
- autogen
- DejaGnu

Check installed versions of DejaGnu and required softare Expect and Tcl.

.. code-block:: shell-session

    $ runtest --version
    DejaGnu version 1.6.2
    Expect version  5.45.4
    Tcl version     8.6

DejaGnu Introduction
====================

DejaGnu Testcases
-----------------


GCC Integration of TestSuites
=============================

GNU Tiny TestSuites
===================

Setup
-----

create gcc/testsuites/tiny
create dg.exp


Lexical testing
---------------

Does the language scanner follow all the rules of the input characters, and will proper error messages get emitted if there are illegal constructs.

Syntactical testing
-------------------

Does the language parser follow all the rules of the syntax and will the compiler generate meaningful, even helpful, hints on how to fix the syntax error.


Semantically testing
--------------------

Will the compiled code produce the expected results, including error handling like zero divide, over/underflows etc.

Will datatypes be enforced, and proper diagnostic messages created to there are unsupported assignments or calculations, For example: a=10*true is not a valid expression and assignment.

Try it out
==========

make check-tiny
