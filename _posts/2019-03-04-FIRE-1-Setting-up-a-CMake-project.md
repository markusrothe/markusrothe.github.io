---
layout: dark-post
title:  "(FIRE-1) Setting up a CMake project"
tags: [programming, FIRE, CMake, cpp]
modified: 2019-03-04
categories: [FIRE]
---

Welcome to the first of a series of posts about my current spare time project: The **FIRE** rendering engine.
Within this series I want to describe how I build up the engine from scratch and my thoughts about it along the way.
All the code will be open source and will be hosted here: [FIRE](https://github.com/markusrothe/FIRE/tree/rework)

Before even starting to code any line I want to set up several things up front:

1. A build environment that supports me in targeting both Windows (64Bit) and Linux (x86_64) platforms and possibly more in the future.
2. A test environment that helps my development using a Test-Driven-Design (TDD) approach.
3. An automatic build job to continuously compile all code and run all unit tests.
4. Unit test coverage measurements.

In this post, I will focus on the first point: *The build environment*.

I am using [CMake](https://cmake.org/) as this allows me to either generate Makefiles on Linux or Visual Studio Solution files on Windows.
It also allows for easy building and testing (and later: packaging). 
The way CMake works is by adding `CMakeLists.txt` script files throughout a project's directories that describe the several build targets (libraries or executables) that shall be built.
Here is the initial directory structure for *FIRE*:

{% highlight c++ %}
FIRE
|--CMakeLists.txt
|--FIRE
|  |--CMakeLists.txt
|  |--src
|  |  |--FIRE.cpp
|  |--include
|  |  |--FIRE
|  |     |--FIRE.h
|  |--test
|  |  |-- //...
|--examples
|  |--CMakeLists.txt
|  |--example1
|  |  |--CMakeLists.txt
|  |  |--main.cpp
//...
{% endhighlight %}

The top-level `CMakeLists.txt` contains general information about the CMake project.
For now, we only need the version of CMake that we want to support as the minimum and the name of the project.

(I chose version 3.12 as the minimum because I want to use "modern" CMake principles using CMake targets and target properties --> more on that later!)

{% highlight c++ %}
## CMakeLists.txt
cmake_minimum_required(VERSION 3.0)

project(FIRE)

add_subdirectory(FIRE)
add_subdirectory(examples)
{% endhighlight %}

The last two lines instruct CMake to continue its processing in the respective subdirectories (in order).
The first subdirectory is intended for the FIRE library itself.
The second subdirectory will contain example executables that use the FIRE library.

Here is the CMakeLists.txt inside the FIRE subdirectory:

{% highlight cmake %}
## FIRE/CMakeLists.txt

add_library(FIRE STATIC
	include/FIRE/FIRE.h
	src/FIRE.cpp
)

target_include_directories(FIRE
PUBLIC 
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> 
  $<INSTALL_INTERFACE:include> 
)
{% endhighlight %}

The call to `add_library` defines a new library target `FIRE` along with its source files that make up the library.
For now, `FIRE` will be built as a static library (therefore the STATIC keyword behind the target name).

After a call to `add_library`, the target can be used in other CMake functions as in `target_include_directories` or `target_link_libraries`.

The next function `target_include_directories` is used to specify the include directories for the given target which is used when compiling the target.
Other targets that link against `FIRE` via `target_link_libraries` will automatically also get the include path to `FIRE`s public headers that way.
Note that we specify two kinds of properties for the `FIRE` target: BUILD_INTERFACE and INSTALL_INTERFACE. The first one is used for targets that are built as part of the same CMake project (such as our example executables).
The second one is used by external projects that might use an installed version of FIRE (using CMake's idea of an install prefix where the library will get installed to once we configure it that way - I will talk about this in a future post).

With those two CMakeLists.txt files we can already build the `FIRE` library. The two mentioned source files `FIRE.h` and `FIRE.cpp` will only contain a simple function that prints a string to `std::cout` for now (just to have something that can be executed): 
{% highlight c++ %}
// FIRE/include/FIRE/FIRE.h
namespace FIRE
{
    void HelloWorld();
}
{% endhighlight %}

{% highlight c++ %}
// FIRE/src/FIRE.cpp
#include <FIRE/FIRE.h>
#include <iostream>

namespace FIRE
{
    void HelloWorld()
    {
        std::cout << "Hello FIRE!\n";
    }
}
{% endhighlight %}

The next step is to create an executable that uses FIRE.
For that, we implement a CMakeLists.txt file that is very similar to FIRE's CMakeLists.txt:

{% highlight cmake %}
## FIRE/examples/example1/CMakeLists.txt

add_executable(example1
	main.cpp
)

target_link_libraries(example1
PRIVATE
	FIRE
)
{% endhighlight %}

First, we define an executable target (`example1`) via `add_executable` with its related source file(s), similar to the `FIRE` library target created with `add_library`.
Next we specifiy what libraries the executable target shall link against using `target_link_libraries`.
The PRIVATE keyword specifies that we are using `FIRE` as a private dependency. 

If we were creating another library (say something like `FIREAddons.lib`) instead of an executable we might have wanted to specify the dependency to `FIRE` as either PUBLIC or INTERFACE to show how exactly `FIREAddon` depends on `FIRE`. (For instance, if `FIREAddons` would use `FIRE` by including one of `FIRE`s headers inside one of its own public headers, that would signal a transitive dependency from another executable or lib through `FIREAddons` to `FIRE`.
Anyone that uses `FIREAddons` would have to also link against `FIRE` itself.
But, in our case, we are creating an executable that no one will ever depend on, so a PRIVATE dependency is the way to go.

Inside `example1`s main.cpp we just call the function that we defined inside `FIRE`:

{% highlight c++ %}
#include <FIRE/FIRE.h>

int main() { FIRE::HelloWorld(); }
{% endhighlight %}

Now all that is left to do is build our project and run `example1` to see that everything works:

{% highlight bash %}
$> mkdir build
$> cd build
$> cmake ..
$> cmake --build .
# ... navigate to the executable inside the build directory and run it.
{% endhighlight %}

First, we create a build directory for cmake to generate its build files in.
Next, we step into the build folder and call cmake with the repositories root directory.
Finally, we compile everything via cmake. (On Linux we could have just called make directly. On Windows, a .sln file would have been generated that we could open inside Visual Studio).

Depending on the platform, the binaries will be created in different folders. Once we find the `example1` executable we can run it.

In the next post we'll set up the unit test environment for `FIRE` using the googletest unit testing framework.
To check out the code of this post, take a look at this git tag: [FIRE-1](https://github.com/markusrothe/FIRE/tree/FIRE-1)
