---
layout: single
title:  "Adding test support"
tags: [programming, FIRE, CMake, cpp]
modified: 2019-03-06
categories: [FIRE]
excerpt_separator: <!-- more -->
classes: wide
toc: true
---

Welcome to the second post about my rendering engine project **FIRE**!

Within this series I want to describe how I build up a rendering engine from scratch and my thoughts about it along the way.
All the code will be open source and will be hosted here: [FIRE](https://github.com/markusrothe/FIRE)

In the [previous post](https://www.markusrothe.dev/fire/2019/03/04/FIRE-1-Setting-up-a-CMake-project.html), I talked about the initial setup of my project and how to get a basic CMake project (consisting of a library and an executable that uses the library) running.
Now, the next step is to add a unit tests executable for **FIRE** that uses the [gtest](https://github.com/google/googletest) unit testing framework.

<!-- more -->

We need to consider four steps:

1. We have to pull in gtest as a third-party dependency.
2. We need to create a test executable that links against gtest's libraries.
3. We'll have to create a dummy test to check whether everything works and the tests can be executed.
4. The tests should run after every build, so we have to setup a post-build step.

### Adding gtest
First, we have to get gtest into our own CMake project but fortunately for us, gtests is well-documented.
I chose to follow [this README](https://github.com/google/googletest/blob/master/googletest/README.md#incorporating-into-an-existing-cmake-project)'s advice to download gtest during FIRE's cmake configuration step and then build gtest as part of my project.

For that, two new CMake script files are necessary: `gtest_CMakeLists.txt.in` and `gtest_build.cmake`.
The first one contains CMake code to download gtest and add is as an external project to FIRE and the second script triggers the download and build of gtest itself.
Note that most of these two files have been taken directly from gtest's documentation.

{% highlight cmake linenos %}
#gtest_CMakeLists.txt.in
cmake_minimum_required(VERSION 2.8.2)

project(googletest-download NONE)

include(ExternalProject)
ExternalProject_Add(googletest
  GIT_REPOSITORY    https://github.com/google/googletest.git
  GIT_TAG           master
  SOURCE_DIR        "${CMAKE_CURRENT_BINARY_DIR}/googletest-src"
  BINARY_DIR        "${CMAKE_CURRENT_BINARY_DIR}/googletest-build"
  CONFIGURE_COMMAND ""
  BUILD_COMMAND     ""
  INSTALL_COMMAND   ""
  TEST_COMMAND      ""
)
{% endhighlight %}

`ExternalProject_Add` is a complex function.
I recommend checking out [its documentation](https://cmake.org/cmake/help/v3.10/module/ExternalProject.html) for all the details. 
For our purposes, we only need to specify that we want to pull in an external project, which is a git repository, the branch that we want to checkout and the source and binary directories where CMake will put all the code / builds for gtest.
Note the call to `include(ExternalProject)`. This function pulls in function definitions defined inside the `ExternalProject` CMake script that can usually be found within CMake's own install folder (CMake maintains a CMAKE_MODULE_PATH variable that lists all directories that CMake shall search when trying to `include()` a script, so you can append your own directories to it).

The second script (`gtest_build.cmake`) will trigger the download of gtest and its build.

{% highlight cmake linenos %}
#gtest_build.cmake
# Download and unpack googletest at configure time
configure_file(gtest_CMakeLists.txt.in googletest-download/CMakeLists.txt)
execute_process(COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" .
  RESULT_VARIABLE result
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
if(result)
  message(FATAL_ERROR "CMake step for googletest failed: ${result}")
endif()

execute_process(COMMAND ${CMAKE_COMMAND} --build .
  RESULT_VARIABLE result
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
if(result)
  message(FATAL_ERROR "Build step for googletest failed: ${result}")
endif()

# Prevent overriding the parent project's compiler/linker
# settings on Windows
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)

# Add googletest directly to our build. This defines
# the gtest and gtest_main targets.
add_subdirectory(${CMAKE_CURRENT_BINARY_DIR}/googletest-src
                 ${CMAKE_CURRENT_BINARY_DIR}/googletest-build
                 EXCLUDE_FROM_ALL)
{% endhighlight %}

Let's start with the first line:
{% highlight cmake %}
configure_file(gtest_CMakeLists.txt.in googletest-download/CMakeLists.txt)
{% endhighlight %}

`configure_file` takes two arguments: An input file path (usually a file inside the sources) and an output file path (usually a path inside the project's build directory tree). 
It will copy the input file to the specified output path. 
It is very common to see input files with the ending `.in` to denote that this is a file that will be used as an input and be copied somewhere into the build tree. 
During the copy, `configure_file` is also able to substitute values that are referenced as `@VAR@` or `${VAR}` in the input file. 
We do not need this substitution functionality, but for reference, take a look at `configure_file`'s [documentation](https://cmake.org/cmake/help/latest/command/configure_file.html).
In our case, we only copy the gtest_CMakeLists.txt.in into *googletest-download* inside our build directory.
Essentially, we get the gtest CMake project inside the *googletest-download* folder.

Next, we run CMake to configure and build the gtest project:

{% highlight cmake %}
execute_process(COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" .
  RESULT_VARIABLE result
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
...
execute_process(COMMAND ${CMAKE_COMMAND} --build .
  RESULT_VARIABLE result
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
{% endhighlight %}

Lastly, we have to add the gtest source directory in order to get gtest's library target name definitions that `FIRE's` build-targets can link against.

{% highlight cmake %}
...
add_subdirectory(${CMAKE_CURRENT_BINARY_DIR}/googletest-src
                 ${CMAKE_CURRENT_BINARY_DIR}/googletest-build
                 EXCLUDE_FROM_ALL)
{% endhighlight %}

All that is left to do now is to add everything to the `FIRE` CMake project:

{% highlight cmake linenos %}
#CMakeLists.txt
...
project(FIRE)
include(gtest_build)
...
{% endhighlight %}

Now, whenever we run CMake for `FIRE` the gtest CMake targets are pulled in.
Next, we need to create a test executable that can link against gtest.

### Creating the test executable
Lets take a look at FIRE's directory structure again:

{% highlight shell linenos %}
FIRE
|--CMakeLists.txt
|--FIRE
|  |--CMakeLists.txt
|  |--src
|  |--include
|  |--test
|  |  |--CMakeLists.txt
|  |  |--FIRETest.cpp
{% endhighlight %}

I've added a new CMakeLists.txt and a source-file *FIRETest.cpp* inside *FIRE/test*.
For CMake to consider the CMakeLists.txt file inside *FIRE/test*, a call to `add_subdirectory` is needed at the end of CMakeLists.txt one level above *FIRE/test*:

{% highlight cmake linenos %}
## FIRE/CMakeLists.txt
...
add_subdirectory(test)
{% endhighlight %}

Inside the test folder's CMakeLists.txt we'll define the new test executable target `FIRE_tests`. First, we'll create the executable via `add_executable` and pass in our only test source file.
Then, we'll specify all libraries that `FIRE_tests` will link against. In this case, as `FIRE_tests` is a test executable that no other target will depend on (it does not have any public headers itself that someone else might want to use), we link everything as PRIVATE. 

{% highlight cmake linenos %}
## FIRE/test/CMakeLists.txt
add_executable(FIRE_tests
    ./FIRETest.cpp
)

target_link_libraries(FIRE_tests
PRIVATE
    FIRE
    gtest
    gtest_main
    gmock
)
{% endhighlight %}

`FIRE_tests` needs to link against 4 different link-targets:
1. `FIRE` in order to be able to call `FIRE`s functions and test it.
2. `gtest` as the main part of the test framework. This library offers macros for us that we can use to create our own isolated unit test cases.
3. `gtest_main` to make use of gtest's own main function that starts up and runs all the test cases. This basically just contains a main function which initializes gtest and calls `RUN_ALL_TESTS()`. (We could easily use our own main function and call these functions ourselves but for now we do not need to.)
4. `gmock` as part of the test framework to create mocks of classes and interfaces which we might want to use inside our test cases.

Now we already have everything we need to build `FIRE_tests`. If we run cmake, compile the code and run the test-executable we'll get the following output:

{% highlight shell linenos %}
[==========] Running 0 tests from 0 test suites.
[==========] 0 tests from 0 test suites ran. (0 ms total)
[  PASSED  ] 0 tests.
{% endhighlight %}

We have not yet added any test-case, but we can see that the executaion of `FIRE_tests` works.

### Writing the first (dummy) test
Let us add a small dummy test case inside FIRE/test/FireTest.cpp:

{% highlight c++ linenos %}
// FIRE/test/FireTest.cpp
#include <FIRE/FIRE.h>
#include <gtest/gtest.h>

TEST(FIRETest, Hello_FIRE_Test)
{
    FIRE::HelloWorld();
    EXPECT_TRUE(true);
}
{% endhighlight %}

Here, we first include both, a header of FIRE and the main gtest header `gtest/gtest.h`. 
The compiler can find these headers because with our call to `target_link_libraries` CMake already configured the correct include paths. (CMake knows these because for example in the case of `FIRE` we provided them ourselves via a call to `target_link_libraries` when defining the `FIRE` library target.).
After the includes, we created a simple unit test case using gtest's `TEST()` macro.
For now we just call `FIRE`s only function `HelloWorld()` and add a (rather superfluous) assertion that checks whether true is in fact true... . Keep in mind that we only want to setup the test execution itself and do not (yet) really care about the content of our test cases.

Next, we need to make sure that the test executable is run after every compilation as a post-build step.

### Configuring a post-build step

Because we are using gtest as our testing framework, we can make use of utility functions, provided by CMake, that help us with setting up the post-build run of our test-cases. Inside `FIRE_tests`s CMakeLists.txt, we have to add the following lines:

{% highlight cmake linenos %}
include(GoogleTest)
gtest_discover_tests(FIRE_tests)

add_custom_command(TARGET FIRE_tests POST_BUILD
                   COMMAND ctest -C $<CONFIGURATION> --output-on-failure
)
{% endhighlight %}

By including the `GoogleTest` CMake script we gain access to a gtest-specific utility function `gtest_discover_tests()`. This function enumerates all the test cases inside the source files that are part of a given target. In our example, the test case `Hello_FIRE_test` from the previous section will be listed and the list will be fed to a CMake related utility called "ctest". ctest runs all tests that have been added to its list of tests (and `gtest_discover_tests()` just did that for us). In the last line(s) we call `add_custom_command` with our `FIRE_tests` target which adds a command that shall be run as a POST_BUILD step. As the command, we specify ctest. That way, whenever `FIRE_tests` has been built, the new custom command will be run. ctest comes with a multitude of options that you can look up [here](https://cmake.org/cmake/help/v3.0/manual/ctest.1.html).

However, we are not done yet.
We still need to enable ctest before anything will work. For that, we have to add these lines to the top-level CMakeLists.txt of the *FIRE* project:

{% highlight cmake linenos %}
#CMakeLists.txt
...
include(CTest)
enable_testing()
...
{% endhighlight %}

Now we're done. Whenever we run CMake, gtest will be updated (if needed) and its targets will be added to the *FIRE* project. For every build, the unit tests inside `FIRE_tests` are executed automatically. Essentially, we could start implementing the FIRE engine in a test-driven way now, but before that I want to add an automatic build job that is run with every commit and a way to measure the code coverage that we get from our unit tests.

(You can check out the code for this post via [this git tag](https://github.com/markusrothe/FIRE/tree/FIRE-2))
