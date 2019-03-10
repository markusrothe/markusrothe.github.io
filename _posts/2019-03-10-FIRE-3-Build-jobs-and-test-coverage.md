---
layout: dark-post
title:  "(FIRE-3) Build jobs and test coverage"
tags: [programming, FIRE, CMake, cpp, travis ci, coveralls]
modified: 2019-03-10
categories: [FIRE]
excerpt_separator: <!-- more -->
---

Welcome to the third post about my rendering engine project **FIRE**!

Within this series I want to describe how I build up a rendering engine from scratch and my thoughts about it along the way.
All the code will be open source and will be hosted here: [FIRE](https://github.com/markusrothe/FIRE)

In the [previous post](https://www.markusrothe.dev/fire/2019/03/06/FIRE-2-Adding-test-support.html), 
I've added unit testing support to FIRE using the googletest C++ testing framework and configured a CMake custom post-build job to execute the unit tests after every build.

This time, I want to set up an automatic build job that compiles FIRE with different compilers (clang, gcc, msvc) and runs the unit tests on those platforms.
In addition, I want to be able to see to what degree FIRE's code is tested, so I need to measure the code coverage within the build job, too.

<!-- more -->

### Continuous Integration with Travis CI

I am going to use [Travis CI](https://travis-ci.org/) for my automatic build job as it fulfills all of my requirements that I have on continous integration / automatic build jobs:

1. It integrates seamlessly with GitHub, where FIRE is hosted, too.
2. It is free for open source projects.
3. Since October 2018, Travis CI offers Windows as a build platform next to Linux and MacOs.
4. It is highly configurable, allowing me to use current versions of clang, gcc and msvc.
5. There is a lot of documentation for Travis, both on its official website and on stackoverflow.

Before we can use Travis to build and test our code, FIRE's repository needs to be connected with Travis. 
I won't go into detail about how to set this up here as [Travis' tutorials](https://docs.travis-ci.com/user/tutorial/#to-get-started-with-travis-ci) cover that in detail already.

Once the repository is connected, we can configure our build job using a file called `.travis.yml` that we put into the root directory of FIRE's repository.

Here is my `.travis.yml`.

{% highlight yml linedivs %}
dist: xenial
language: cpp

matrix:
    include:
        - os: linux
          compiler: cpp
        - os: linux
          compiler: clang
        - os: windows
    
before_install:

    - if [ "$TRAVIS_OS_NAME" = "linux" ]; then sudo add-apt-repository ppa:ubuntu-toolchain-r/test -y; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ]; then sudo apt-get update -q; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ]; then sudo apt-get install -qq; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then pip install --user cpp-coveralls; fi
    
install:
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then sudo apt-get install -qq g++-7; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then export CXX="g++-7" CC="gcc-7";fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-7 90; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 90; fi
    
script:
    - mkdir build
    - cd build
    - cmake -DCOVERAGE=1 .. 
    - cmake --build .
    
after_success:
    - coveralls --root .. -E ".*CMakeFiles.*" -E ".*googletest.*" -E ".*googlemock.*" 
{% endhighlight %} 

Let's go through it step by step:

{% highlight yml %}
dist: xenial
language: cpp
{% endhighlight %} 
First, we specify that we want to use the xenial distribution.
This build environment already comes with git, cmake 3.12 and clang-7, so we do not need to install any of these manually.
Unfortunately, the gcc version is 5.4 but I prefer gcc-7, so we need to install that ourselves during the build (more on that below).
And, because we want to build C++-code, we specifiy the language cpp.

Next up, we define a build matrix:
{% highlight yml %}
matrix:
    include:
        - os: linux
          compiler: cpp
        - os: linux
          compiler: clang
        - os: windows
{% endhighlight %} 

This matrix specifies that, on Linux, we want to build with clang and gcc ('cpp' is an alias for 'gcc' here) and with the default compiler on Windows (VS 2017).
Thus, in Travis, we will end up with three different build jobs (which will be executed in parallel).

The next step is to install everything that our build jobs might need:
{% highlight yml %}
before_install:

    - if [ "$TRAVIS_OS_NAME" = "linux" ]; then sudo add-apt-repository ppa:ubuntu-toolchain-r/test -y; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ]; then sudo apt-get update -q; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ]; then sudo apt-get install -qq; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then pip install --user cpp-coveralls; fi
install:
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then sudo apt-get install -qq g++-7; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then export CXX="g++-7" CC="gcc-7";fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-7 90; fi
    - if [ "$TRAVIS_OS_NAME" = "linux" ] && [ "$CXX" = "g++" ]; then sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 90; fi
{% endhighlight %} 

We add the `ppa:ubuntu-toolchain-r/test` repository in order to be able to download packages from it.
After updating packages and installing, we'll install `gcc-7` and `cpp-coveralls` (our tool of choice for measuring code coverage - see below).
Note that we also export the c/c++ compiler environment variables and let the default gcc be gcc-7.
(xenial comes with `gcc-5.4` as the default, but we want to override that with `gcc-7` for our builds). 
Also note that we only install `coveralls` if we're building on linux with gcc. 
There is no need to measure code coverage on every platform/build.

Afterwards, we do the same calls to cmake that we would do locally to build FIRE.
(Travis puts us into the root directory of our clone repository in the beginning, so creating the build directory is done from within FIRE's root).
{% highlight yml %}
script:
    - mkdir build
    - cd build
    - cmake -DCOVERAGE=1 .. 
    - cmake --build .
{% endhighlight %} 

In the first call to cmake, we provide a new parameter `-DCOVERAGE=1`.
We will use that parameter within cmake later to decide whether we want to measure code coverage within the build or not.

If the build and the unit tests were successful, we call coveralls to upload the coverage measurements to the [coveralls website](https://coveralls.io/):
{% highlight yml %}
after_success:
    - coveralls --root .. -E ".*CMakeFiles.*" -E ".*googletest.*" -E ".*googlemock.*" 
{% endhighlight %}

After committing and pushing the new `.travis.yml`, Travis will run and (on the Travis website) we will be presented with an overview of our build(s):
<figure>
	<img src="/images/travis_ci.png" alt="">
	<figcaption>FIRE's Travis CI overview</figcaption>
</figure>
We can see all three of our builds (gcc, clang and msvc) including their status and build times. 
If a build breaks, we will get notified via mail by Travis, too.

### Code Coverage with Coveralls

In the previous section we have already seen how [Coveralls](https://coveralls.io/) is installed on the travis build slave(s).
Measuring code coverage means that we want to measure how much of FIRE's code is actually tested. Obviously, we want to aim for a high number (>90%) of code coverage.
(100% is an utopian fantasy that we will likely never reach once FIRE's code gets a bit more complex. Especially as it is quite hard to test rendering code with unit tests).

Similar to Travis CI, Coveralls integrates seamlessly with GitHub and is free for open source repositories.
After setting up the coverage job for the FIRE repository (following [Coveralls Documentation](https://docs.coveralls.io/)), we need to change our CMake scripts to be able to actually measure something:

That is where the `-DCOVERAGE=1` option (which we set previously in the `.travis.yml` file) comes in.
We'll add the following cmake code to `FIRE`'s CMakeLists.txt below `FIRE`'s target definition:

{% highlight cmake linedivs %}
# FIRE/CMakeLists.txt
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" AND COVERAGE)
    target_compile_options(FIRE PUBLIC --coverage)
    target_link_libraries(FIRE PUBLIC --coverage)
endif()
{% endhighlight %}

If the compiler that we are compiling with is gcc, we want to add `--coverage` as a compiler flag and as a linker flag.
Internally, that boils down to using Gnu's [gcov](https://gcc.gnu.org/onlinedocs/gcc/Gcov-Intro.html#Gcov-Intro) tool to profile the code.
Measurements include how often each line of code executes, which lines are executed and which ones are not and how long everything actually took.

Having this set up, a build job that is completed by travis will upload the coverage measurements to coveralls, culminating in this overall view:

<figure>
	<img src="/images/coveralls.png" alt="">
	<figcaption>FIRE's Coveralls overview</figcaption>
</figure>

We can see the measurements of the latest builds that have been completed.
At the bottom we can browse through the file tree to see which lines of code got executed with by our unit tests.

We can also look into each file and see line by line if and how often it ran:

<figure>
	<img src="/images/coveralls-file.png" alt="">
	<figcaption>Coveralls overview for FIRE.cpp</figcaption>
</figure>

With unit tests, a build job for multiple platforms and coverage measurements in place, we can start with actually programming something inside FIRE.
At some point (once FIRE has got a bit more "meat" to it) I also want to cover packaging with CMake. But before that, we need to actually have something that is worth packaging.

Over the course of the next few posts, I want to cover the implementation of a simple rendering use-case: Rendering a colored 3D cube. 
This will involve setting up a render window with OpenGL support, the cube's geometry and material and a camera to view it in 3D.

(You can check out the code for this post via [this git tag](https://github.com/markusrothe/FIRE/tree/FIRE-3))