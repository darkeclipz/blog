---
title: "Ray-sphere intersection"
date: 2022-11-10T10:08:00+01:00
draft: false
---

One of the first things you will stumble across when writing a raytracer is to find out where a ray and a sphere intersect.
In this post we will derive a method that finds this point from first principles.

## Vector-form definition of a sphere

A sphere is mathematically defined, in vector-form, as

$$
||\ \mathbf{x} - \mathbf{c}\ ||^2 = r^2,
$$

where $\mathbf{x}$ is the point on the sphere, $\mathbf{c}$ is the center of the sphere, and $r^2$ is the radius of the sphere.
It is saying that the vector between the center of the sphere and the point on the sphere must have a length of $r^2$.

<tikz>
    <script type="text/tikz">
        \begin{tikzpicture}
            \shade[shading=ball, ball color=white] (0,0) circle (4);
            \draw[dashed] (0,0) ellipse (4cm and 1.5cm);
            \draw[->, dashed] (0, 0) -- (2.2, -1.2) node[xshift=-0.8cm,yshift=0.8cm] {\LARGE $r$};
        \end{tikzpicture}
    </script>
</tikz>

The interesting thing about this _vector-form definition of a sphere_ is that it upholds in 2D, 3D, and even more dimensions, although more than 3 dimensions is a little bit hard to interpret geometrically.

{{< hint info >}}
**A side note about notation.** The expression $||\ \mathbf{x} - \mathbf{c}\ ||^2$ is the squared length of the vector. 
The square root in the formula of the length of the vector and the squaring cancel each other out. 
So instead of squaring the length of the vector, the dot-product $(\mathbf{x} - \mathbf{c}\)\cdot(\mathbf{x} - \mathbf{c}\)$ is more efficient to compute.
{{< /hint >}}

## Vector-form definition of a ray

A ray is mathematically defined, in vector-form, as

$$
\mathbf x = \mathbf o + d \mathbf u,
$$

where $\mathbf x$ is the point on the ray, $\mathbf o$ is the origin of the ray, $d$ is the distance from the origin of the ray to the point $\mathbf x$, and $\mathbf u$ is the direction of the ray. It is important to note that the _direction of the ray is a normalized vector_, otherwise the distance will not correspond to a unit of length. As with the sphere, the _vector-form definition of a ray_ will also work in 2 or 3 dimensions.

## The quadratic equation

You might wonder what this has to do with it, but bear with me for a moment. Anyone that has had high school mathematics knows that the quadratic equation is used to solve _any quadratic equation_ of the form $ax^2 + bx + c = 0$. 
Any quadratic equation is solved with the quadratic formula 

$$
x = \dfrac{-b \pm \sqrt{b^2 - 4ac}}{2a},
$$

where $b^2 - 4ac$ is known as _the discriminant_ which we will denote with the symbol $\Delta$.
Depending on the value of the discriminant $\Delta$, there are a three different cases of outcomes.

 1. $\Delta > 0$ means that the equation has _two solutions_.
 2. $\Delta = 0$ means that the equation has _exactly one solution_.
 3. $\Delta < 0$ means that the equation has no _real solutions_. It does have _imaginary solutions_, but we will ignore those for now.

After this small side tour of the quadratic equation, it is now time to find the intersection point of the ray and the sphere.

## Finding the intersection point of the ray and the sphere

The general idea is to replace $\mathbf x$ in the definition of the sphere with the definition of the ray and work this out algebraically until we get the equation in the form of the quadratic equation. We will then use the quadratic equation to find the solution(s) for $d$. Once we know $d$, we plug that into the equation of the ray to find the intersection point.

### Determining the coefficients

Starting with the definition of the sphere 

$$||\ \mathbf{x} - \mathbf{c}\ ||^2 = r^2,$$

we will replace $\mathbf{x}$ with the definition of the ray to get the equation

$$||\ \mathbf o + d \mathbf u - \mathbf{c}\ ||^2 = r^2.$$

The first thing we need to do is to get the equation in a form that is easier to manipulate. To do so, we use the dot-product from the sphere section to rewrite the squared length of the vector. This helps us to rewrite the equation as

$$
(\mathbf o + d \mathbf u - \mathbf{c}\) \cdot (\mathbf o + d \mathbf u - \mathbf{c}\) = r^2.
$$

Notice that $d$ is the only unknown variable in the equation and that the vectors $\mathbf{o}$ and $\mathbf{u}$ are constants, as is $r^2$. With this in mind we will rewrite the expression as the binomial

$$
([\mathbf o - \mathbf{c}] + d \mathbf u \) \cdot ([\mathbf o - \mathbf{c}] + d \mathbf u \) = r^2.
$$

By expanding the binomial we can rewrite it as 

$$
(\mathbf o - \mathbf c)\cdot (\mathbf o - \mathbf c) + 2(\mathbf o - \mathbf c) d \mathbf u + d^2 \mathbf u \cdot \mathbf u - r^2 = 0.
$$

Finally we rearrange the terms to get the equation in the form of a quadratic

$$
d^2 (\mathbf u \cdot \mathbf u) + 2d (\mathbf u \cdot [\mathbf o - \mathbf c]) + (\mathbf o - \mathbf c) \cdot (\mathbf o - \mathbf c) - r^2 = 0,
$$

where 

$$
\begin{align} 
    a & = \mathbf u \cdot \mathbf u  \\\\
    b & = 2 (\mathbf u \cdot [\mathbf o - \mathbf c]) \\\\
    c & = (\mathbf o - \mathbf c) \cdot (\mathbf o - \mathbf c) - r^2.
\end{align}
$$

{{< hint warning >}}
**Caution.** Be aware of not accidentaly mixing up multiplication of the scalars and the dot-products of the vectors!
The dot-product operation has been explicitly denoted with $\cdot$.
{{< /hint >}}

### Solving the quadratic equation

Once we have found the coefficients $a$, $b$, and $c$ of the quadratic equation, we can find the solutions for $d$, if any exist. The first thing we need to do is determine if there actually is a solution. We can do this by looking at the discriminant of the quadratic which is 

$$
\Delta = b^2 - 4ac.
$$

There is no solution if $\Delta < 0$, which means that the ray missed the sphere. In this case we stop the algorithm. There is exactly one solution if $\Delta = 0$, but since we are operating on floating point numbers, we will ignore this case and evaluate it as a the more general case where $\Delta \geq 0$.

To do so, we need to determine $d_1$ and $d_2$ by evaluating both cases of the quadratic formula. Because we are only interested in the point where the ray intersects with the front of the sphere, and not in the point where the ray goes out from the back, we need to determine which of those two is the closest to us. Also note that _if any solution is negative, we have intersected with a sphere that is behind us_, and since we are not looking into that direction, we will ignore that solution.

We can use the following simple procedure to find the value of $d$ that we are interested in.

 1. First we order the solutions such that $d_1 > d_2$, if this is not the case then we swap the values.
 2. Let $d^* := d_1$.
 3. If $d^* < 0$, we ignore this solution, and set $d^* := d_2$.
 4. If $d^* < 0$, then both solutions are negative meaning that there is no intersection that we are interested in.
 5. If we get here this means that $d^*$ is the solution that we are interested in, thus we have found the value of $d$.

Once we have found the value of $d$, the last step is to plug this into the ray equation to find $\mathbf{x}$.

### Finding the intersection point

To find the point $\mathbf x$ that we are interest in, all that is left to do is to evaluate the ray equation

$$
\mathbf x = \mathbf o + d \mathbf u,
$$

with the value of $d$ that we have found earlier.

{{< hint info >}}
**Todo.** Add an implementation in Rust or C# of the ray—sphere intersection algorithm.
{{< /hint >}}

## Afterword

In this post we showed how to derive the equations that are used for a ray—sphere intersection from first principles. Starting with the definition of a ray and a sphere. Using those definitions, we replaced $\mathbf{x}$ in the sphere equation with the equation of the ray. We algebraically manipulated the equation into the form of a quadratic equation, which can then be solved with the quadratic formula. Finally we looked at the solutions of the quadratic formula to determine the value of $d$ that corresponds to the solution, which we then used to find the point $\mathbf x$ by plugging this back into the ray equation.

<!-- ## Implementation of the theory

Now that we have a mathematical procedure to find the intersection point between a ray and a sphere, it is time to implement this as a procedure in code that we can use to solve this problem. To show a few different implementations, we will implemente it with C#, Rust, and Javascript. -->