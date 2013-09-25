
Running Google Tests with CTest 
===============================

Motivation
----------

So we love the [Google Test](http://code.google.com/p/googletest/) framework. It
provides us a nice test harness to perform unit testing for our codebase. It
also lets us utilize [**value
parameterized**](http://code.google.com/p/googletest/wiki/AdvancedGuide#Value_Parameterized_Tests)
tests. Basically, these kinds of tests let us run a single type of test over
many inputs. Sounds useful, right? It is.

The problem, though, is how they're named. Normally, if you have a unit test of
the form

.. code:: c++
   
   TEST(SomeCategory, SomeName) {
     // do some testing
   }

then the corresponding test name will be ```SomeCategory.SomeName```, and
everything's hunky-dory. Google provides CMake macros that will find all these
kinds of tests and provide the appropriate command to add them, i.e., 

.. code::
   
   ADD_TEST(SomeCategory.SomeName YourGTestExecutable --gtest_filter=SomeCategory.SomeName)

But what happens if you have a parameterized test? In the GTest framework, such
a test looks something like

.. code:: c++
   
   // a parameterized test
   TEST_P(SomeCategory, SomeName) {
     // test a given case with the parameter from GetParam()
   }
   
   // declare the parameters
   const SomeType params[] = {param1, param2, ...}

   // call the tests
   INSTANTIATE_TEST_CASE_P(SomeTestCase, SomeCategory,
                        ::testing::ValuesIn(params));

As expected, each parameter is a separate test when you run GTest, and they're
named ```SomeTestCase/SomeCategory.SomeName/n``` where ```n``` is the parameter
number currently being tested. 

Whereas the original test case was relatively easy to find using regular
expressions (which is what Google's macro does), the parameterized tests are
not.

Solution
--------

Outline
+++++++

I was originally inspired by another [blog
post](http://smspillaz.wordpress.com/2012/07/05/unit-test-autodiscovery-with-cmake-and-google-test/),
but didn't want to go about using the exact same approach.

In any case, the general approach is to do the following using CMake and
associated tools

* build the test exectuable
* run the executable with the --gtest_list_tests flag
* parse the output and add the tests appropriately

A few key insights are required for this workflow

* CMake has an
  [ADD_CUSTOM_COMMAND](http://www.cmake.org/cmake/help/cmake2.6docs.html#command:add_custom_command)
  macro that let's you run a shell command **after** a target has been built
* the ADD_TEST macro is actually copied to a file named CTestTestfile.cmake in the build directory
* the test target, i.e., ```make test``` simply reads CTestTestfile.cmake for each test

Specifics
+++++++++

I took a python-script based approach. The script needs the following command line arguments:

* exectuable -- which exectuable to run
* output -- where to put the parsed ADD_TEST macros (i.e., the CTestTestfile.cmake)

I called my script "generate_test_macros.py", and it's in my [source
tree](https://github.com/gidden/cyclus/blob/ctest-fix/src/Config/generate_test_macros.py). Assuming
that you know where the script is, the CMake command to implement this solution
is

.. code::
   
   set( tgt "SomeTestExecutable")
   set( script "path/to/the/script/generate_tests.py")
   set( exec "--executable=${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${tgt}")
   set( out "--output=${CYCLUS_BINARY_DIR}/CTestTestfile.cmake")
   add_custom_command(TARGET ${tgt}
     POST_BUILD
     COMMAND python ${script} ${exec} ${out}
     COMMENT "adding tests from ${tgt}"
     DEPENDS
     VERBATIM
     )

Now, after you run cmake, you can run the normal ```make``` and ```make test```
commands, and all your tests will run!
