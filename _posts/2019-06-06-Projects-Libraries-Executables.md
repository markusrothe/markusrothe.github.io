---
title: "Projects, Libraries and Executables"
layout: single
tags: [CMake, Tutorial]
toc: true
excerpt: How to start with CMake by creating projects, building libraries and executables.
---
Within this post I want to explain how to start building projects with the CMake build tools.

# Creating a CMake Project
CMake is organized in *projects*. A project is a collection of one or more *targets* (libraries, executables, ...) that can be built.
Most of the time a codebase contains only one single project. Sometimes, very complex code bases use multiple sub-projects within one top-level project.

## CMake Version & Project Names
The very first line in a CMake project should state the minimum version of CMake to use. 
This ensures that at least the given minimum version of CMake is working and ensures backwards compatibility down to the given version.
Afterwards, usually the project name is defined.
The project definition and version should be placed into a file `CMakeLists.txt`:

{% highlight cmake linenos %}
cmake_minimum_required(VERSION 3.10)
project(MyProject)
{% endhighlight %}

## Directories and CMakeLists.txt Files
CMake files are structured hierarchically throughout a project.
The project definition resides at the top level of the project. 
Sub-folders can be added to the project by calling the function `add_subdirectory()`.
Each sub-folder should then again contain a `CMakeLists.txt`.
Let's assume that we want to create both, a library called `foo` and an executable `bar` in our first CMake project.
We might consider the following folder structure for that:

{% highlight ascii linenos %}
MyProject/
├── CMakeLists.txt
├── foo/
│   ├── CMakeLists.txt
│   ├── include/
│   │   └── foo.h
│   └── src/
│       └── foo.cpp
└── bar/    
    ├── CMakeLists.txt
    └── src/
        └── main.cpp
{% endhighlight %}

We'll have two folders, one for a library (`foo`) that contains its public interface headers (inside `foo/include/`) and its implementation source files(inside `foo/src/`).
The other one is for the executable (`bar`), which also contains its implementation (inside `bar/src`).
In the top-level `CMakeLists.txt`, we'll have to add both folders as sub-directories.
{% highlight cmake linenos %}
add_subdirectory(foo)
add_subdirectory(bar)
{% endhighlight %}

When running CMake, it will step through the `CMakeLists.txt` files in both sub-directories.

# Building Targets
In "Modern CMake" (Basically CMake version 3.0 and higher), one should think about everything in terms of **targets**. A target can be a particular library or executable to build or a custom build step to be executed during the build. For now we are only concerned about libraries and executables. It is easiest to think about targets as some kind of "objects" that have their own properties which can be read or written.
A target's properties should also be the preferred way how information about a target is propagated (in contrast to custom variables for instance).
## Building Libraries
A library target can be added with a call to 
{% highlight cmake linenos %}
add_library(foo 
    include/foo.h
    src/foo.cpp 
)
{% endhighlight %}
The first parameter specifies the name of the lib, followed by the source files that shall be compiled into the lib.

## Include Directories
Usually, when working with non-trivial code bases, you want to organize your code in such a way that the public interface of your library is in their own folder, separated from the implementation files.
These are usually header files that you want to deliver alongside the compiled library for other projects to include.
If a client that is using your library wants to include one of your headers, the path to the headers of your lib have to be known by the compiler. The client usually specifies the include path in the when calling the compiler.
CMake allows to set a property at the target that describes the include path to be used for the target via the function `target_include_directories()`

{% highlight cmake linenos %}
target_include_directories(foo
    PUBLIC    
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> 
        $<INSTALL_INTERFACE:include>
)
{% endhighlight %}

Here, two properties of the target are populated with basically the same information, but with different meanings.

The `BUILD_INTERFACE` property describes the full path to the include directory of the lib within the context of *building* the lib / the current project. In the example, the `CMAKE_CURRENT_SOURCE_DIR` variable is used to get the full path to the directory that the `CMakeLists.txt` file which is currently being processed is located in (".../pathToTheProject/foo/"). 

The `INSTALL_INTERFACE` property is used for *clients outside of the project that contains the lib* which import the library from within another project. Importing libraries and using them will be covered in another post, though. 

## Building Executable Targets
Analogously to libraries, executable targets can be defined with the function 
{% highlight cmake linenos %}
add_executable(bar 
    src/main.cpp
)
{% endhighlight %}

When building, this will create an executable based on the given source files.

## Linking Libraries
Let's say you want to use a function defined in the lib `foo` within the executable `bar`. That means, you will need to know how to link the library and how to specify the include path in order to be able to include the library's public header(s). 
Both of these things are combined in the function 
{% highlight cmake linenos %}
target_link_libraries(bar
    PRIVATE
        foo
)
{% endhighlight %}
This function call means that `bar` should link against `foo` and it should do so without exposing the dependency to `foo` to the outside (PRIVATE).
If `bar` was not an executable, but another library this would become quite important. A client that is using `bar` might not want to pull in the dependency to `foo` itself and thus it should not get that kind of dependency transitively via `bar`'s dependency to `foo` either:

![image-center](/out/images/TransitiveDependency/TransitiveDependency.svg){: .align-center}

# Running CMake
In order to invoke CMake it is usually best to do so from a separate build folder and not directly inside the project's source folder.
Running it is then as easy as: 

{% highlight bash linenos %}
mkdir build     # Create a separate build folder
cd build        # Step into the new build folder
cmake ..        # Run CMake (Configuraton + Generation)
cmake --build . # Compile
{% endhighlight %}

CMake runs in two separate steps: "Configuration" and "Generation". 
The Configuration step populates the CMake cache by setting variables up that are defined throughout the project. You can then also choose to edit the cache by hand and running the Configure step again.
Afterwards, the Generation step will generate the respective Makefiles (or Visual Studio solutions etc.) based on the previously created configuration.

On Unix, using `cmake --build .` is basically equivalent to calling `make`.

# Code & Resources

The code for this post can be found on Github via the following tag:
[Projects_Libraries_Executables](https://github.com/markusrothe/cmake_essentials/tree/Projects_Libraries_Executables)

Also, checkout Daniel Pfeifer's [brilliant talk about modern CMake](https://www.youtube.com/watch?v=bsXLMQ6WgIk)