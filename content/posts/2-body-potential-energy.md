---
title: "2-body potential energy function"
date: 2022-10-16T08:47:11+01:00
draft: false
---

Suppose that we have the sun-earth system. The potential energy of a satellite (any point $(x, y)$ in the system) is determined by the following equation:

$$
u =  -\frac{1-\mu}{\sqrt{(x + \mu)^2 + y^2}} - \frac{\mu}{\sqrt{[x - (1 - \mu)]^2 + y^2}} - \frac{1}{2}(x^2 + y^2),
$$

where $\mu$ is the distance from the center of rotation (the barycenter). 

If we use the equation to create a contour plot, we get a pretty cool visualization of the potential energy in the sun-earth system. The neat thing about this is that you can see the Lagrange points!

![2-body potential energy](/2body-potential-energy-plot.png)

The plot has been created with the following Python code:

{{< highlight python >}}
mu = 0.1
R = 1
sun_pos = np.array([-mu*R, 0])
earth_pos = np.array([(1-mu)*R, 0])


def f(x, y):
    t1 = -(1-mu) / ((x + mu)**2 + y**2)**0.5
    t2 = -mu / ((x - (1-mu))**2 + y**2)**0.5
    t3 = -0.5 * (x**2 + y**2)
    u = t1 + t2 + t3
    return u


n = 256
x = np.linspace(-1.75, 1.75, n)
y = np.linspace(-1.75, 1.75, n)
X, Y = np.meshgrid(x, y)

plt.figure(figsize=(8, 8))
levels = np.linspace(-2, -1.4, 10)
plt.contourf(X, Y, f(X, Y), 8, alpha=.75, cmap=plt.cm.hot, levels=levels)
C = plt.contour(X, Y, f(X,Y), 2, colors='black', levels=levels)
plt.scatter(sun_pos[0], sun_pos[1], s=300, c='#ffd352', edgecolor='black')
plt.scatter(earth_pos[0], earth_pos[1], s=50, c='#326fa8', edgecolor='black')
plt.scatter(0, 0, marker='x', lw=1, c='black')
plt.show()
{{< /highlight >}}