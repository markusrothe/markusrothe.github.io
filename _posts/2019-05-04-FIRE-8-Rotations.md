---
layout: dark-post
title:  "(FIRE-8) Rotations"
tags: [programming, FIRE, cpp, GL]
modified: 2019-05-04
categories: [FIRE]
usemathjax: true
excerpt_separator: <!-- more -->
---

{:refdef: style="text-align: center;"}
<figure>
	<img src="/images/FIRE-8-CubeRotation.gif" alt="">
</figure>
{:refdef}

Welcome to the 8th post about my rendering engine project **FIRE**!

Within this series I want to describe how I build up a rendering engine from scratch and my thoughts about it along the way.
All the code will be open source and will be hosted here: [FIRE](https://github.com/markusrothe/FIRE)

[In the last post](https://www.markusrothe.dev/fire/2019/05/01/FIRE-7-Model-Matrix.html) we've added support to position and translate `Renderables` within a 3D scene.
Today, we'll extend it further to also support rotations.

<!-- more -->

#### Quaternions
We're going to implement rotations with the help of [quaternions](https://en.wikipedia.org/wiki/Quaternion).
Quaternions are an extension of complex numbers that make it relatively easy to describe rotations.
A quaternion looks like this:

$$q = [w, x, y, z] = w + xi + yj + zk$$

It consists of a vector part $$(x, y, z)$$ and a scalar part $$w$$. 
Similar to the value $$i$$ for complex numbers, $$(i, j, k)$$ denotes the imaginary part of the quaternion, so

$$i^2 = j^2 = k^2 = -1$$

For now, it is sufficient to describe a rotation via an rotation-axis $$A$$ and an angle $$\omega$$. 
You can think of the quaternion encoding this rotation.
In fact, the quaternion that corresponds to a rotation around $$A$$ with angle $$\omega$$ can be written like this:

$$q = cos\frac{\omega}{2} + Asin\frac{\omega}{2} = 
\begin{bmatrix}
w \\
x\\
y\\
z
\end{bmatrix} = 
\begin{bmatrix}
cos\frac{\omega}{2}\\
A_{x}sin\frac{\omega}{2}\\
A_{y}sin\frac{\omega}{2}\\
A_{z}sin\frac{\omega}{2}
\end{bmatrix}$$

A rotation of a point $$P$$ via the quaternion $$q$$ can be described with the following formula:

$$\phi(P) = qPq^{-1}$$

$$q^{-1}$$ is the *inverse* of a quaternion which is defined as: $$q^{-1} = \frac{\overline{q}}{q^2}$$

$$\overline{q}$$ is the *conjugate* of a quaternion $$\overline{q} = w - xi - yj - zk$$.

$$q^2$$ is the *dot-product* of $$q$$ with itself, meaning $$q^2 = w^2 + x^2 + y^2 + z^2$$

With that, we have all the equations we need to be able to rotate a Point P around an axis. 

#### Integration into **FIRE**

Instead of implemening quaternions myself, I am making use of the [OpenGL Mathematics library (glm)](https://glm.g-truc.net/0.9.9/index.html) in **FIRE**. 
Internally, glm implements (among many other things) quaternions as described above. 
Here is the code for the rotation of a point `m_lookAt` around an axis with a given angle (in degrees).

{% highlight c++ linedivs %}
void Rotate(Vector3 const& axis, float angle)
{
    auto const result = glm::rotate(
        glm::vec3(m_lookAt.x, m_lookAt.y, m_lookAt.z),
        glm::radians(angle),
        glm::vec3(axis.x, axis.y, axis.z));

    /* ... */
}
{% endhighlight %}


(You can check out the code for this post via [this git tag](https://github.com/markusrothe/FIRE/tree/FIRE-8))
    
