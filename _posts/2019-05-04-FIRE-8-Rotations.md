---
layout: dark-post
title:  "(FIRE-8) Rotations"
tags: [programming, FIRE, cpp, GL]
modified: 2019-05-12
categories: [FIRE]
usemathjax: true
excerpt_separator: <!-- more -->
---

{:refdef: style="text-align: center;"}
<figure>
	<img src="/images/FIRE-8-CubeRotations.gif" alt="">
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
glm also comes with a brilliant API for all kinds of matrix transformations.
In **FIRE**, we're going to store the model matrix of an object directly as a member of the `Transform` class. Applying transformations like translations or rotations then becomes trivial:

{% highlight c++ linedivs %}
class Transform::Impl
{
public:
    Impl(Vector3 const& pos, Vector3 const& viewDir)
        : m_modelMatrix(glm::translate(ToGlmVec3(pos)))
    {
        SetOrientation(viewDir);
    }

    Vector3 Position() const
    {
        return ToVec3(m_modelMatrix[3]);
    }

    Vector3 Orientation() const
    {
        glm::vec3 scale, translation, skew;
        glm::vec4 perspective;
        glm::quat orientation;
        glm::decompose(m_modelMatrix, scale, orientation, translation, skew, perspective);

        return ToVec3(glm::rotate(orientation, ToGlmVec3(m_viewDir)));
    }

    void SetOrientation(Vector3 dir)
    {
        m_viewDir = std::move(dir);
    }

    void Translate(float x, float y, float z)
    {
        m_modelMatrix = glm::translate(m_modelMatrix, glm::vec3(x, y, z));
    }

    void Rotate(Vector3 const& axis, float angle)
    {
        m_modelMatrix = glm::rotate(m_modelMatrix, glm::radians(angle), ToGlmVec3(axis));
    }

    /* ... */

private:
    glm::mat4 m_modelMatrix;
    Vector3 m_viewDir;
};
{% endhighlight %}


(You can check out the code for this post via [this git tag](https://github.com/markusrothe/FIRE/tree/FIRE-8))
    
