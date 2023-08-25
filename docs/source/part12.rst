
*******************
Adding A Test Suite
*******************

.. note:: 
  Work in progress

You now have a full functioning compiler. Great job getting to this point.

Next step is to ensure the current code does not regress as GCC upstream 
is updated, or new features is added to the GNU Tiny programming language.
GCC comes with an elaborate testing facility based on DejaGNU (TODO: LINK)

In this blog we will add basic tests and validations of some of the sematic 
and lexical rules for the GNU Tiny language.

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

To take advantage of the integrated test suites there are a few changes needed 
outside the gcc-src/gcc/tiny source folder. You will have to update some of the GCC 
configuration files. Most of the GCC configuration files have been around for a very
long time, so do not get discouraged by all the content you will encounter. It used 
in various ways by the GCC toolchain. For now, just accept this and add the changes
needed for enabling the Tiny testsuites.

  1. First you need to let GCC know that your frontend can be included into the test automation framework by changing gcc-src/Makefile.def.
  2. Second you need to add include the Tiny testsuites into the gcc-src/testsuite/tiny folder.


Makefile.def
------------

In the file gcc-src/Makefile.def, find where the languages variable is being set 
and scroll to the end of the assignments, and the tiny language

.. code-block:: Makefile

    languages = { language=tiny;   gcc-check-target=check-tiny; };

Validate the content by using the git diff command:

.. code-block:: shell

    $ cd gcc-src
    $ git diff HEAD@{1} Makefile.def


The diff should show just the one line you added.

.. code-block:: diff

    diff --git a/Makefile.def b/Makefile.def
    index 9b4a8a2bf7a..77972f55073 100644
    --- a/Makefile.def
    +++ b/Makefile.def
    @@ -688,6 +688,7 @@ languages = { language=d;   gcc-check-target=check-d;
                                    lib-check-target=check-target-libphobos; };
     languages = { language=jit;    gcc-check-target=check-jit; };
     languages = { language=rust;   gcc-check-target=check-rust; };
    +languages = { language=tiny;   gcc-check-target=check-tiny; };
    
     // Toplevel bootstrap
     bootstrap_stage = { id=1 ; };

With these changes in place you are now ready to let GCC know about the new language Tiny.

autogen
-------

The autogen tool will generate the Makefile.in file based on the content of Makefile.def

.. code-block:: shell
    
    $ autogen Makefile.def


The tool will not provide any prompt, so once autogen completes, it is recommended to check if you changes made it to the gcc-src/Makefile.in

Makefile.in
-----------

.. code-block:: shell
    
    git diff HEAD@{1} Makefile.in


.. code-block:: diff
    
    diff --git a/Makefile.in b/Makefile.in
    index 144bccd2603..5769cee1bbc 100644
    --- a/Makefile.in
    +++ b/Makefile.in
    @@ -444,7 +444,7 @@ LIBCFLAGS = $(CFLAGS)
    CXXFLAGS = @CXXFLAGS@
    LIBCXXFLAGS = $(CXXFLAGS) -fno-implicit-templates
    GOCFLAGS = $(CFLAGS)
    -GDCFLAGS = @GDCFLAGS@
    +GDCFLAGS = $(CFLAGS)
    GM2FLAGS = $(CFLAGS)
    
    # Pass additional PGO and LTO compiler options to the PGO build.
    @@ -61790,6 +61790,14 @@ check-gcc-rust:
            (cd gcc && $(MAKE) $(GCC_FLAGS_TO_PASS) check-rust);
    check-rust: check-gcc-rust
    
    +.PHONY: check-gcc-tiny check-tiny
    +check-gcc-tiny:
    +       r=`${PWD_COMMAND}`; export r; \
    +       s=`cd $(srcdir); ${PWD_COMMAND}`; export s; \
    +       $(HOST_EXPORTS) \
    +       (cd gcc && $(MAKE) $(GCC_FLAGS_TO_PASS) check-tiny);
    +check-tiny: check-gcc-tiny
    +
    
    # The gcc part of install-no-fixedincludes, which relies on an intimate
    # knowledge of how a number of gcc internal targets (inter)operate.  Delegate.

Looks like there are two new targets: check-gcc-tiny and check-tiny.


.. TODO: check make check-tiny at this point with out the changes totiny/Make-lang.in 

Just a few more steps to be completed.

gcc/tiny/Make-lang.in
---------------------

In the gcc/tiny/Make-lang.in file we need to let the generic test suite framework know that the Tiny language will use the standard testsuites features:

.. code-block:: shell
    
    $ git diff HEAD@{1} Make-lang.in

.. code-block:: diff

    diff --git a/gcc/tiny/Make-lang.in b/gcc/tiny/Make-lang.in
    index c473b974c08..a3405518e46 100644
    --- a/gcc/tiny/Make-lang.in
    +++ b/gcc/tiny/Make-lang.in
    @@ -81,3 +81,10 @@ tiny.stagefeedback: stagefeedback-start
            -mv tiny/*$(objext) stagefeedback/tiny
    
    selftest-tiny:
    +
    +# List of targets that can use the generic check- rule and its // variant.
    +# Invoke with make check-tiny
    +lang_checks += check-tiny
    +lang_checks_parallelized += check-tiny
    +# For description see the check_$lang_parallelize comment in gcc/Makefile.in.
    +check_tiny_parallelize = 10000



.. TODO: check make check-tiny at this point with out the adding the gcc-src/gcc/testsuites/tiny



gcc-src/testsuites
------------------


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
