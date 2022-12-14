---
title: "28 Tauri Aa"
date: 2022-12-11T10:08:00+01:00
draft: false
---

<iframe frameborder="0" src="https://www.shadertoy.com/embed/ds2SDw?gui=false&t=0&paused=false&muted=false" allowfullscreen></iframe>

The idea of this shader is to visualize a star and the gravity field that is generated by it. A few key aspects of the shader:


 * It uses raymarching to render the scene by taking four samples per pixel.
 * The coordinates of the point on the sphere are mapped to UV-coordinates which are used for the texture.
 * The closest distance to the sphere is also being tracked by the raymarcher, which is used to add the glow. The glow is defined by three negative exponential functions, $e^{-k}$, which are added together. Although light fall-off follows the inverse-square law, the negative exponential function results in a much softer look.
 * The plane is distorted in the y-direction by the gravitional force of the star. This force is calculated as $F = G \\ /\\ r^2$, where $G$ is picked to give good looking results.
 * The step size of the raymarcher decreases inversely proportional to the gravitional force to ensure that it does not over step in the distance field. This only alleviates the problem with minor distortions.
 * 28 Tuari Aa is actually a blue star, although this one is yellow/red. I just needed a name for the shader.