---
title: "Nova fractals"
date: 2020-12-20T08:47:11+01:00
draft: false
---

<!--
    https://wiki2.org/en/Nova_fractal
    http://imajeenyus.com/mathematics/20121112_distance_estimates/distance_estimation_method_for_fractals.pdf
    https://www.iquilezles.org/www/articles/distancefractals/distancefractals.htm
    https://mathr.co.uk/blog/2013-06-22_distance_estimation_for_newton_fractals.html
    Interior smooth iter: https://www.chiark.greenend.org.uk/~sgtatham/newton/
-->

In the [previous post we looked at creating Newton fractals](/posts/newton-fractals) at arrived at the generalized Newton's method.

The method can be generalized even further, which is then known as a Nova fractal.

![Nova fractal](/nova-fractal.png)

<figcaption>Mandelbrot version of the Nova fractal, with $f(z) = z^3 - 1$, and $z=1$.</figcaption>

The Nova fractal is created by using Newton's method, and adding a parameter $c$ to the end:

$$
    z_{n+1} = z - \frac{f(z)}{f'(z)} + c.
$$

We can get a Julia version of the fractal if we have fixed value for $c$, and use $z$ for the pixel position. Likewise, we can also get a Mandelbrot version by setting $z$ to a fixed point, and using $c$ for the pixel position. Note that the value of $z$, should be a stable point, e.g. a root.

To get the image that is displayed above, the function $f(z) = z^3 - 1$ is used, which has a stable point at $z = 1$. This gives the function:

$$
    z_{n+1} = z - \frac{z^3-1}{3z^2} + c,
$$

or in more general terms, where $p$ indicates the power:

$$
    z_{n+1} = z - \frac{z^p-1}{pz^{p-1}} + c.
$$

To color this, we will keep track of how many iterations it takes before $z$ reaches a root, which is when $f(z) = 0$, or when $z_{n+1}-z_n = 0$, meaning that we haven't moved this iteration.

The [live demo below is available on ShaderToy](https://www.shadertoy.com/view/ttccRH). It will display the Mandelbrot version of the Nova fractal, and if the mouse is used the Julia version is displayed.

<iframe frameborder="0" src="https://www.shadertoy.com/embed/ttccRH?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe>

Another variation can be found if we set $z_0$ to $c$. This produces a more regular version, which can be seen in the image below.

![Nova 4th power](/nova-fractal-z4.png)

<figcaption>Nova fractal of $f : z \rightarrow z^4 - 1$, and $z_0 = c$.</figcaption>

Now all that is missing is a method to get a smooth iteration count. 
I will come back to this post once I found it, and update the images.

