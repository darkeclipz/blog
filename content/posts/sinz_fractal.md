---
title: "Sin(z) fractals"
date: 2022-11-06T10:42:11+01:00
draft: false
---

Another interesting set of fractals is produced by looking at the Julia set of the iterated function

$$
z_{n+1} = \sin (z_n) \cdot c.
$$

Instead of looking at the conjugate of the complex number to determine if the orbit escapes to infinity, only the imaginary part of $z$ is used in this case. If the imaginary part of $z$ is greater than 50, then it is decided that the orbit escapes to infinity. This is important to get the correct result, because it won't work if the escape criteria of a Mandelbrot set are used.

## The function sin(z)

The sine function of a complex number $z$ is defined with the relationship

$$
\sin(z) = \sin(a + bi) = \sin a + \cosh b + (\cos a + \sinh b)i,
$$

which is used to define the `complex_sin` function.

{{<highlight rust>}}
fn complex_sin(z: Complex) -> Complex {
    Complex { 
        a: z.a.sin() * z.b.cosh(),
        b: z.a.cos() * z.b.sinh()
    }
}
{{</highlight>}}

## The map of sin(z)

With that out of the way, a `csinz` function is defined that will generate the fractal.
Note that the smooth iteration counter is not used here because it is for exponential functions, which is not the case here.

{{<highlight rust>}}
fn csinz(x: f32, y: f32) -> f32 {
    let mut z = Complex { a: x, b: y };
    let c = Complex { a: 1.0, b: 0.0 };
    let max = 256;
    let mut i = 0;
    while i < max && z.b.abs() < 50.0 {
        z = c * complex_sin(z);
        i += 1;
    }
    return i as f32 / max as f32;
}
{{</highlight>}}

Because this is a Julia set, we pick the point $(1, 0i)$ as a start. The fractal is colored with the function that is defined in the [Mandelbrot with Rust](mandelbrot-rust) post.

![sinz](/sinz.png)
<figcaption>Plot of $f(z) = c \sin(z)$ at $c = 1.0 + 0i$.</figcaption>

And voila, there it is.

## Renders of sin(z)

If we use different values for $c$, we will get different Julia sets. There are quite a few awesome looking sets that we can find. The rest of this post shows renderings of different Julia sets.

The colorization is done with the cosine color function that is defined in the [Mandelbrot with Rust](mandelbrot-rust) post. Note that we are using a few different color palettes here, which can be found on [Inigo Quilez's page about this colorization technique](https://iquilezles.org/articles/palettes/).

### Vortex

[![vortex](/vortex.png)](/vortex.png)
<figcaption>$c = 1 + 0.1i$</figcaption>

### Virus

[![virus](/virus.png)](/virus.png)
<figcaption>$c = 1 + i$</figcaption>

### Swirl

[![swirl](/swirl.png)](/swirl.png)
<figcaption>$c = 1 + 0.3i$</figcaption>

### Florets

[![florets](/florets.png)](/florets.png)
<figcaption>$c = 0.2 + i$</figcaption>

## Shadertoy implementation

I have also implemented the fractal as a shader on Shadertoy. 
It is included below.
You can use the mouse to view the different Julia sets.

<div style="text-align: center">
    <iframe frameborder="0" src="https://www.shadertoy.com/embed/ms23Dw?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe>
</div>