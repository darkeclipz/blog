---
title: "Totient maximum"
date: 2022-03-26T08:47:11+01:00
draft: false
---

In this post we will look at solving problem 69 on Project Euler. This problem is about using Euler's totient function to find a given maximum. Without any further introduction, let's dive into it.

## Problem statement

Euler's Totient function, $\varphi(n)$, is used to determine the number of numbers less than $n$ which are relatively prime to $n$. For example, as 1, 2, 4, 5, 7, and 8, are all less than nine, and relatively prime to nine, $\varphi(9) = 6$. It can be seen that $n=6$ produces a maximum for $n / \varphi(n)$ for $n \leq 10$.</p><p>Find the value of $n \leq 1,000,000$ for which $n / \varphi (n)$ is maximum.

_The table has been left out for brevity._


## Euler's Totient function

Euler's totient function, $\varphi(n)$, counts all the numbers up to $n$ that do not divide $n$. In other words, it counts all the numbers $k$ for which $\text{GCD}(k, n) = 1$. This function has been introduced by Leonhard Euler in 1763. The function is a multiplicative function, meaning that if $\text{GCD}(m, n) = 1$, then $\varphi(m) \cdot \varphi(n) = \varphi(mn)$. There is an example in the problem statement that demonstrates how to calculate the function.

Reading further on the Wikipedia page gives us another definition of $\varphi(n)$:

$$
\varphi(n) = n \prod\limits_{p | n} \left ( 1 - \dfrac{1}{p} \right ),
$$

which is a product over the distinct prime numbers dividing $n$.

## Naive implementation

A simple approach to this problem is to define a method `int EulersTotient(int n)` which loops from 1 to $n - 1$, and count if $\text{GCD}(k, n) = 1$. The implementation of this idea can be found in the code snippet below.

{{< highlight csharp >}}

bool IsCoprime(int a, int b) {
    return GCD(a, b) == 1;
}

int EulersTotient(int n)
{
    int count = 1;

    for (int k = 2; k < n; k++)
    {
        if (IsCoprime(a, b))
        {
            count++;
        }
    }

    return count;
}

{{< /highlight >}}

Note that this function grows linearly based on $n$. However, to solve the problem, we need to calculate the function for all numbers up to $n$, which will result in around $n^2 / 2$ calculations. I did implement this method and executed the code, but after a few minutes I terminated the application and started looking for a more efficient method, because this was clearly going to take a few hours.

## Fundamental theorem of arithmetic

The fundamental theorem of arithmetic states that any number can be written as a product of prime numbers. A number is either a prime number, or a composite number, where the composite number is a product of prime numbers. This fact is used to proof the alternative formula of $\varphi(n)$, which we will do now.

There exists a unique expression $n = p_1^{k_1} p_2^{k_2} \cdots p_r^{k_r}$, where $p_1, p_2, \ldots, p_r$ are prime numbers and each $k_i \geq 1$. This is a number written out as its product of primes. Since neither of the primes can divide another prime, we can use the multiplicative property of Euler's totient, which gives

$$
\begin{aligned}\varphi(n) &= \varphi(p_1^{k_1})\ \varphi( p_2^{k_2}) \cdots \varphi(p_r^{k_r}) \\\ &= p_1^{k_1-1}(p_1 - 1)\ p_2^{k_2-1}(p_2 - 1)\cdots p_r^{k_r-1}(p_r - 1) \\\ &= p_1^{k_1}\left(1 - \frac{1}{p_1}\right)\ p_2^{k_2}\left(1 - \frac{1}{p_2}\right) \cdots p_r^{k_r}\left(1 - \frac{1}{p_r}\right) \\\ &= p_1^{k_1}\ p_2^{k_2} \cdots p_r^{k_r} \left(1 - \frac{1}{p_1}\right) \left(1 - \frac{1}{p_2}\right) \cdots \left(1 - \frac{1}{p_r}\right) \\\ &= n \left(1 - \frac{1}{p_1}\right) \left(1 - \frac{1}{p_2}\right) \cdots \left(1 - \frac{1}{p_r}\right).\end{aligned}
$$

Instead of calculating the greatest common divisor for all numbers up to $n$, we can use a product of the prime factorization of $n$. This method is a lot less computationally intensive, although we now have a new problem, finding the prime factorization of $n$.

## Sieve of Eratosthenes

The idea of the sieve of Eratosthenes is to cross out composite numbers by using multiples of prime numbers, until only the prime numbers remain in the sieve. The explanation is very brief, because the sieve is used in a lot more project Euler problems, and I don't really want to get into it for this post, because <a href="https://blog.rotgers.io/2021/01/divisibility-properties-prime-numbers.html" target="_blank">I have written another post about this topic</a>. To get the general idea, this is the algorithm that is used to generate the sieve:

```
Initialize an array S of Booleans of size n+1.
Set all values in S to true.
Set S[0] to false.
Set S[1] to false.
For i in [2, (int)sqrt(n)]:
  If i is prime:
    While i * j < n:
       Set i * j to false
       Increment j by 1
```

However, instead of using an array of Booleans, this implementation will use an array of integers. A number is prime if the element in the array is zero, and otherwise the element will contain the largest prime divisor (e.g. $S[ij] = i$). This allows us to use the sieve to factorize a number into its prime factorization. This leads to the following implementation:

{{< highlight csharp >}}

class FactorizationSieve
{
    int _size { get; set; }
    int[] _sieve { get; set; }
    const int PRIME = 0;
    
    public FactorizationSieve(int n)
    {
        _size = n + 1;
        _sieve = new int[_size];
        InitializeSieve(_size);
    }

    void InitializeSieve(int n)
    {
        _sieve[0] = 1;
        _sieve[1] = 1;

        int upperBound = (int)Math.Sqrt(n);

        for (int i = 2; i < upperBound; i++)
        {
            if (IsPrime(i))
            {
                for (int j = 2; i * j < _size; j++)
                {
                    _sieve[i * j] = i;
                }
            }
        }
    }

    public bool IsPrime(int k) => _sieve[k] == PRIME;

    public IEnumerable<int> PrimeFactors(int k)
    {
        List<int> factors = new();

        while (k != 1)
        {
            int factor = _sieve[k];

            if (factor == PRIME)
            {
                factors.Add(k);
                return factors;
            }

            factors.Add(factor);
            k /= factor;
        }

        return factors;
    }

    public IEnumerable<int> DistinctPrimeFactors(int k)
    {
        return PrimeFactors(k).Distinct();
    }
}

{{< /highlight >}}

## Solving the problem

With all that in place, we can redefine $\varphi(n)$ in terms of the product of primes, instead of counting when the greatest common divisor is one. This gives an improved implementation of the Euler's totient function:

{{< highlight csharp >}}
double EulersTotient(int n, FactorizationSieve sieve) 
{
    var distinctPrimeFactors = sieve.DistinctPrimeFactors(n).ToArray();

    if (distinctPrimeFactors.Length == 0)
    {
        return 1;
    }

    double product = n;

    for (int i = 0; i < distinctPrimeFactors.Length; i++)
    {
        product *= (1.0 - 1.0 / (double)distinctPrimeFactors[i]);
    }

    return product;
}
{{< /highlight >}}

If we run this function with $n=1,000,000$, then it finds the solution in around 170 milliseconds! In the comments another user mentioned a simple optimization—which I will not spoil here—that reduced the runtime to 26 milliseconds. It can be found by carefully looking at the local minima and maxima of $n / \varphi(n)$.

An even more beautiful explanation can be found in the document that gets unlocked after solving the puzzle, which I'm also not going to spoil here, but this clever thinking allows it to be solved by hand!