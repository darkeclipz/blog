---
title: "Mandelbrot with Rust"
date: 2022-10-23T08:47:11+01:00
draft: false
---

This write-up will explain how an image of the Mandelbrot set can be created with Rust.
The image that the program generates is displayed below.
It uses a smooth iteration counter that is used to get a smooth coloring.
This is one of the first programs that I have written in Rust, and I have kept it pretty simple, meaning that it does not feature multi-threaded support.

![Mandelbrot](/mandelbrot.png)

## Importing the image crate

To generate images in Rust we can use the `image` crate. We can add this crate by adding a dependency in the `Cargo.toml` file.

{{< highlight toml >}}
[dependencies]
image = "0.24.4"
{{< /highlight >}}

And by importing the symbol in the top of our program.

{{< highlight rust >}}
extern crate image;
{{< /highlight >}}

## Defining Complex arithmetic

To create the Mandelbrot set, we will first define a `Complex` type that will implement `Add`, `Mul`, and a method for the argument squared.

{{< highlight rust >}}
#[derive(Clone, Copy)]
struct Complex {
    pub a: f32,
    pub b: f32,
}
{{< /highlight >}}

We also implement the `Clone` and `Copy` traits with the `derive` attribute.

### Addition

First we define the `Add` trait. Complex addition is defined as
$$
	(a+bi) + (c+di) = a + b + (c+d)i
$$

{{< highlight rust >}}
impl std::ops::Add for Complex {
    type Output = Self;

    fn add(self, rhs: Self) -> Self {
        Complex {
            a: self.a + rhs.a,
            b: self.b + rhs.b,
        }
    }
}
{{< /highlight >}}

### Multiplication

And then the `Mul` trait. Complex multiplication is defined as
$$
(a + bi) \cdot (c + di) = a^2 - b^2 + (ad + bc)i
$$

{{< highlight rust >}}
impl std::ops::Mul for Complex {
    type Output = Self;

    fn mul(self, rhs: Self) -> Self {
        Complex { 
            a: self.a * rhs.a - self.b * rhs.b, 
            b: self.a * rhs.b + self.b * rhs.a ,
        }
    }
}
{{< /highlight >}}

### Argument

Finally, we will also add a method to calculate the argument. The argument is defined as 
$$
|\ a + bi\ | = \sqrt{a^2 + b^2}
$$

However we will leave the square root out of it, since we do not need it. We define this function as `arg_sq`.

{{< highlight rust >}}
impl Complex {
    fn arg_sq(self) -> f32 {
        self.a * self.a + self.b * self.b
    }
}
{{< /highlight >}}

{{< hint info >}}
**Computational efficiency.** While iterating the formula, the escape criteria is $\sqrt{a^2 + b^2} \gt 2$, 
which can be more efficiently computed by squaring both sides to get rid of the square root, giving $a^2 + b^2 \gt 4$.
{{< /hint >}}

## The Mandelbrot set 

With that in place, we can define the `mandelbrot(x, y)` function. The Mandelbrot set is calculated by applying the recursive relationship:
$$
z_{n+1} = z_n^2 + c
$$
where $z$ is a complex number $(a + bi)$. 

 * If $|\ z\ | \rightarrow \infty$, then the complex number does not belong to the Mandelbrot set.
 * If $|\ z\ | \leq 2$, then we say the the complex number *does* belong to the Mandelbrot set. 

We can check if the complex numbers shoot off to infinity by evaluating the recursive relationship, say, for 256 times. If it hasn't escaped by then, we assume that it is in the Mandelbrot set and we will color that pixel black. The more iterations that we use, the more detailed the image will look. The result of the function is a value $t \in [0, 1]$ that indicates the number of iterations it reached before it escaped to infinity. If the argument squared is greater than $4$, then the orbit will escape to infinity. We are checking with $32$ instead of $2^2$ to get rid of some artifacts in the image.

{{< highlight rust >}}
fn mandelbrot(x: f32, y: f32) -> f32 {
    let mut z = Complex { a: 0.0, b: 0.0 };
    let c = Complex { a: x, b: y };
    let max = 256;
    let mut i = 0;
    while i < max && z.arg_sq() < 32.0 {
        z = z * z + c;
        i += 1;
    }
    return (i as f32 - z.arg_sq().log2().log2()) / (max as f32);
}
{{< /highlight >}}

It is also calculating the fractional part of $i$, which we use to get a continuous value for $t$. This is called a *smooth iteration counter*, and is used to create a nice transition between the colors for different discrete values of $i$.

## Calculating the pixel color

We use the value $t$ to loop over a color map function, which is going to return the RGB values of the pixel. The color map function is defined below.

{{< highlight rust >}}
fn color(t: f32) -> [u8; 3] {
    let a = (0.5, 0.5, 0.5);
    let b = (0.5, 0.5, 0.5);
    let c = (1.0, 1.0, 1.0);
    let d = (0.0, 0.10, 0.20);
    let r = b.0 * (6.28318 * (c.0 * t + d.0)).cos() + a.0;
    let g = b.1 * (6.28318 * (c.1 * t + d.1)).cos() + a.1;
    let b = b.2 * (6.28318 * (c.2 * t + d.2)).cos() + a.2;
    [(255.0 * r) as u8, (255.0 * g) as u8, (255.0 * b) as u8]
}
{{< /highlight >}}

The *color function is created by Inigo Quilez*, and is used a lot in the Shadertoy community.

## Generating the image

Finally we tie all of this together in the `main` method, where we are going to generate the image. Let $w$ be the width of the image in pixels, and $h$ the height of the image. Then the image has a domain of $x \in [0, \mathrm{w}]$ and $y \in [0, \mathrm{h}]$ which we need to normalize, because the interesting part of the Mandelbrot set resides in the rectangle $[-2, 2] \times [-2, 2]$.

We will also scale and translate the coordinates to get a better looking image.

{{< highlight rust >}}
fn main() {
    let image_width = 1920;
    let image_height = 1080;

    let mut image_buffer = image::ImageBuffer::new(
        image_width, image_height);

    for (x, y, pixel) in image_buffer.enumerate_pixels_mut() {
        let u = x as f32 / image_height as f32;
        let v = y as f32 / image_height as f32;
        let t = mandelbrot(2.5 * (u - 0.5) - 1.4, 2.5 * (v - 0.5));
        *pixel = image::Rgb(color((2.0 * t + 0.5) % 1.0));
    }

    image_buffer.save("mandelbrot.png").unwrap();
}
{{< /highlight >}}

If we run this with `cargo run`, the result will be written to `mandelbrot.png`, which results in the image that has been included on the top of this document, and below here as well.

![Mandelbrot](/mandelbrot.png)

## Julia sets

Instead of assigning our pixel position to $c$, we are going to assign it to $z$. This leaves $c$ for another arbitrary value. If we generate the image with these parameters, we get something that is called a Julia set. All of the fractals have a corresponding Julia set, which we get by assigning the position to $z$ instead of $c$.

Note that to get an interesting image, we need to use a good value for $c$. I've changed the `mandelbrot` function to generate the Julia set, which now looks like this:

{{< highlight rust >}}
fn julia(x: f32, y: f32) -> f32 {
    let mut z = Complex { a: x, b: y };
    let c = Complex { a: 0.38, b: 0.28 };
    let max = 256;
    let mut i = 0;
    while i < max && z.arg_sq() < 32.0 {
        z = z * z + c;
        i += 1;
    }
    return (i as f32 - z.arg_sq().log2().log2()) / (max as f32);
}
{{< /highlight >}}

We are using the value $(0.38, 0.28)$ for $c$ in this example. This will generate the following image:

![Julia set](/julia.png)

## Burning ship fractal

If we take the absolute value of $z$ during the iteration, we can generate the Burning ship fractal. The absolute value of a complex number is defined as
$$
\mathrm{abs}({z}) = |\ Re(z)\ | + i|\ Im(z)\ |
$$
The absolute bars are already used to indicate the argument of $z$, so I have named the function $\mathrm{abs}$ just to be clear.

We will implement this as another trait:

{{< highlight rust >}}
impl Complex {
    fn abs(self) -> Self {
        Complex { 
            a: self.a.abs(), 
            b: self.b.abs()
        }
    }
}
{{< /highlight >}}

In the while loop in the Mandelbrot function, we take the absolute of $z$ at each iteration.

{{< highlight rust >}}
while i < max && z.arg_sq() < 32.0 {
	z = z.abs();
	z = z * z + c;
	i += 1;
}
{{< /highlight >}}

If we now generate the image, we will get the following result:

![Burning ship](/burning_ship.png)

The Burning ship fractal has a pretty cool Julia set, so make sure to check that out as well!

## Final source code

This is the final source code which contains everything that we have implemented above.

{{< highlight rust >}}
extern crate image;

#[derive(Clone, Copy)]
struct Complex {
    pub a: f32, 
    pub b: f32,
}

impl std::ops::Add for Complex {
    type Output = Self;

    fn add(self, rhs: Self) -> Self {
        Complex {
            a: self.a + rhs.a,
            b: self.b + rhs.b,
        }
    }
}

impl std::ops::Mul for Complex {
    type Output = Self;

    fn mul(self, rhs: Self) -> Self {
        Complex { 
            a: self.a * rhs.a - self.b * rhs.b, 
            b: self.a * rhs.b + self.b * rhs.a ,
        }
    }
}

impl Complex {
    fn arg_sq(self) -> f32 {
        self.a * self.a + self.b * self.b
    }
}

impl Complex {
    fn abs(self) -> Self {
        Complex { 
            a: self.a.abs(), 
            b: self.b.abs()
        }
    }
}

fn mandelbrot(x: f32, y: f32) -> f32 {
    let mut z = Complex { a: 0.0, b: 0.0 };
    let c = Complex { a: x, b: y };
    let max = 256;
    let mut i = 0;
    while i < max && z.arg_sq() < 32.0 {
        z = z * z + c;
        i += 1;
    }
    return (i as f32 - z.arg_sq().log2().log2()) / (max as f32);
}

fn julia(x: f32, y: f32) -> f32 {
    let mut z = Complex { a: x, b: y };
    let c = Complex { a: 0.38, b: 0.28 };
    let max = 256;
    let mut i = 0;
    while i < max && z.arg_sq() < 32.0 {
        z = z * z + c;
        i += 1;
    }
    return (i as f32 - z.arg_sq().log2().log2()) / (max as f32);
}

fn burning_ship(x: f32, y: f32) -> f32 {
    let mut z = Complex { a: 0.0, b: 0.0 };
    let c = Complex { a: x, b: y };
    let max = 256;
    let mut i = 0;
    while i < max && z.arg_sq() < 32.0 {
        z = z.abs();
        z = z * z + c;
        i += 1;
    }
    return (i as f32 - z.arg_sq().log2().log2()) / (max as f32);
}

fn color(t: f32) -> [u8; 3] {
    let a = (0.5, 0.5, 0.5);
    let b = (0.5, 0.5, 0.5);
    let c = (1.0, 1.0, 1.0);
    let d = (0.0, 0.10, 0.20);
    let r = b.0 * (6.28318 * (c.0 * t + d.0)).cos() + a.0;
    let g = b.1 * (6.28318 * (c.1 * t + d.1)).cos() + a.1;
    let b = b.2 * (6.28318 * (c.2 * t + d.2)).cos() + a.2;
    [(255.0 * r) as u8, (255.0 * g) as u8, (255.0 * b) as u8]
}

fn main() {
    let image_width = 1920;
    let image_height = 1080;

    let mut image_buffer = image::ImageBuffer::new(
        image_width, image_height);

    for (x, y, pixel) in image_buffer.enumerate_pixels_mut() {
        let u = x as f32 / image_height as f32;
        let v = y as f32 / image_height as f32;
        let t = mandelbrot(2.5 * (u - 0.5) - 1.4, 2.5 * (v - 0.5));
        *pixel = image::Rgb(color((2.0 * t + 0.5) % 1.0));
    }

    image_buffer.save("mandelbrot.png").unwrap();
}
{{< /highlight >}}