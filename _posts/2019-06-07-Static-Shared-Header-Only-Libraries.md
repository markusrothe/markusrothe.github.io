---
title: "Static-, Shared- & Header-Only Libraries"
layout: single
tags: [CMake, Tutorial]
toc: true
excerpt: How to create static, shared or header-only (interface) library targets with CMake.
---

Learn how to create different types of libraries in CMake. 
Throughout the post, we'll consider the following folder structure:

{% highlight ascii linenos %}
MyProject/
├── CMakeLists.txt
├── foo/
│   ├── CMakeLists.txt
│   ├── include/
│   │   ├── foo_headerOnly.h
│   │   └── foo.h
│   └── src/
│       └── foo.cpp
└── bar/    
    ├── CMakeLists.txt
    └── src/
        └── main.cpp
{% endhighlight %}

`foo` contains the code for the different library types that we want to create.
`bar` is an executable that links against the different libs.

# Static Library Targets
A static library bundles a set of functions and variables that have been compiled.
Linking against the static library leads to the code of the library being merged into another library or executable.
Only the code that is referenced by the client will get merged.

In CMake, a static library target can be created explicitly by adding the `STATIC` keyword to the call to `add_library`:

{% highlight cmake linenos %}
add_library(fooStatic STATIC
    include/foo.h
    src/foo.cpp
)
{% endhighlight %}

# Shared Library Targets
A shared library is not linked into an application during the build. 
It is loaded during runtime by the application that wants to use it.
This has the advantage that the code of the shared library can be shared among different applications.
The disadvantage, though, is that managing the dependencies to shared libraries can become quite difficult in contrast to a static lib where one can be sure that the lib is present in a proper version.

In CMake, a shared library target can be created by adding the `SHARED` keyword to the call to `add_library`.

{% highlight cmake linenos %}
add_library(foo SHARED 
    include/foo.h
    src/foo.cpp
)
set_target_properties(foo
    PROPERTIES
    POSITION_INDEPENDENT_CODE 1
)
{% endhighlight %}

Note that a shared library has to be compiled as position independent because no matter at which memory address the library is going to be loaded, it still has to function properly.
The `POSITION_INDEPENDENT_CODE` property (which can be set via the call to `set_target_properties()`) makes sure that the library is compiled with the `-fPIC` compiler flag.

# Header-Only Library Targets
A header-only library has the benefit that it does not need to be compiled into a library.
However, handling dependencies to other libraries might prevent the creation of such a library.
Additionally, all source code of a header-only library is published which is something that a library creator might not want.

In CMake, a header-only library target is called an Interface library and is created by adding the `INTERFACE` keyword to the call to `add_library`:

{% highlight cmake linenos %}
add_library(fooHeaderOnly INTERFACE)
target_include_directories(fooHeaderOnly
    INTERFACE 
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> 
        $<INSTALL_INTERFACE:include>
)
{% endhighlight %}

Although no build output is created, properties can still be set on the target. Basically, the interface library target can be used in the same way as any other target.
Note that properties can only be set on the "Interface level". 
That means, the target's INTERFACE_* properties can be set. 
Other properties that would require some kind of build-output cannot be set.

# Object Libraries
If a library is to be built both as a shared lib and as a static lib, CMake allows to create so-called object library targets.
This type of library target allows for the code to only be compiled once and not separately for each lib.

{% highlight cmake linenos %}
add_library(objfoo OBJECT
    include/foo.h
    src/foo.cpp
)
add_library(foo SHARED $<TARGET_OBJECTS:objfoo>)
add_library(fooStatic STATIC $<TARGET_OBJECTS:objfoo>)
{% endhighlight %}

The shared and static library targets can then be declared by using the `TARGET_OBJECTS` property of the object library target.

# Code & Resources

The code for this post can be found on Github via the following tag:
[Static-, Shared- & Header-Only libs](https://github.com/markusrothe/cmake_essentials/tree/Static-Shared-Header-Only-Libraries)

Also, checkout Daniel Pfeifer's [brilliant talk about modern CMake](https://www.youtube.com/watch?v=bsXLMQ6WgIk)