---
layout: dark-post
title:  "(FIRE-4) Creating a window"
tags: [programming, FIRE, CMake, cpp, GL]
modified: 2019-03-17
categories: [FIRE]
excerpt_separator: <!-- more -->
---

Welcome to the fourth post about my rendering engine project **FIRE**!

Within this series I want to describe how I build up a rendering engine from scratch and my thoughts about it along the way.
All the code will be open source and will be hosted here: [FIRE](https://github.com/markusrothe/FIRE)

[Last time](https://www.markusrothe.dev/fire/2019/03/10/FIRE-3-Build-jobs-and-test-coverage.html)
I've created a build job to build with gcc, clang and msvc using Travis CI as well as code coverage measurements via gcov and coveralls.

Now, we can start with actually coding something. As a first step, we need a window that can display everything we render.
So we're taking a look at how to create (and close) a window using [GLFW](https://www.glfw.org/), an open-source, multi-platform library that provides an API for the creation of windows.

<!-- more -->
#### Pulling in GLFW as a dependency

GLFW is another third-party dependency that we have to pull in, similar to gtest.
I've followed the same approach that I used for gtest (which I described [in this post](https://www.markusrothe.dev/fire/2019/03/06/FIRE-2-Adding-test-support.html)).
That means, during configuration of our cmake project, the GLFW git repository will be cloned and added as a subdirectory to our own project.
(Fortunately, GLFW is also a CMake-Project, which makes it easy to pull it in for us)

#### Linking against GLFW
Once we know the GLFW library targets, we can use them to link against GLFW inside FIRE's own CMakeLists.txt:

{% highlight cmake linedivs %}
# FIRE/CMakeLists.txt
target_link_libraries(FIRE
PRIVATE
    glfw
)
{% endhighlight %}
That is all there is to it! Now we can make use of all the headers that make up the public interface of GLFW.
`target_link_libraries()` already configures the include paths needed for compiling and linking `FIRE`.
Note that we link against GLFW PRIVATEly. This is an important detail: We do not want to expose any third-party dependencies to users of our library and we will go to great lengths in order to ensure that throughout the development of FIRE. Effectively, this means that no public header of FIRE must include a header of a third-party library.
It also forces us to think about correctly wrapping away third-party dependencies inside FIRE.

#### Implementing the `Window` class

This is what we need from a `Window` class in the beginning:

1. The window has a title.
2. The window shall have a width and a height and shall be resizable.
3. The window can be closed and we can check whether is has been closed.
4. The window supports [double buffering](https://en.wikipedia.org/wiki/Multiple_buffering#Double_buffering_in_computer_graphics)

So let's create a bunch of test cases to verify these criteria:

{% highlight c++ linedivs %}
TEST_F(WindowTest, HasATitle)
{
    EXPECT_EQ(title, window.GetTitle());
}

TEST_F(WindowTest, HasAWidth)
{
    EXPECT_EQ(width, window.GetWidth());
}

TEST_F(WindowTest, HasAHeight)
{
    EXPECT_EQ(height, window.GetHeight());
}
{% endhighlight %}

These are straightforward. Once we have written the tests and we see that they fail, we implement a constructor and a bunch of getters in a new `Window` class:

{% highlight c++ linedivs %}
class Window
{
public:
    explicit Window(
        std::string const title, unsigned int width, unsigned int height);
        
    std::string GetTitle() const;
    unsigned int GetWidth() const;
    unsigned int GetHeight() const;
    //...
private:
    std::string const m_title;
    unsigned int m_width;
    unsigned int m_height;
};
{% endhighlight %}

The tests will run, but this does not yet make use of GLFW and it for sure does not create a window that we can drag around, resize and close. 
We will need a full working example for this, so here is the `main()`-function of our example executable that we created in one of the previous posts:

{% highlight c++ linedivs %}
int main(int, char**)
{
    FIRE::Window window{"example1", 800, 600};

    auto context{std::make_unique<FIRE::GLRenderContext>(window)};
    window.SetRenderContext(std::move(context));

    while(!window.ShouldClose())
    {
        window.SwapBuffers();
        window.PollEvents();
    }
}
{% endhighlight %}

First, we create a window with a title, a width and a height.
Then we'll create and set a render context that we want to use.
In a loop we periodically check whether someone has closed the window. 
If not, we'll swap buffers (more about this in a later post) and we poll the window for any events (such as a request to close the window).

(When implementing FIRE I will use a mix of automated unit tests and example executables like this. 
In my opinion it is really hard to fully unit-test rendering applications, especially if there is a need for checking graphical output.
So a mixture of unit tests, correct abstractions/interfaces and example executables for the real rendering code is my way to go about it.)

In the example code above, you see this ominous `FIRE::GLRenderContext` class we create.
This is what we will be using to wrap away the dependency to the GLFW third-party library.
Let us write some more test cases to make this more clear. 
First we'll start with the check whether the window was closed. We'll also add a test-fixture that encapsulates all the code which would otherwise be repeated across our test cases:

{% highlight c++ linedivs %}
namespace FIRE
{

class RenderContextMock : public RenderContext
{
public:
    // ... 
    MOCK_METHOD0(ShouldClose, bool());
    MOCK_METHOD0(Close, void(void));
    // ...
};
} // namespace FIRE
class WindowTest : public ::testing::Test
{
public:
    WindowTest()
        : context(std::make_unique<FIRE::RenderContextMock>())
    {
    }
    FIRE::Window window{title, width, height};
    std::unique_ptr<FIRE::RenderContextMock> context;
};

// ...

TEST_F(WindowTest, CanBeClosed)
{
    EXPECT_CALL(*context, ShouldClose())
        .WillOnce(Return(false))
        .WillOnce(Return(true));

    window.SetRenderContext(std::move(context));

    EXPECT_FALSE(window.ShouldClose());
    window.Close();
    EXPECT_TRUE(window.ShouldClose());
}
{% endhighlight %}

This is what our test case checks: We expect that the window should not close after creating it. 
Once the user calls `window.Close()`, any further call to `window.ShouldClose()` should now return true.
We are making use of an interface `RenderContext` that we mock via gmock:

{% highlight c++ linedivs %}
    EXPECT_CALL(*context, ShouldClose())
        .WillOnce(Return(false))
        .WillOnce(Return(true));
{% endhighlight %}

This line sets an expectation on our mock object. We expect that the function `ShouldClose()` will be called twice.
In the first call, we configure it to return `false`. In the second call we make it return `true`.
Note that we do two things here: We *expect* the function to be called twice, but we *configure* it to do something when it is called.
The fact that it returns true or false is not a check that we assert on, but something that we want the mock to do.

Next, we inject the mock object into the window class:

{% highlight c++ linedivs %}
    window.SetRenderContext(std::move(context));
{% endhighlight %}

This is the reason why we use an interface. Inside our unit test, we can easily inject a mock object that has nothing to do with GLFW.
But we can still test whether our window class behaves correctly. That is, when we call `window.ShouldClose()` it has to forward that call to its RenderContext.
Here are parts of the interface of the RenderContext:

{% highlight c++ linedivs %}
class RenderContext
{
public:
    virtual ~RenderContext() = default;
    // ...
    virtual bool ShouldClose() = 0;
    virtual void Close() = 0;
};
{% endhighlight %}

Once we have done the same thing for other functions (`SwapBuffers()`, `PollEvents()`, `Resize()` - Check out [this tag](https://github.com/markusrothe/FIRE/tree/FIRE-4)),
we can think about the real implementation of a RenderContext that makes use of GLFW.

#### Using GLFW

Let's take a look at the `GLRenderContext` class we used inside our example executable. 
(I've left out a bunch of functions to keep it simple.)
{% highlight c++ linedivs %}
class GLRenderContext : public RenderContext
{
public:
    explicit GLRenderContext(Window& window);
    ~GLRenderContext() override;

    //...
    bool ShouldClose() override;
    void Close() override;


private:
    class Impl;
    std::unique_ptr<Impl> m_impl;
};
{% endhighlight %}

Remember that we did not want to expose the dependency to GLFW to users of `FIRE`?
The [Pimpl-Idiom](https://en.cppreference.com/w/cpp/language/pimpl) is one way to do that. 
{% highlight c++ linedivs %}
private:
    class Impl;
    std::unique_ptr<Impl> m_impl;
{% endhighlight %}

We forward-declare a class `Impl` and let the `GLRenderContext` class own a pointer to it.
Inside the `GLRenderContext.cpp` file we will implement the `Impl` class, keeping it out of our public header that way:

{% highlight c++ linedivs %}
#include <GLFW/glfw3.h>
class GLRenderContext::Impl
{
public:
    explicit Impl(Window& window);
    ~Impl();
    bool ShouldClose();
    void Close();
private:
    GLFWwindow* m_window;
};
GLRenderContext::Impl::Impl(Window& window)
{
    if(!glfwInit())
    {
        std::cerr << "glfwInit failed\n";
        std::exit(EXIT_FAILURE);
    }

    m_window = glfwCreateWindow(
        window.GetWidth(), window.GetHeight(), window.GetTitle().c_str(), NULL,
        NULL);
    // ...
    glfwMakeContextCurrent(m_window);
}

GLRenderContext::Impl::~Impl()
{
    glfwDestroyWindow(m_window);
}

bool GLRenderContext::Impl::ShouldClose()
{
    return glfwWindowShouldClose(m_window);
}

void GLRenderContext::Impl::Close()
{
    glfwSetWindowShouldClose(m_window, GLFW_TRUE);
}

GLRenderContext::GLRenderContext(Window& window)
    : m_impl(std::make_unique<GLRenderContext::Impl>(window))
{
}

GLRenderContext::~GLRenderContext() = default;

bool GLRenderContext::ShouldClose()
{
    return m_impl->ShouldClose();
}

void GLRenderContext::Close()
{
    m_impl->Close();
}
{% endhighlight %}

We need to create an instance of the `Impl` class in our original `GLRenderContext`s constructor:
{% highlight c++ linedivs %}
GLRenderContext::GLRenderContext(Window& window)
    : m_impl(std::make_unique<GLRenderContext::Impl>(window))
    //...
{% endhighlight %}

Once we have that, all the calls to `GLRenderContext` will just be forwarded to `Impl`:

{% highlight c++ linedivs %}
bool GLRenderContext::ShouldClose()
{
    return m_impl->ShouldClose();
}
{% endhighlight %}

`Impl` contains the actual implementation in terms of GLFW:

{% highlight c++ linedivs %}
bool GLRenderContext::Impl::ShouldClose()
{
    return glfwWindowShouldClose(m_window);
}
{% endhighlight %}

Similarly, we can initialize GLFW and create the actual window inside `Impl`s constructor.
(For a more detailed overview of GLFW, take a look at [its documentation](https://www.glfw.org/documentation.html))
{% highlight c++ linedivs %}
GLRenderContext::Impl::Impl(Window& window)
{
    if(!glfwInit())
    {
        std::cerr << "glfwInit failed\n";
        std::exit(EXIT_FAILURE);
    }
    m_window = glfwCreateWindow(
        window.GetWidth(), window.GetHeight(), window.GetTitle().c_str(), NULL,
        NULL);

    // ..
    glfwMakeContextCurrent(m_window);
}
{% endhighlight %}

And that's it. Now we can add a bunch of other functions in the same manner (after writing test cases using the RenderContext mock, of course). Take a look at the source code for their details. In essence, they follow the same principles that we've seen above, though.

If we start the example executable, we will get a window that can be dragged around and closed:

<figure>
	<img src="/images/window.PNG" alt="">
	<figcaption>Our window</figcaption>
</figure>

Next time, we'll start drawing something into that window. Right now there is an awful lot of black going on there...

(You can check out the code for this post via [this git tag](https://github.com/markusrothe/FIRE/tree/FIRE-4))
