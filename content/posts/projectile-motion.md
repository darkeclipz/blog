---
title: "Projectile motion (part I)"
date: 2022-11-19T10:08:00+01:00
draft: false
---

In this post we will be looking at the motion of projectiles. 
Starting from first principles by deriving the equations governing projectile motion. 
In a later post we will be studying how the different quantities are related.

## Deriving the equations

A projectile is an object which is moving over time because of its velocity and acceleration. As we know from physics, components that are perpendicular behave independently. 
In our 2D case this means that the horizontal and vertical components of motion are independent of each other. 
Any motion in the horizontal direction does not affect the motion in the vertical direction and vice versa.

### Acceleration

To keep things simple, we will start with a projectile that has no horizontal motion. 
Because we are modeling a projectile on earth, the force of gravity, denoted by $g$, is always acting on the projectile. 
Let $a_x$ be the horizontal acceleration and $a_y$ the vertical acceleration.
Since no force is being excerted in the horizontal direction, we know that $a_x = 0$.
Likewise, for the vertical component the gravitational force is pushing the projectile downwards, so $a_y = -g$.
The acceleration at time $t$ is then denoted by

$$
\mathbf a(t) = a_x \cdot \mathbf {\hat i} + a_y \cdot \mathbf { \hat j }, \tag{1}
$$

where $\mathbf {\hat i}$ is the unit vector in the x-direction, and $\mathbf {\hat j}$ is the unit vector in the y-direction.
Note that we could plug-in the values of $a_x$ and $a_y$, but we are not going to do that yet.

### Velocity

We need to integrate the function with respect to $t$, giving the indefinite integral

$$
\begin{align}
\int \mathbf a(t)\ dt &= \int \left ( a_x \cdot \mathbf {\hat i} + a_y \cdot \mathbf { \hat j } \right )\ dt \\\
 \mathbf v(t) &= a_x t \cdot \mathbf {\hat i} + a_y t \cdot \mathbf {\hat j} + C
\end{align} 
$$

Note that the constant of integration $C$ is equal to the velocity at $t=0$, denoted as $\mathbf v(t)$ giving the equation of the velocity

$$
\mathbf v(t) = a_x t \cdot \mathbf {\hat i} + a_y t \cdot \mathbf {\hat j} + \mathbf v(0) \tag{2}
$$

### Displacement

To get the displacement of the projectile, we need to integrate (2) once more with respect to $t$.

$$
\begin{align}
\int \mathbf v(t) &= \int \left( a_x t \cdot \mathbf {\hat i} + a_y t \cdot \mathbf {\hat j} + \mathbf v(0) \right)\ dt \\\
&= \frac{1}{2} a_x t^2 \cdot \mathbf {\hat i } + \frac{1}{2} a_y t^2 \cdot \mathbf {\hat j } + t \mathbf v(0) + C
\end{align}
$$

And again, the constant of integration is equal to the displacement at $t=0$, detoned as $\mathbf d(0)$ giving the equation of the displacement

$$
\mathbf d(t) = \frac{1}{2} a_x t^2 \cdot \mathbf {\hat i } + \frac{1}{2} a_y t^2 \cdot \mathbf {\hat j } + t \mathbf v(0) + \mathbf d(0) \tag{3}
$$

Alright, it looks more ugly then it is because most of the component are going to disappear once we have the initial conditions of the projectile. 
Let's look at this with an example.

## Acceleration from the gravitational force

Continuing with our example with only vertical motion. Suppose the initial position is at $(0, 0)$ and there is no initial velocity. 
This reduces (3) to only the following terms

$$
\mathbf d (t) = -\frac{1}{2} g t^2 \cdot \mathbf { \hat j } \tag{4}
$$

which gives us the position of the projectile at $t$. 
Because the projectile is only moving downward, this is a rather boring observation.
What is more interesting is that the displacement is growing quadratically with respect to time. 
This means that the particle is accelerating forever, which is happening because in this simple model there is no such thing as drag and a terminal velocity.

## Trajectories with initial velocity

To make things more interesting we will need to add an initial velocity, denoted by $\mathbf v(0)$. 
We do so by getting a unit vector from the unit circle with angle $\theta$ to get the direction, and multiply this by the force $F$.
This velocity is acting as a constant force, because there is no drag in this system.
All in all, this results in the following model to define the displacement of the projectile.

$$
\mathbf d(t) = \mathbf d(0) + F \left( \cos( \theta ) \cdot \mathbf{\hat i} +  \sin( \theta ) \cdot \mathbf{\hat j}  \right )t  -\frac{1}{2} g t ^2 \cdot \mathbf { \hat j } \tag{5}
$$

To make this a bit more tangible, we will demonstrate (5) with an example. 

## A simple demonstration

Suppose that we have the following initial conditions:

 1. The projectile starts at $(0, 0)$.
 2. The initial angle is $\pi / 4$.
 3. The initial force is $100\text m / \text s$.
 4. The gravitational force is $9.81 \text{m} / \text s ^ 2$.

We will be using Python to model these conditions and create a plot of the trajectory.
First we define all the constants, the velocity vector, and the initial position.
Then we calculate the displacement at increasing values of $t$ and plot the result.

{{< highlight python >}}
g = 9.81
a = np.array([0, -g])
angle = math.pi / 4
force = 100
vx = force * math.cos(angle)
vy = force * math.sin(angle)
v = np.array([vx, vy])
p = np.array([0, 0])

T = np.linspace(0, 14.4, 100)
x, y = zip(*[0.5 * a * (t * t) + v * t for t in T])

plt.figure(dpi=200)
plt.plot(x, y, c='#0388fc', lw=2)
plt.axhline(0, c='gray', ls='dashed', alpha=0.5)
plt.gca().set_aspect('equal')
plt.legend(['Projectile'])
plt.xlabel('Horizontal distance (m)')
plt.ylabel('Vertical distance (m)')
{{</ highlight >}}

![Projectile motion 1](/projectile-1.png)

It is cool to see that as time progresses the gravitational force is reducing the vertical component of the velocity vector.
In the next post we will be taking a deeper look into equation (5) and solve the equation for specific quantities. 
This allows us to anser questions such as "if we want to hit a point $X$, what initial angle and velocity do we need?".
