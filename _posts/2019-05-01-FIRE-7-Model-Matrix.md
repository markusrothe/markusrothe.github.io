---
layout: dark-post
title:  "(FIRE-7) Scenes & The Model Matrix"
tags: [programming, FIRE, cpp, GL]
modified: 2019-05-01
categories: [FIRE]
usemathjax: true
excerpt_separator: <!-- more -->
---

Welcome to the 7th post about my rendering engine project **FIRE**!

Within this series I want to describe how I build up a rendering engine from scratch and my thoughts about it along the way.
All the code will be open source and will be hosted here: [FIRE](https://github.com/markusrothe/FIRE)

[In the last post](https://www.markusrothe.dev/fire/2019/04/14/FIRE-6-3D-Basics.html) we made the transition from 2D to 3D by setting up a camera and performing a perspective projection of our 3D scene onto a 2D screen. 
We've also described the different coordinate systems that we have to think about in a 3D scene, namely model space, world space, view space and screen space.

This time, we will have to think how we want to start setting up our engine in terms of its software architecture. 
Keep in mind it is still quite early in `FIRE`s development and we cannot describe every single detail up front (we even would not want to design everything in the beginning as things might change quite drastically in the future, anyway).
However, we can start thinking about the direction that we want to go with `FIRE`s architecture.

<!-- more -->

#### Scenes, SceneComponents & Renderables
Here is a basic sketch of how we want to organize a Scene.

{:refdef: style="text-align: center;"}
<figure>
	<img src="/images/FIRE-7-classes.svg" alt="">
</figure>
{:refdef}

A `Scene` contains up to n `SceneComponents`, which again contain up to n `Renderables`.
Each `Renderable` aggregates an instance of `Transform`, `Mesh` and `Material`.
The `Renderer` then can take a scene and render everything in it.

We've covered the `Mesh` class before in a [previous post](https://www.markusrothe.dev/fire/2019/03/24/FIRE-5-Rendering-a-triangle.html).
The `Material` is related to our shader setup and we will have more posts about this in the future.
For this post, let's take a look at the `Transform` class:

#### Positioning Renderables in a Scene
**FIRE** needs to offer a way for users to place object in the scene freely and to modify an object's position during runtime.
(Otherwise, a user would only see still images which, in a game, could get boring quite fast). 

That's where geometric transformations come in. 
In our case, we want to [translate](https://en.wikipedia.org/wiki/Translation_(geometry)) objects within the 3D world space.
(We'll cover rotation, scaling etc. once we need that, too).

A translation is an affine transformation 

$$y = Ax + t$$

The matrix $A$ is a linear transformation matrix (that, for example, scales or rotates $x$) followed by a translation $t$.
That means if we wanted to simply move the point $x$ by 2 units in the y-direction and by 3 units in z-direction, we'd use this formula:

$$y = \begin{bmatrix}
1 & 0 & 0\\
0 & 1 & 0\\
0 & 0 & 1\\
\end{bmatrix}x + \begin{bmatrix}
0 \\
2 \\
3 \\
\end{bmatrix}$$

We can use a mathematical "trick" to express this whole formula (the linear transformation and a translation) as one single matrix multiplication by using homogeneous coordinates.

$$y = A'x$$

$A'$ is an augmentation of matrix $A$ and looks like this for our translation above:

$$
y = 
\begin{bmatrix}
1 & 0 & 0 & 0\\
0 & 1 & 0 & 2\\
0 & 0 & 1 & 3\\
0 & 0 & 0 & 1\\
\end{bmatrix}
x = 
\begin{bmatrix}
1 & 0 & 0 & 0\\
0 & 1 & 0 & 2\\
0 & 0 & 1 & 3\\
0 & 0 & 0 & 1\\
\end{bmatrix}
\begin{bmatrix}
x_0\\
x_1\\
x_2\\
1
\end{bmatrix}
=
\begin{bmatrix}
x_0\\
x_1 + 2\\
x_2 + 3\\
1
\end{bmatrix}
$$

Note that we added a $1$ as a fourth coordinate to the point $x$.
The first three coordinates of the result $y$ describe the result of the affine transformation $y = Ax + t$.
If we wanted to scale or rotate the point $x$ at the same time, we'd just need to adapt the upper left 3x3 matrix of $A'$.
Using homogeneous coordinates we can easily combine all the transformations we need in one single matrix. 
For the `Renderables` in FIRE, the matrix $A'$ is called the *model matrix* which we encapsulate in the `Transform` class.
During rendering we can simply take the model matrix and multiply it with the view and projection matrices we used last time. 

Here's a screenshot of an example using **FIRE** to render a scene containing multiple cubes at different positions:
{:refdef: style="text-align: center;"}
<figure>
	<img src="/images/FIRE-7-Cubes.png" alt="">
</figure>
{:refdef}

We now have a way to position objects. 
The next step is to be able to also change the orientation of an object.
In the next post we'll take a look at that by adding rotations to the `Transform` class.

(You can check out the code for this post via [this git tag](https://github.com/markusrothe/FIRE/tree/FIRE-7))

