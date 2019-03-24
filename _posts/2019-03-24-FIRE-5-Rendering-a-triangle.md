---
layout: dark-post
title:  "(FIRE-5) Creating a window"
tags: [programming, FIRE, cpp, GL]
modified: 2019-03-24
categories: [FIRE]
excerpt_separator: <!-- more -->
---

Welcome to the fifth post about my rendering engine project **FIRE**!

Within this series I want to describe how I build up a rendering engine from scratch and my thoughts about it along the way.
All the code will be open source and will be hosted here: [FIRE](https://github.com/markusrothe/FIRE)

[Now that we have a window](https://www.markusrothe.dev/fire/2019/03/17/FIRE-4-Creating-a-window.html), we can start by drawing a simple 2D triangle into it.

That means we have to ...
1. ... create a `Mesh` class for the geometry of the triangle.
2. ... set up our first shader as part of the triangle's material.
3. ... upload the geometry (and some meta-data about it) to the graphics card
4. ... render the triangle

<!-- more -->
#### The client's example code

Here is what a client who uses FIRE might want to do: 

{% highlight c++ linedivs %}
{% raw %}
int main(int, char**){
    // ...
    FIRE::Mesh triangleMesh{"triangleMesh"};
    triangleMesh.AddVertices(
        {{-1.0f, -1.0f, 0.0f}, {1.0f, -1.0f, 0.0f}, {0.0f, 1.0f, 0.0f}});
    triangleMesh.AddIndices({0, 1, 2});
    triangleMesh.GetVertexDeclaration().AddSection("vPos", 3u, 0, 0);

    FIRE::Renderable triangle{"triangle"};
    triangle.SetMesh(std::move(triangleMesh));

    auto renderer{FIRE::GLFactory::CreateRenderer()};
    while(!window.ShouldClose())
    {
        window.PollEvents();
        renderer->Render(triangle);
        window.SwapBuffers();
    }
}
{% endraw %}
{% endhighlight %}

Firstly, he wants to set up a mesh, adding vertices to it.
Secondly, he wants to describe the geometry (which vertices belong to which triangle, are there only positions or also colors and normals associated with the mesh and so on...).
Then, the mesh gets wrapped in a `Renderable` which a `Renderer` can then take as a parameter order to render it.

In the following I'll go into the details of these steps and the involved classes. 
Everything is done with this first example in mind. Later, as the client's code gets more complicated, `FIRE`s code will likely grow more complicated, too.

#### The `Mesh` class
Our `Mesh` class will be a rather simple container for the triangle's geometry. 
All we need is a `std::vector<>` that holds some kind of position info:

{% highlight c++ linedivs %}
struct Mesh
{
    explicit Mesh(std::string name);
    
    // ...
    
private:
    std::string m_name;
    std::vector<Vertex> m_vertices;
    std::vector<unsigned int> m_indices;
    VertexDeclaration m_vertexDeclaration;
}
{% endhighlight %}

A `Vertex` is just a struct containing three float values (x, y, z).
Note that we also store a vector of indices in the mesh. 
This will describe how we have to interpret the vertices in terms of the triangles that make up the mesh.
Three consecutive indices will make up one triangle, meaning that in our client's example code above, the first vertex will also be the first one of the triangle.
If our example mesh was made out of more than one triangle we can reuse vertices that way instead of adding them to the mesh multiple times.
For instance, if we would draw a quad (two triangles) instead of one triangle, we could do:
{% highlight c++ linedivs %}
{% raw %}
triangleMesh.AddVertices(
    {{-1.0f, -1.0f, 0.0f}, 
    {1.0f, -1.0f, 0.0f}, 
    {1.0f, 1.0f, 0.0f},
    {-1.0f, 1.0f, 0.0f}});
triangleMesh.AddIndices({0, 1, 2, 2, 1, 3});
{% endraw %}
{% endhighlight %}

The first triangle is made up of the vertices with index `0, 1, 2` while the second triangle is made up of vertices with index `2, 1, 3`.
This format is also needed once we interact with OpenGL to upload the geometry.

Lastly, the `Mesh` also contains a `VertexDeclaration` member. 
This will become relevant once we are setting up our first shader.

#### Uploading the mesh
Now we'll interact with OpenGL for the first time! 
In order to draw the triangle, we have to tell OpenGL what it should render.
The way that is done is by setting up "Buffer Objects". 
Buffer objects store an array of unformatted memory which is allocated by the OpenGL context / the GPU. 

We want to store the triangle's vertex and index data inside buffer objects. 
A buffer object that stores vertices is called a **Vertex Buffer Object (VBO)** and the buffer object used for the indices is called an **Index Buffer** or **Index Buffer Object (IBO)**.

To start off, we first need to create and a **Vertex Array Object (VAO)**. 
A VAO is an OpenGL object that stores state (such as the VBO and IBO).

{% highlight c++ linedivs %}
GLuint vao;
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);
{% endhighlight %}

Once the VAO is bound, we can first create the VBO.
While the VAO is bound all other buffer objects we create and bind will be associated with that one bound VAO.
That means, always bind the VAO before binding a VBO / IBO!
(If we had a second renderable, we would create another VAO and other VBOs/IBOs for that renderable)

{% highlight c++ linedivs %}
GLuint vbo;
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
{% endhighlight %}

You can see the similarities between the creation and binding of a VAO and a VBO.
First, the identifier (a `GLuint`) is created using `glGen{Buffers/VertexArray}` and then it is bound via `glBind{Buffer/VertexArray}`.

Once the VBO is bound, we can write the vertices to it:

{% highlight c++ linedivs %}
auto const verticesSize = vertices.size() * sizeof(float);

glBufferData(
    GL_ARRAY_BUFFER,
    verticesSize,
    &(vertices[0]),
    GL_STATIC_DRAW);
{% endhighlight %}

`GL_ARRAY_BUFFER` identifies the buffer object as a VBO. We then pass in the **size** of our vertices, followed by a pointer to the start of our vertex data.
(Note that this is not the number of positions, but the size in bytes of the positions, so: `numPositions * 3 * sizeof(float) = 36`).
`GL_STATIC_DRAW` is a hint to OpenGL stating that the geometry will not be modified in the future.

Next up, we upload the index data in the same manner:
{% highlight c++ linedivs %}
GLuint ibo;
glGenBuffers(1, &ibo);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ibo);

auto const indicesSize = indices.size() * sizeof(unsigned int);
glBufferData(
    GL_ELEMENT_ARRAY_BUFFER,
    indicesSize,
    &(indices[0]),
    GL_STATIC_DRAW);
{% endhighlight %}

Here, we indentify the buffer object as an index buffer via `GL_ELEMENT_ARRAY_BUFFER`. That means, it will point into the other buffer (the VBO).
Once we are done, we "unbind" the buffers and the VAO:

{% highlight c++ linedivs %}
glBindBuffer(GL_ARRAY_BUFFER, 0);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
glBindVertexArray(0);
{% endhighlight %}

#### Rendering the triangle
Now that the data has been written to the GL context's memory, we need to bring OpenGL to draw it to the screen.
First, we'll set up a simple shader program consisting of a very simple **vertex shader** and a (also simple) **fragment shader**.
A vertex shader is a small little program that will be executed for every vertex that we have uploaded (hence the name).
It will produce output that will then be the input for the fragment shader. 
A fragment shader is another (also small) little program that will be executed for every "fragment". 
You can think of a fragment as a screen's pixel. 
However, it is possible that one pixel of your screen will cover multiple fragments
(If there are multiple elements/triangles that can be seen through that one pixel. 
Most of the time only the foremost fragment will be processed because that is the only one that is visible through the pixel. 
All others might get discarded.)
We are going to write the shaders in `GLSL` (the GL shading language). Here is our vertex shader:
{% highlight c++ linedivs %}
layout(location = 0) in vec3 vPos;
void main() { 
    gl_Position = vec4(vPos.xyz, 1.0); 
}
{% endhighlight %}

First, we'll define the input to our shader. 
As we only have positions to process we only have one input, a 3D vector called `vPos`.
Note that we also define a "layout" for that input. 
This is relevant in combination with the `VertexDeclaration` of our `Mesh` class, but let's get to that in a moment.

{% highlight c++ linedivs %}
layout(location = 0) in vec3 vPos;
{% endhighlight %}

Next, we write the main function to our vertex shader program.

{% highlight c++ linedivs %}
void main() { 
    gl_Position = vec4(vPos.xyz, 1.0); 
}
{% endhighlight %}

`gl_Position` is the output variable of our vertex shader. 
Because we do not want to modify the input position in any way, we will just pass it through and directly assign vPos to `gl_Position`.

Now, the vertex will be processed by the OpenGL graphics pipeline, broken into fragments and passed to the fragment shader:
{% highlight c++ linedivs %}
out vec4 color;
void main() { 
    color = vec4(1.0, 1.0, 1.0, 1.0); 
}
{% endhighlight %}

We first define the output for the fragment shader, a `color` value consisting of 4 floats (red, green, blue, alpha).
To keep it simple, we just set everything to 1.0, which means the color of every rendered fragment will be white.
Before we can make use of the shaders, we have to compile them and link them together. 
For details about that, take a look at [this code](https://github.com/markusrothe/FIRE/blob/FIRE-5/FIRE/src/GLShaderFactory.cpp).
Once compiled and linked, we will get an integer that represents this shader program (consisting of both, the vertex shader and the fragment shader).

All that is left is to make the connection between our uploaded data and our newly created shaders.
This is where the `VertexDeclaration` and the `layout` inside the vertex shader come in.
We have to tell OpenGL what to do with the uploaded vertices. 
For that, we will make use of the GL function `glVertexAttribPointer`:

{% highlight c++ linedivs %}
void SpecifyVertexAttributes(VertexDeclaration const& vDecl, GLuint shader)
{
    for(auto const& vertexDeclSection : vDecl.GetSections())
    {
        auto attribLocation = glGetAttribLocation(
            shader,
            vertexDeclSection.first.c_str());

        glEnableVertexAttribArray(attribLocation);
        glVertexAttribPointer(
            attribLocation,
            vertexDeclSection.second.size,
            GL_FLOAT,
            GL_FALSE,
            vertexDeclSection.second.stride,
            BUFFER_OFFSET(vertexDeclSection.second.offset));
    }
}
{% endhighlight %}

Our `VertexDeclaration` will contain the name of the shader attributes that we are going to use. 
In the client's example code, he specified an attribute called "vPos" along with its size (3u means 3 float values).
{% highlight c++ linedivs %}
// ...
triangleMesh.GetVertexDeclaration().AddSection("vPos", 3u, 0, 0);
// ...
{% endhighlight %}
We have used that same attribute name inside our shader. 
First, we want to know the "location" that GL uses for the "vPos" attribute. 
We will get that via `glGetAttribLocation`. 
This function takes both the int identifying our linked shader program (see above) and the name of the attribute that it should look up inside that shader program and returns another integer as a "handle" to that location.
We can pass that handle into `glVertexAttribPointer` to tell GL what it should input into the "vPos" parameter of our shader.
(Note that for this reason, the VBO will still be bound while we call `glVertexAttribPointer`!)
So, along with some parameters that are not yet relevant to us, we specify the `attribLocation` ("vPos"), the `size` (3 floats) and the type of the values (GL_FLOAT).

Whenever we want to draw the triangle, GL will "iterate through the VBO" and put 3 floats into the "vPos" parameter of our shader.

All that is left to do is to actually draw everything:

{% highlight c++ linedivs %}
glBindVertexArray(std::get<0>(buffers));
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, std::get<2>(buffers));
glDrawElements(
    GL_TRIANGLES, renderable.GetMesh().Indices().size(), GL_UNSIGNED_INT,
0);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
glBindVertexArray(0);
{% endhighlight %}

First, we will bind the VAO and the IBO of the triangle. 
Then we'll call `glDrawElements()` which finally puts something onto the screen.
Finally, we clean up after ourselves by "un-binding" the VAO and the IBO in reverse order.

If we run the example, we'll end up with this screen:

<figure>
	<img src="/images/FIRE-5-Triangle.PNG" alt="">
	<figcaption>Our first triangle</figcaption>
</figure>

This is it for now. 
Next time, we'll take a look at color values for our triangle's vertices and start to work on bringing everything into 3D.

(You can check out the code for this post via [this git tag](https://github.com/markusrothe/FIRE/tree/FIRE-5))
