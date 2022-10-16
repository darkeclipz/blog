---
title: "N-body simulation in 2D"
date: 2022-10-16T07:47:11+01:00
draft: false
---

A few days ago I implemented a n-body simulation in Python, which is used to generate the following animation:

![10-body animation](/ten-bodies.gif)

It uses a vectorized form of calculating the acceleration matrix, which is used to update the positions of the bodies. The velocity is integrated with leapfrog integration, which unlike Euler integration, is stable for oscillatory motion. It is pretty efficient. The acceleration matrix is calculated for 2D, but is easily extended to work in 3D as well.

Anyway, here is the code:


{{< highlight python >}}
n = 10
gravity = 0.0001
step_size = 20/1000
half_step_size = 0.5 * step_size
softening = 0.1

position = 4 * (np.random.rand(n, 2) - 0.5)
velocity = 1 * (np.random.rand(n, 2) - 0.5)
acceleration = np.zeros((n, 2))
mass = 500 * np.ones((n, 1))

com_p = np.sum(np.multiply(mass, position), axis=0) / np.sum(mass, axis=0)
com_v = np.sum(np.multiply(mass, velocity), axis=0) / np.sum(mass, axis=0)
for p in position: p -= com_p
for v in velocity: v -= com_v

position[0] = np.array([0, 0])
velocity[0] = np.array([0, 0])
mass[0] = 10000

fig = plt.figure()
fig.patch.set_facecolor('white')
ax = plt.axes(xlim=(-2.5, 2.5), ylim=(-2.5, 2.5))
ax.set_aspect(1)
scatter = ax.scatter(position[:,0], position[:,1], np.sqrt(mass), 
                     color="#56c474", edgecolors='black', lw=1)


def calculate_acceleration_matrix(P, M, gravity, softening):
    x = P[:,0:1]
    y = P[:,1:2]
    dx = x.T - x
    dy = y.T - y
    inv_r3 = (dx**2 + dy**2 + softening**2) ** (-1.5)
    ax = gravity * (dx * inv_r3) @ M
    ay = gravity * (dy * inv_r3) @ M
    return np.hstack((ax, ay))


def animate(i):
    global acceleration, velocity, position, softening 
    global step_size, half_step_size

    velocity += acceleration * half_step_size
    position += velocity * step_size
    
    acceleration = calculate_acceleration_matrix(
        position, mass, gravity, softening)

    velocity += acceleration * half_step_size
    scatter.set_offsets(position)


anim = animation.FuncAnimation(
    fig, animate, frames=400, interval=20, blit=False)

anim.save('filename.gif', writer='imagemagick')
{{< /highlight >}}

