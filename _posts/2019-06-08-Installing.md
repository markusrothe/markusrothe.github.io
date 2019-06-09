---
title: "Installing"
layout: single
tags: [CMake, Tutorial]
toc: true
excerpt: How to install files and targets with CMake.
---

After building a CMake project, you might want to copy all the build artifacts to some location on your file system.
CMake's `install()` function allows you to easily define rules for installing CMake targets, files, directories etc. 

# Specifying The Install Destination
No matter what you want to install, all versions of the `install()` command take a `DESTINATION` parameter to specify where you want to install something.
If you provide a full path as the destination, the path will be used directly.
If you provide a relative path, it will be interpreted relative to the value of the CMake variable `CMAKE_INSTALL_PREFIX`. 

# Installing Files & Directories
To install a file call:

{% highlight cmake linenos %}
install(FILES <files...>
    DESTINATION <path>
)
{% endhighlight %}

If you want to rename the file while copying you can specify the `RENAME` parameter. 
If you do that, you can only specify one file to install and not a list of files.
There are additonal parameters to set the file permissions or to specify for which CMake Configuration (Debug, Release, etc.) the install command shall be executed.

Similar to installing single files, you can install whole directories:
{% highlight cmake linenos %}
install(DIRECTORY <dirs...>
    DESTINATION <path>
)
{% endhighlight %}

# Installing targets

Similar to installing files, you can also install a CMake target that has been created with `add_library`, `add_executable`, `add_custom_target` etc.

{% highlight cmake linenos %}
install(TARGETS bar
    DESTINATION bin
)
{% endhighlight %}

If you are creating an executable that depends on a shared lib , you might want to install the shared lib to the location of the executable.
To do so, you can use the `install(FILES...)` command as explained above, but as the file to copy, you can specify a target property of the shared lib which describes the path to the shared library file.

{% highlight cmake linenos %}
add_executable(bar
    src/main.cpp
)

target_link_libraries(bar
    PRIVATE
        foo
)

install(TARGETS bar
    DESTINATION bin
)

install(FILES 
    $<TARGET_FILE:foo>
    DESTINATION bin
)
{% endhighlight %}

Here, the executable `bar` depends on the shared lib `foo`.
First, the target `bar` is installed and then the `$<TARGET_FILE:foo>` generator expression is used to access the `TARGET_FILE` property of the target `foo`.
This property describes the path to the shared lib. Installing it this way will copy the `foo` library next to `bar`.

# Windows specifics with shared libraries
If you're trying to create an executable that depends on a shared library (dll) on Windows, you have to take care that you are specifying the dll's export / import interface correctly in your C++ code:

{% highlight c++ linenos %}
#ifdef _WIN32
    #ifdef BUILD_FOO
        #define SHARED_EXPORT __declspec(dllexport)
    #else
        #define SHARED_EXPORT __declspec(dllimport)
    #endif
#else
    #define SHARED_EXPORT
#endif

namespace foo
{
    std::string SHARED_EXPORT HelloWorld();
}
{% endhighlight %}

Here, the `__declspec(dllexport)` is used to publish the function `HelloWorld()` at the dll's export interface.
If you don't do this on Windows, the function will not be accessible from other executables or libraries.
Note we want to specify `dllexport` only when creating the lib. 
Users of the lib, that include the library's header would want to import the interface and thus the declaration of `HelloWorld()` has to be `__declspec(dllimport)`.
For that a compile-definition is used, which can also be set via CMake's `set_target_properties`:

{% highlight cmake linenos %}
set_target_properties(foo
    PROPERTIES
    POSITION_INDEPENDENT_CODE 1
    COMPILE_DEFINITIONS BUILD_FOO
)
{% endhighlight %}

This will set the preprocessor-define `BUILD_FOO` only for building the shared library. When building the executable, `BUILD_FOO` is not defined and `__declspec(dllimport)` is used instead.

# Code & Resources
The code for this post can be found on Github via the following tag:
[Installing](https://github.com/markusrothe/cmake_essentials/tree/Installing)

Also, checkout Daniel Pfeifer's [brilliant talk about modern CMake](https://www.youtube.com/watch?v=bsXLMQ6WgIk)