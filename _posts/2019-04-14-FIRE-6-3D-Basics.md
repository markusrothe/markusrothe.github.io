---
layout: dark-post
title:  "(FIRE-6) 3D Basics"
tags: [programming, FIRE, cpp, GL]
modified: 2019-04-15
categories: [FIRE]
excerpt_separator: <!-- more -->
---

Welcome to the sixth post about my rendering engine project **FIRE**!

Within this series I want to describe how I build up a rendering engine from scratch and my thoughts about it along the way.
All the code will be open source and will be hosted here: [FIRE](https://github.com/markusrothe/FIRE)

[Last time](https://www.markusrothe.dev/fire/2019/03/24/FIRE-5-Rendering-a-triangle.html) we rendered our first 2D triangle. 
We've set up a very basic shader as well as the OpenGL buffer objects that hold the triangle's geometry.

This time we'll add the most basic prerequisites for rendering a 3D cube.
First, we need to create the cube's mesh. 
Second, we'll have to create a camera class that will view the cube from an arbitrary position. 
Finally, we have to adapt our existing shader to effectively bring our 3D cube onto a 2D screen via a perspective projection. 
Here is the result that we're going for:

<figure>
	<img src="/images/FIRE-6-Cube.PNG" alt="">
</figure>

<!-- more -->

#### Creating the cube mesh

The cube mesh will consist of 8 vertices connected via 12 triangles. Here is a sketch:

<figure>
	<img src="/images/FIRE-6-CubeMesh.png" alt="">
</figure>

Fortunately, **FIRE** already provides everything we need to create the 3D mesh for our cube. 
We only have to change the client's code according to the sketch:

{% highlight c++ linedivs %}
{% raw %}
FIRE::Mesh cubeMesh{"cubeMesh"}; 

cubeMesh.AddVertices({{-1.0f, -1.0f, 1.0f},
                      {1.0f, -1.0f, 1.0f},
                      {1.0f, -1.0f, -1.0f},
                      {-1.0f, -1.0f, -1.0f},
                      {-1.0f, 1.0f, 1.0f},
                      {1.0f, 1.0f, 1.0f},
                      {1.0f, 1.0f, -1.0f},
                      {-1.0f, 1.0f, -1.0f}});
cubeMesh.AddIndices({0, 1, 5, 0, 5, 4,
                     1, 2, 6, 1, 6, 5,
                     2, 3, 7, 2, 7, 6,
                     3, 0, 4, 3, 4, 7,
                     4, 5, 6, 4, 6, 7,
                     2, 1, 3, 3, 1, 0});
{% endraw %}
{% endhighlight %}

Our cube has 12 triangles which means that we need 36 indices to fully describe them. The order of the indices is important as this decides whether our triangles are front-facing or back-facing. 

In OpenGL the default setting is for triangles to be front-facing when they're orientated counterclockwise:

<figure>
	<img src="/images/FIRE-6-CubeIndices.png" alt="">
</figure>

Looking at the sketch above, we can easily figure out all the indices for all sides of the cube.
We could have also implemented the generation of the indices in a more algorithmic fashion, but for now the hardcoded indices suffice.

#### Setting up a camera

Next up, we have to think about from what position we want to look at our cube. 
If we were to render the cube without any kind of camera or perspective projection we would only see the screen completely filled with the cube's color. 

Before starting we have to introduce the notion of world space vs. camera space vs. screen space:

In general, a mesh will first be modeled in its own coordinate system. 
Every vertex of the mesh is relative to the model's origin (0,0,0).
This is called the model space.
Once we load the mesh into our engine (or rather our scene), the mesh will be positioned relative to the 3D world's origin instead. 
Every mesh (and later a camera or light sources) will be positioned in 3D space relative to the 3D world's origin.
This is called "world space". Our cube currently is positioned at the origin (0, 0, 0). This is because we did not any code that "moves" the cube somewhere else, so the cube's "model space origin" matches the 3D world's origin directly. 
If we were to move the cube 10 units in x-direction, some more units in z-direction and then rotate the cube, the vertices would still be the same [(1,1,1) - (-1,-1,-1)] within model space, but in world space the vertices would be somewhere completely different.

<figure>
	<img src="/images/FIRE-6-WorldSpace.png" alt="">
</figure>

Now let's add a camera into the mix, too. When viewing the world from a camera's perspective, that creates yet another coordinate system: the "camera space" or "view space". In this coordinate system, the camera's position is the origin and everything else like our cube will be relative to the camera's position / origin.

Once we know where everything (meshes, etc.) is located relative to our camera and we know where our camera is relative to the 3D world's origin, we still have to bring a 3D world onto a 2D screen. 
This is done via a perspective projection.
Imagine that a camera has a frame in front of it that it is looking through and let's view it from the side:

<figure>
	<img src="/images/FIRE-6-Camera.PNG" alt="">
</figure>

That frame (red in the picture) is our 2D screen. 
A projection of our 3D scene onto the screen can be imagined by firing a ray from the camera's position through the screen into the scene. 
The 2D position where the ray intersects the screen is the position that a 3D object behind the screen will occupy on the screen.
Now imagine further that we fire a ray through every position on the screen. (Such a screen position is a pixel.)

This kind of projection can be done with some mathematics and clever matrix calculations. Among other parameters, it is paramterized by the field of view angle of our camera (fovy) as well as the aspect ratio of the screen.
(I'll cover more about the perspective projection in another post.)
Everything on the screen we consider as being in "screen space".

Here is the rundown of everything: First, we have to take our mesh and place that in 3D world space, then, we will have to look at the mesh from our camera's point of view which is another coordinate system transformation and lastly, we have to project everything onto the 2D screen via the perpective projection which is yet another coordinate system change. For every coordinate system, we create a matrix that describes one coordinate system change, leaving us with three matrices: 
* **Mm** is the matrix that takes our mesh from model space into world space.
* **Mv** is the matrix that takes our mesh from world space into view space.
* **Mp** is the matrix that takes our mesh from view space into screen space.

If we now take a vertex from our initial cube mesh (for instance v0=(-1, -1, -1)) and multiply all these matrices with that vertex, we will end up with a position of that vertex on the 2D screen (provided the vertex can be seen by our camera, of course)

**v_screen = Mp * Mv * Mm * v0**

Note that we multiple from the right and in reverse order due to the rules for matrix multiplication.

#### Passing the model-/view-/perspective matrices to the shader

Now that we know how to get a 3D position onto our 2D screen, we have to adapt our current shaders to take the different matrices described above and do the calculation **v_screen = Mp * Mv * Mm * v** for every vertex.

In contrast to the vertex which is always different for every shader invocation, the matrices stay the same for every vertex. 
So we need to pass matrices to a shader "uniformly" for every vertex.

This is done by declaring a uniform variable in our vertex shader. 
We only need one variable as we can do the matrix multiplication once inside the C++ code and pass the resulting matrix to the shader.
Otherwise we would do the same multiplication of these three matrices multiple times redundantly.
We'll call the resulting matrix the "ModelViewProjection-Matrix (MVP)"

{% highlight c++ linedivs %}
layout(location = 0) in vec3 vPos;
uniform mat4 MVP;
void main() { 
    gl_Position = MVP * vec4(vPos.xyz, 1.0);
};
{% endhighlight %}

Once we declared this variable, we have to pass the MVP matrix to the shader from our C++ code before drawing:

{% highlight c++ linedivs %}
auto const shader = m_materialManager->GetShader(renderable.GetMaterial());
glUseProgram(shader);
auto const uniformLocation = glGetUniformLocation(shader, "MVP");
glUniformMatrix4fv(uniformLocation, 1, GL_FALSE, <<matrix data>>);
glDrawElements(
        GL_TRIANGLES, static_cast<GLsizei>(renderable.GetMesh().Indices().size()), GL_UNSIGNED_INT, 0);
glUseProgram(0);
{% endhighlight %}

First, we'll bind the shader via `glUseProgram(<shaderID>)`. Once bound, every uniform we try to set will be sent to the bound shader.
Next, we need to know the location that the uniform variable will be bound to based on the variable's name. We can find that via `glGetUniformLocation(<shaderID>, <uniformName>)`.
Then, we upload the matrix data to the shader using `glUniformMatrix4fv(...)` with the previously determined uniform location before drawing and un-binding the shader.

And that's it! If we start our executable now we will see a 3D cube that has been correctly projected onto our screen. As a bonus (and to see whether we did everything truly correctly) we can change our fragment shader to take the vertex positions as the vertex' colors, too. We will end up with a colored cube as seen in the first screenshot.

Next time, we will do some refactorings of our code as we do have some duplications that we need to take care of. For instance, we do have two different "entities" that have an individual position, our cube and the camera. If we added light sources or other meshes, and if we want to move them around we'll have to think about a generalized way of handling positions (or orientation) and we also have to be able to change them during runtime.

(You can check out the code for this post via [this git tag](https://github.com/markusrothe/FIRE/tree/FIRE-6))
