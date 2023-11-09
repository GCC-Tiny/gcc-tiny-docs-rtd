
******************
Regression Testing
******************

.. note:: 
  Work in progress

You now have a full functioning compiler. Great job getting to this point.

Next step is to ensure the current code does not regress as GCC upstream 
is updated, or new features are added to the GNU Tiny programming language.
GCC comes with an elaborate testing facility based on 
`DejaGnu <https://www.gnu.org/software/dejagnu>`_
and
`Expect <https://wiki.tcl-lang.org/page/Expect>`_
.

Just as an example of how important it is to keep validating that everything 
is still working after changes to the sourse code we just have to look at the
Tiny compiler itself. For the square root test program we wanted to ensure the
result remain consistent, so adding a simple test should not be a problem.

.. code-block::
    :linenos:

    var s : float;
    s := 2.0;
    var i : int;
    var x : float;
    x := 1.0;
    for i := 1 to 100 do
    x := 0.5 * (x + s / x);
    end
    write x; # { dg-output "1.414214" }
 
But lo and behold, the Tiny lexer had a bug when reading a comment on the very
last line of the program. See all the details in
`GCC Tiny Bug #46 <https://github.com/GCC-Tiny/gcc-tiny-docs-rtd/issues/46>`_
.


This blog will briefly introduce DejaGnu and Expect and add basic tests and 
validations of some of the sematic and lexical rules for the GNU Tiny language.

Prerequisites
=============

Use the package manager that comes with you distribution to install

- autogen
- DejaGnu

Check installed versions of DejaGnu and required softare Expect and Tcl.

.. code-block:: shell-session

    $ which runtest
    /usr/bin/runtest

To check the version of the runtest program change to the gcc-src folder. This 
is needed because the runtest program need various information regarding 
the configured compilation targets. 

.. code-block:: shell-session

    $ cd gcc-src
    $ runtest --version
    DejaGnu version 1.6.2
    Expect version  5.45.4
    Tcl version     8.6

With the basic installation done you can continue with understanding DejaGnu
and creation of testsuites for the Tiny programming language.

DejaGnu Introduction
====================

There are a number of good references on how to implement DejaGnu:

- `How to guide <https://www.embecosm.com/appnotes/ean8/ean8-howto-dejagnu-1.0.html>`_
- `DejaGnu documentation <https://www.gnu.org/software/dejagnu/dejagnu.pdf>`_
- `Expect programming <https://wiki.tcl-lang.org/page/Expect>`_
- `Expect Wiki <https://en.wikipedia.org/wiki/Expect>`_
- `Book Exploring Expect <https://learning.oreilly.com/library/view/exploring-expect/9781565920903/>`_

DejaGnu is using the tcl extension Expect to control interactions with 
the GCC compilers, linux utilities and the produced output.


Expect
------

Expect is a program to control interactive applications. These applications 
interactively prompt and expect a user to enter keystrokes in response. 
By using Expect, you can write simple scripts to automate these 
interactions. And using automated interactive programs, you will be 
able to solve problems that you never would have even considered before.


DejaGnu Testcases
-----------------

DejaGnu uses special formatted comments, also known as dejagnu directives, 
in the source code for testing purposes. For example

.. code-block:: shell

    write "GCC ROCKS" ; # { dg-output "GCC ROCKS" }

The comment contains the directive dg-output, which instructs DejaGnu to 
check the output from the compiled program and compare the output with the
text "GCC ROCKS". In this case the source will be compiled, linked and the
resulting progam will be executed. 
Other directives is used to check for specific messages from the various 
stages of the compiler. For example

.. code-block:: shell

    var i : bool;
    i := ; # { dg-error "unexpected ‘;’" }

Here the directive is checking for an expected syntax error message 
from the compiler. In the example there an unexpected semicolon.

DejaGnu Directives
------------------

To see the full list of supported DejaGnu directives look in the /usr/share/dejagnu/dg.exp

For the purpose of this blog just a few are used

- dg-do do-what-keyword
	'do-what-keyword' is tool specific and is passed unchanged to
	${tool}-dg-test.  An example is gcc where 'keyword' can be any of:
	preprocess, compile, assemble, link or run
	and will do one of: produce a .i, produce a .s, produce a .o,
	produce an a.out, or produce an a.out and run it. The default is
	'compile'.

- dg-error regexp comment
	Indicate an error message <regexp> is expected on this line.
	The test fails if it doesn't occur.

- dg-output regexp
	Indicate the expected output of the program is <regexp>.
	There may be multiple occurrences of this, they are concatenated.



.. graphviz  :  : part12_expect.dot

DejaGnu Setup
-------------

The instructions on how to integrate DejaGnu is fairly straight forward: 

1. Driver program in gcc-src/gcc/testsuite/tiny.dg/dg.exp
2. Callback definitions in gcc-src/gcc/testsuite/lib/tiny-dg.exp

Larger compilers also use a library for various functions. Placed in 
testsuite/lib/${tool}.dg. 
For Tiny there is currently no need for this utility function, but if the 
testsuite expands into more features, maybe it will be added.

tiny.dg/dg.exp
~~~~~~~~~~~~~~

This is the first file the runtest program loads relevant to the testsuite 
for the tool. It first includes the tiny-dg.exp file containing the callback 
definitions for the source compilation and then invokes the main 
functions: dg-init, dg-runtest and dg-finish.

Create testsuite/tiny.dg/dg.exp

.. code-block:: shell
    :linenos:

    #   Copyright (C) 2009-2023 Free Software Foundation, Inc.

    # This program is free software; you can redistribute it and/or modify
    # it under the terms of the GNU General Public License as published by
    # the Free Software Foundation; either version 3 of the License, or
    # (at your option) any later version.
    # 
    # This program is distributed in the hope that it will be useful,
    # but WITHOUT ANY WARRANTY; without even the implied warranty of
    # MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    # GNU General Public License for more details.
    # 
    # You should have received a copy of the GNU General Public License
    # along with GCC; see the file COPYING3.  If not see
    # <http://www.gnu.org/licenses/>.

    # testsuite/tiny.dg/dg.exp
    # Load support procs.
    load_lib tiny-dg.exp

    # If a testcase doesn't have special options, use these.
    global DEFAULT_TINYCFLAGS
    if ![info exists DEFAULT_TINYCFLAGS] then {
        set DEFAULT_TINYCFLAGS ""
    }

    # Initialize 'dg'.
    dg-init

    # Main loop.
    dg-runtest [lsort [glob -nocomplain $srcdir/$subdir/*.tiny ] ] "" $DEFAULT_TINYCFLAGS

    # All done.
    dg-finish


lib/tiny-dg.exp
~~~~~~~~~~~~~~~

The DejaGnu testsuite is a maze of callbacks and source based 
overwrites for local and global functions. It is daunting to fully
analyze how it really works. For now we will chose to do an implementation
that just do enough to showcase the possibilities for the GCC Testsuites. 
Later we might dive into what is really happening behind the scenes when the
runtest commands executes.

The next file we need to create defines the function tiny-dg-test, 
aka ${tool}-dg-test. This function will setup the needed parameters
for the compilation of the Tiny test source code. The file further defines
two helper functions that we could have placed in a lib/tiny.exp file, but
we will leave the refactoring for a later blog.

Create testsuite/lib/tiny-dg.exp

.. code-block:: shell
    :emphasize-lines: 47
    :linenos:

    #   Copyright (C) 2009-2023 Free Software Foundation, Inc.

    # This program is free software; you can redistribute it and/or modify
    # it under the terms of the GNU General Public License as published by
    # the Free Software Foundation; either version 3 of the License, or
    # (at your option) any later version.
    # 
    # This program is distributed in the hope that it will be useful,
    # but WITHOUT ANY WARRANTY; without even the implied warranty of
    # MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    # GNU General Public License for more details.
    # 
    # You should have received a copy of the GNU General Public License
    # along with GCC; see the file COPYING3.  If not see
    # <http://www.gnu.org/licenses/>.


    #
    # DejaGnu Setup for the Tiny language
    #   For details to the Dejagnu directives and more 
    #   see: https://gcc.gnu.org/onlinedocs/gccint/Directives.html
    #

    #
    # Define tiny callbacks for dg.exp.
    # Loading /usr/share/dejagnu/dg.exp
    #
    load_lib dg.exp

    #
    # tiny-dg-test
    #    This is called from share/dejagnu/dg.exp
    # 
    proc tiny-dg-test { prog do_what extra_tool_flags } {
    puts "+lib/tiny-dg.exp: tiny-dg-test [file rootname [file tail $prog]]"

        # Set up options, based on what we're going to do.
    #    - Setting of the compiler type is handled in the dejagnu file /usr/share/dejagnu/target.exp
    #      Use the c++ compiler as Tiny is integrated into the gcc compiler.
    #    - Suppress advanced diagnostic messages from gcc: additional_flags=-fdiagnostics-plain-output
        set options [list "c++" "additional_flags=-fdiagnostics-plain-output"]

        set compile_type [compile-type $do_what]
        set output_file  [compile-outfile $do_what $prog]

        verbose "tiny_compile $prog $output_file $compile_type $options" 4
        set comp_output [tiny_compile "$prog" "$output_file" "$compile_type" $options]

        return [list $comp_output $output_file]
    }


    # 
    # compile-type 
    # -- based on gcc-dg.exp, proc gcc-dg-test-1
    #    translate dg-do directive to compile type: preprocess, assembly, object, executable
    #
    proc compile-type { do_what } {
        switch $do_what {
    "preprocess" {
        set compile_type "preprocess"
    }
    "compile" {
        set compile_type "assembly"
    }
    "assemble" {
        set compile_type "object"
    }
    "link" {
        set compile_type "executable"
    }
    "run" {
        set compile_type "executable"
    }
    default {
        perror "$do_what: not a valid dg-do keyword"
        set compile_type ""
    }
        }
    return $compile_type 
    }

    # 
    # compile-outfile 
    # -- based on gcc-dg.exp, proc gcc-dg-test-1
    #    translate dg-do directive to compile outfile type: .i, .s, .o, -exe
    #
    proc compile-outfile { do_what prog } {
        switch $do_what {
    "preprocess" {
        set output_file "[file rootname [file tail $prog]].i"
    }
    "compile" {
        set output_file "[file rootname [file tail $prog]].s"
    }
    "assemble" {
        set output_file "[file rootname [file tail $prog]].o"
    }
    "link" {
        set output_file "[file rootname [file tail $prog]]-exe"
    }
    "run" {
        set output_file "./[file rootname [file tail $prog]]-exe"
    }
    default {
        perror "$do_what: not a valid dg-do keyword"
        set output_file ""
    }
        }
    return $output_file
    }

The compiler is invoked on line 47 via the function tiny_compile.

With the files in place the next step is to add the integration into the Makefile, 
and to start to populate the Tiny testsuites with meaningful tests.

GCC Integration of TestSuites
=============================

To take advantage of the integrated test suites there are a few changes needed 
outside the gcc-src/gcc/tiny source folder. You will have to update some of the GCC 
configuration files. Most of the GCC configuration files have been around for a very
long time, so do not get discouraged by all the content you will encounter. It used 
in various ways by the GCC toolchain. For now, just accept this and add the changes
needed for enabling the Tiny testsuites.

  1. First you need to let GCC know that your frontend can be included into the test automation framework by changing gcc-src/Makefile.def and gcc/tiny/Make-lang.in
  2. Second you need to create the GNU Tiny testsuites into the gcc-src/gcc/testsuite/tiny folder


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


The tool will not provide any prompt, so once autogen completes, it is recommended to 
check if your changes made it to the gcc-src/Makefile.in

Makefile.in
-----------

.. code-block:: shell
    
    $ git diff HEAD@{1} Makefile.in


.. code-block:: diff
    
    diff --git a/Makefile.in b/Makefile.in
    index 144bccd2603..5769cee1bbc 100644
    --- a/Makefile.in
    +++ b/Makefile.in
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

Looks like there are two new phony targets: check-gcc-tiny and check-tiny. 
We will disect this later once we execute the make check-tiny command.

Let's check everything is still working as expected

.. code-block:: shell

    $ cd gcc-build
    $ rm -rf *
    $ ../gcc/configure --prefix=$PWD/../gcc-install --disable-multilib --disable-bootstrap --enable-languages=c,c++,tiny
    $ make -j
    $ make -j install

If everything goes well you should see something like

.. code-block:: shell-session

    make[1]: Leaving directory '/home/chatai/github/gcc-build'

Now would be a good time to try the new target for the Tiny test suite.

.. code-block:: shell

    $ make check-tiny

The output will be very verbose. In a later section we will dive into the 
meaning of the output. For now the ... denote the shortened output.

.. code-block:: shell-session

    r=`${PWDCMD-pwd}`; export r; \
    s=`cd ../gcc; ${PWDCMD-pwd}`; export s;
    ...
    FLEX="flex"; export FLEX; LEX="flex"; ...
    ...
    (cd gcc && make ... check-tiny);
    make[1]: Entering directory '/home/chatai/github/gcc-build/gcc'
    make[1]: *** No rule to make target 'check-tiny'.  Stop.
    make[1]: Leaving directory '/home/chatai/github/gcc-build/gcc'
    make: *** [Makefile:19673: check-gcc-tiny] Error 2


The error indicates that make could not find the expected target check-tiny. 
Still some more work to complete this part of the integration.


gcc/tiny/Make-lang.in
---------------------

In the gcc/tiny/Make-lang.in file we need to let the generic test suite 
framework know that the Tiny language will use the standard testsuites 
features:

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

To activate the changes to the Make-lang.in file you need to run make again.

.. code-block:: shell

    $ make -j
    $ make -j install
    $ make check-tiny


.. code-block:: shell-session

    $ make check-tiny

    r=`${PWDCMD-pwd}`; export r; \
    s=`cd ../gcc; ${PWDCMD-pwd}`; export s; \
    FLEX="flex"; export FLEX; ...
    (cd gcc && make ... check-tiny);
    make[1]: Entering directory '/home/chatai/github/gcc-build/gcc'
    Making a new config file...
    echo "set tmpdir /home/chatai/github/gcc-build/gcc/testsuite" >> ./site.tmp
    rm -rf testsuite/tiny-parallel
    make[2]: Entering directory '/home/chatai/github/gcc-build/gcc'
    (rootme=`${PWDCMD-pwd}`; export rootme; \
    srcdir=`cd ../../gcc/gcc; ${PWDCMD-pwd}` ; export srcdir ; \
    if [ -n "" ] \
    && [ -n "$GCC_RUNTEST_PARALLELIZE_DIR" ] \
    && [ -f testsuite/tiny-parallel/finished ]; then \
    rm -rf testsuite/tiny; \
    else \
    cd testsuite/tiny; \
    rm -f tmp-site.exp; \
    sed '/set tmpdir/ s|testsuite$|testsuite/tiny|' \
            < ../../site.exp > tmp-site.exp; \
    /bin/bash ${srcdir}/../move-if-change tmp-site.exp site.exp; \
    EXPECT=`if [ -f ${rootme}/../expect/expect ] ; then echo ${rootme}/../expect/expect ; else echo expect ; fi` ; export EXPECT ; \
    if [ -f ${rootme}/../expect/expect ] ; then  \
        TCL_LIBRARY=`cd .. ; cd ${srcdir}/../tcl/library ; ${PWDCMD-pwd}` ; \
        export TCL_LIBRARY ; \
    fi ; \
    `if [ -f ${srcdir}/../dejagnu/runtest ] ; then echo ${srcdir}/../dejagnu/runtest ; else echo runtest; fi` --tool tiny ; \
    if [ -n "$GCC_RUNTEST_PARALLELIZE_DIR" ] ; then \
        touch ${rootme}/testsuite/tiny-parallel/finished; \
    fi ; \
    fi )
    WARNING: Couldn't find tool init file
    Test run by chatai on Sun Aug 27 10:01:24 2023
    Native configuration is x86_64-pc-linux-gnu

                    === tiny tests ===

    Schedule of variations:
        unix

    Running target unix
    Using /usr/share/dejagnu/baseboards/unix.exp as board description file for target.
    Using /usr/share/dejagnu/config/unix.exp as generic interface file for target.
    Using /home/chatai/github/gcc/gcc/testsuite/config/default.exp as tool-and-target-specific interface file.

                    === tiny Summary ===

    make[2]: Leaving directory '/home/chatai/github/gcc-build/gcc'
    make[1]: Leaving directory '/home/chatai/github/gcc-build/gcc'


The line === tiny summary === indicates the runtest command was invoked. 
As expected there is a warning that a tool init file cannot be found. 
This is from the runtest command. Let get started on added the needed 
setup for the execution of the runtest commands.

gcc-src/testsuites
------------------


GNU Tiny TestSuites
===================

Setup
-----

create gcc/testsuites/tiny
create tiny/dg.exp
create lib/tiny-dg.exp
create lib/tiny.exp


Lexical testing
---------------

Does the language scanner follow all the rules of the input characters, 
and will proper error messages get emitted if there are illegal constructs.

Syntactical testing
-------------------

Does the language parser follow all the rules of the syntax and will the 
compiler generate meaningful, even helpful, hints on how to fix the 
syntax error.


Semantically testing
--------------------

Will the compiled code produce the expected results, including error handling 
like zero divide, over/underflows etc.

Will datatypes be enforced, and proper diagnostic messages created to there 
are unsupported assignments or calculations, For example: a=10*true is not 
a valid expression and assignment.

Try it out
==========

make check-tiny


Next Steps
==========

- Add parallel testsuite execution
- Refactor Tiny utilities into separate file
- Explain all the DejaGnu directives
- Deep dive into runtest script
- ...
