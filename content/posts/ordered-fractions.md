---
title: "Ordered fractions"
date: 2022-08-10T00:00:00+01:00
draft: false
# tags: ['project-euler', 'fractions']
---

In this post we will look at solving Project Euler's problem #71, which is about ordered fractions.
I have been puzzling with pen and paper for a while during this problem, and researching about fractions really paid off big time for this problem.
The final solution is pretty elegant in my opinion. Let's get into it!

## Problem statement

Consider the fraction $n / d$, where $n$ and $d$ are positive integers. If $n \lt d$ and $\text{GCD}(n, d) = 1$, it is called a reduced proper fraction.

If we list the the set of reduced proper fractions for $d \leq 8$ in ascending order of size, we get:

```
1/8, 1/7, 1/6, 1/5, 1/4, 2/7, 1/3, 3/8, 2/5, 3/7, 1/2, 4/7, 3/5, 5/8, 2/3, 
5/7, 3/4, 4/5, 5/6, 6/7, 7/8
```

It can be seen that $2/5$ is the fraction immediately to the left of $3/7$. By listing the set of reduced proper fractions for $d \leq 1,000,000$ in ascending order of size, find the numerator of the fraction immediately to the left of $3/7$.

## Naive approach

As usual, we will start with a naive implementation. The easiest solution is to simply generate all the fractions, sort them, and then find the fraction that is to the left of $3/7$. To find out if this is actually feasible, we will calculate the upper bound of the number of fractions that we need to generate.

Suppose we want to generate all the fractions op to $d \leq 5$. Suppose $n \leq d$ for simplicity, then we can number and count the fractions as illustrated in the image below.

![Countability of the rationals](/ordered-fractions.png)
<figcaption>Figure 1 — Countability of $\mathbb{Q}$.</figcaption>

This is simply $d^2$. Half of the fractions, and the diagonal, are greater than one, and we do not need those. This gives an upper bound of $(d^2 - d) / 2$. First we find the total number of fractions, then we remove the diagonal, and finally divide this by two. This leads to an upper bound of $499,999,500,000$ fractions that we need to generate. Needless to say, this naive method is not going to work (unless you have too much time on your hands).

## Farey sequences and the mediant

After trying to find any patterns in the development of the sequence of fractions, for incrementing values of $d$, there was not a lot of success. With a fresh mind after letting the problem rest for a day, I started looking on Wikipedia about fractions. At some point, this has led to a page about <a href="https://en.wikipedia.org/wiki/Farey_sequence" target="_blank">Farey sequences</a>, which is exactly the sequence as described in the problem statement. Bingo!

If we quote Wikipedia, then a Farey sequence is defined in the following manner.

<i>
  The n-th Farey sequence $F_n$ is defined as (ordered with respect to magnitude) sequence of reduced fractions $a/b$ (with coprime $a$, $b$) such that $b \leq n$.
  <br/><br/>
  If two fractions $a / c \lt b / d$ are adjacent fractions in a segment of $F_n$ then the determinant relation $bc-ad=1$ mentioned above is generally valid and therefore the mediant is the simplest fraction in the interval $(\frac{a}{c}, \frac{b}{d})$, in the sense of being the fraction with the smallest denominator.
  <br/><br/>
  Thus the mediant will then (first) appear in the $(c+d)$-th Farey sequence and is the "next" fraction that is inserted between $a / c$ and $b / d$.
</i>


<p>
  Well, well, well, if this isn't exactly what we are looking for! This gives a method to calculate the "next" fraction that appears between $a / c$ and $b / d$.
  If we look up the definition of a <i>mediant</i>, then we find that for two fractions $a / c$ and $b / d$, the mediant is defined as
  $$
  \dfrac{a + b}{c + d}.
  $$
</p>

<h2>Solving the puzzle</h2>
<p>
  Using the definitions from above, we can devise an algorithm that will successively calculate the mediant of two fractions, where the smaller fraction is repeatedly updated with the mediant, and the larger fraction, which is $3 / 7$, stays fixed. We also know that when we find the next mediant we can see to which $F_n$ it belongs, because $n = b + d$. To find the solution, we keep calculating the mediant, until we find that $n$ exceeds the upper bound of $1,000,000$.
</p>
<p>
  This leads to the following, surprisingly simple, algorithm that finds the solution to the puzzle.
</p>

{{< highlight csharp >}}
int a = 2, b = 5, c = 3, d = 7;

while (b + d <= 1e6)
{
    a = a + c;
    b = b + d;
}

Console.WriteLine(a);
{{< /highlight >}}

<p>
  This algorithm finds the solution in around 0.2 milliseconds!
</p>

<h2>Solving the next puzzle</h2>
<p>
  The next problem is about counting the number of fractions that are in $F_n$. If we look at the Wikipedia page about Farey sequences, there is a section about the sequence length and the index of a fraction. This gives us a formula to calculate the number of fractions that are in $F_n$. This formula is
  $$
  |\ F_n\ | = 1 + \sum\limits_{m=1}^n \varphi(m) = 1 + \Phi(n),
  $$
  where $\Phi$ is the summaratory totient. In other words, the sum of the sequence that we get if we evaluate Euler's totient function from $1$ to $n$. We have implemented <a href="https://blog.rotgers.io/2022/08/totient-maximum.html" target="_blank">a fast implementation of Euler's totient function using a factorization sieve in a previous post</a>. Throwing all of that together results in an algorithm that solves the puzzle in around 200 milliseconds.
</p>

<h2>Afterword</h2>
<p>
  Finally implementing this simple solution is not really the interesting part of the puzzle. As they say, the journey is more important than the destination, which really applies to this problem. I haven't heard of Farey sequences before, and as it happens to be, they are a part of a <a href="https://en.wikipedia.org/wiki/Stern%E2%80%93Brocot_tree" target="_blank">Stern-Brocot tree</a> which is another fascinating mathematical construct. A Stern-Brocot tree is a binary tree that contains all the rational numbers. Another super interesting thing I figured out is that there is a connection between Farey sequences and <a href="https://en.wikipedia.org/wiki/Ford_circle" target="_blank">Ford circles</a>, but what really blew my mind is that the Ford circles also appear in the <a href="https://en.wikipedia.org/wiki/Apollonian_gasket" target="_blank">Apollonian gasket</a> $(0, 0, 1, 1)$, which is not that interesting on its own, but somehow all of this is connected! It really is fractals all along, eh?
</p>

![Apollonian gasket](/apollonian_gasket_farey.png)
<figcaption>Figure 2 — Apollonian gasket (0, 0, 1, 1) and the Farey resonance diagram (Wikipedia)</figcaption>