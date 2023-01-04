---
title: "Useful 1-dimensional functions"
date: 2023-01-04T08:47:11+01:00
draft: false
---

## Clamp

The _clamp function ensures that a value stays within a certain range_.
This range is usually between $[0, 1]$, but it can be any range.
The function is defined as

$$
\text{ clamp(x) } = \max ( \min (x, 1), 0 )
$$

If the value of $x$ is greater than $1$, the min function ensures that the result is $1$.
Likewise, if $x$ is less than $0$, the max function ensures that the result is $0$.
Combining both functions results in the clamp function.
The clamp function has the following graph.

<iframe src="https://www.desmos.com/calculator/e8x9zyxsda?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

This function is used in a lot of other functions, as we will see with the next function.
The range $[0, 1]$ is usually parametrized to $[a, b]$.

## Smoothstep

The _smoothstep function is an interpolation and clamping function_.
It is defined by the $S_1(x)$ cubic Hermite function $3x^2 - 2x^3$, which is shown in black in the plot below.

<iframe src="https://www.desmos.com/calculator/c4qw14848o?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

However, the value of $x$ is clamped between $[0, 1]$ before applying $S_1$, resulting in the smoothstep function which is shown in red in the plot.
This is one of the most used functions in for example, computer graphics and animations. 

It is usually defined at a starting point $a$ where the function is zero, and an endpoint $b$ where the function is one. 
This results in the complete definition of the smoothstep function:

$$
\text {smoothstep}(x, a, b) = 3t^2 - 2t^3,
$$

where

$$
t = \text{ clamp }\left(\frac{x - a}{b - a}\right).
$$

In the second equation $x$ is first scaled between $[0, 1]$ and then clamped.

## Abs

The _abs function returns the absolute value of a number_. 
In other words, it always returns a positive number.
It is defined as

$$
\text { abs }(x) = |x| = \begin{align}\begin{cases}x \quad & \text{ if}\  x \geq 0 \\\ -x \quad &\text{otherwise} \end{cases}\end{align}
$$

The graph of the function is rather simple, but don't let that simplicity deceive you. 
It is incredibly useful!

<iframe src="https://www.desmos.com/calculator/frrempatvw?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

Another way of thinking about it is that it duplicates the domain of the function to reflect what is happening on the other half of the domain. 

## Fract

The _fract function returns the fractional part of a number_.
This seemingly simple operation leads the fascinating concept of domain repetition. 
One of the definitions of this function is with the modulo function:

$$
\text{ fract }(x) = \text{mod}(x, 1)
$$

The function rises linearly from 0 to 1, before doing it again, and again...

<iframe src="https://www.desmos.com/calculator/uuk78njaob?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

In the next section we will look at an example that combined all of the function that we have looked into.

## Repeating pulse

In this example we will create a function that has a sharp, but smooth, pulse at an interval. This pattern is repeating forever.

To do so, we will begin with the smoothstep function.
Instead of having it start at zero, and gradually increase to one, we want to have this the other way around.
This can be done by swapping $a$ and $b$ giving:

$$
\text{ smoothstep }(x, 1, 0)
$$

The graph below shows what happens if we change the value of $a$ between $[0.1, 1.0]$. Notice how the curve becomes much sharper.

<iframe src="https://www.desmos.com/calculator/cw51dlgsmv?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

The next thing we will do to shape the pulse is to take the absolute value of $x$ before we apply the smoothstep function:

$$
\text{ smoothstep }(|x|, 1, 0)
$$

This mirrors what is happening on the positive $x$ side to the negative side, which results in the pulse shape as can be seen below.

<iframe src="https://www.desmos.com/calculator/tjoskqnawl?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

We then offset the function such that the pulse sits at $x=0.5$.
This is done by translating the domain, like so:

$$
\text{ smoothstep }(|x-0.5|, 1, 0)
$$

All in all, our pulse function now looks like this.

<iframe src="https://www.desmos.com/calculator/ml4zgobn8c?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

To repeat the pulse an infinite number of times, the fract function is applied to $x$:

$$
\text{ smoothstep }(|\text{fract}(x)-0.5|, 1, 0)
$$

Which is the last function that we need to create the pulse function. 

<iframe src="https://www.desmos.com/calculator/dxezt9mrzd?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

## Application: Anti-aliased grid

Using this pulse function turns out to be useful for rendering computer graphics. 
The use of the smoothstep function helps with drawing a smooth anti-aliased image.
As an example, I have implemented the repeating pulse function to render a grid in two dimensions.
Note that the width of the pulse is only a few pixels!
The image is rotating to emphasize the aliasing.

<iframe width="640" height="360" frameborder="0" src="https://www.shadertoy.com/embed/mlB3z1?gui=true&t=10&paused=false&muted=false" allowfullscreen></iframe>

The shader is defined with the following code:

{{< highlight glsl >}}
mat2 rot(float angle) 
{
    float c = cos(angle), s = sin(angle);
    return mat2(c,s,-s,c);
}

void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = (2.0*fragCoord-iResolution.xy)/iResolution.y;
    
    // Rotate the grid to show aliasing.
    uv *= rot(0.2*iTime);
    
    // Time varying pixel color
    vec3 col = vec3(1);
    
    // Scale the grid so we have more lines.
    float gridSize = 4.;
    
    // Grid lines size independent of the scale of uv.
    float gridLineSize = fwidth(uv.x) * gridSize;
    
    // Domain repetition and offsetting.
    vec2 repeatedUV = abs(fract(gridSize * uv) - 0.5);
    
    // Smoothstep to get the pulse.
    vec2 grid = smoothstep(gridLineSize, 0.0, repeatedUV);
    
    // Add the grid line colors to the output color.
    vec3 gridLineColor = vec3(0.8);
    col = mix(gridLineColor, col, clamp(grid.x + grid.y, 0.0, 1.0));
    
    // Output to screen
    fragColor = vec4(col,1.0);
}
{{< /highlight >}}

Note that in GLSL the smoothstep function has the parameters swapped, so instead of the definition in this post, it is `smoothstep(a,b,x)`.