---
title: "Monte Carlo integration"
date: 2022-11-14T08:47:11+01:00
draft: true
---

Monte Carlo integration is a technique for numerical integration using random numbers.
Suppose we have the following definite multidimensional integral

$$
I = \int_\Omega f(\vec x)\ d \vec{x} \tag{1}
$$

where $\Omega$, a subset of $\mathbb {R}^n$, has volume

$$
V = \int_\Omega d \vec x \tag{2}
$$

The naive Monte Carlo approach is to sample points uniformly on $\Omega$, giving $N$ uniform samples

$$
\vec x_1, \cdots, \vec x_N \in \Omega
$$

The samples are used to evaluate the integrand, summed, and then averaged.

$$
I \approx \dfrac{V}{N} \sum\limits_{i=1}^N f(\vec x_i)  \tag{3}
$$

## Example: 1-dimension

To give an idea of how this is implemented, we are going to integrate the equation of the standard normal distribution over the interval $[-5, 5]$. The normal distribution is defined as 

$$
\text{normal_pdf}(x, \mu, \sigma) = \dfrac{1}{\sigma \sqrt {2\pi}} e^{\frac{1}{2}\left(\frac{x-\mu}{\sigma} \right)^2}
$$

where $\mu$ is the mean and $\sigma$ is the standard deviation. The standard normal distribution is when $\mu = 0$ and $\sigma = 1$. 

<img title="Standard normal distribution" src="/std_normal.png" style="max-width: 26rem;" />

The problem is then defined as the integral of the standard normal distribution

$$
I = \int^5_{-5} \dfrac{1}{\sqrt {2\pi}} e^{\frac{1}{2}x^2}\ dx,
$$


First we need to determine the volume $V$, which in the more general case is 

$$
V = \int^a_b dx = \left[1\right]^b_a = b - a,
$$

which happens to be $V=10$ for our case.
Our samples are uniformly distributed such that $\bar x \sim \text{uniform(-5, 5)}$.



{{< highlight python >}}
def integrate_mc(f, a, b, n):
    estimate = 0
    for _ in range(n):
        x = np.random.uniform(a, b)
        estimate += f(x)
    return (b - a) * estimate / n
{{< /highlight >}}

<img title="Standard normal distribution" src="/std_normal_estimated.png" style="max-width: 26rem;" />

## Error estimation

<img title="Standard normal distribution" src="/std_normal_error.png" style="max-width: 26rem;" />

## Example: Integrating pi

## Variance reduction

https://en.wikipedia.org/wiki/Variance_reduction

## Recursive stratified sampling

## Importance sampling

### Metropolis-Hastings

https://en.wikipedia.org/wiki/Metropolis%E2%80%93Hastings_algorithm


