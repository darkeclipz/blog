---
title: "Path sum: two ways"
date: 2022-06-10T00:00:00+01:00
draft: false
---

In this post we will look at solving problem 81 on Project Euler. This problem is about finding a minimal path sum in a matrix. There are three variants of this problem, and this is the easiest one.

## Problem statement

In the 5 by 5 matrix below, the minimal path sum from the top left to the bottom right, by **only moving to the right and down**, is indicated in bold red and is equal to 2427.


$$
M = \begin{pmatrix}
\color{red}{131} & 673 & 234 & 103 & 18\\\
\color{red}{201} & \color{red}{96} & \color{red}{342} & 965 & 150\\\
630 & 803 & \color{red}{746} & \color{red}{422} & 111\\\
537 & 699 & 497 & \color{red}{121} & 956\\\
805 & 732 & 524 & \color{red}{37} & \color{red}{331}
\end{pmatrix}
$$

Find the minimal path sum of the 80 by 80 matrix that is included with the problem.

## Approach

The problem requires us to find the minimal path sum which is an optimization problem. It is also possible to split the problem into multiple sub-problems. Because of these two characteristics, dynamic programming (DP) is a good approach for solving this problem.

Let $M$ be the matrix that is defined in the problem statement, where we have $m$ rows and $n$ columns. The element $M[i, j]$ is the element at the $i^\text{th}$ row and $j^\text{th}$ column. Note that we have zero-based numbering. To use dynamic programming to solve the problem, we have to define a _recursive relationship_ that identifies the optimal policy at each stage. Because of the recursive relationship, the solution procedure starts at the end and moves backward. If we start at the last element $M[4, 4] = 331$, then we have two routes that lead to this element. The element above it, which is $M[3, 4] = 956$, and the element to the left of it, which is $M[4, 3] = 37$. Of those two elements, there also is a left and top element that leads to <i>that</i> element. If we follow this pattern and create a diagram of it, we can see that the following structure is emerging.

{{< mermaid class="text-center" >}}

flowchart LR

  331 --> 956
  
  331 --> 37
  956 --> 111
  
  956 --> 121
  37 --> 121
  37 --> 524
  
  111 --> 150
  111 --> 422
  121 --> 422
  121 --> 497
  524 --> 497
  524 --> 732

{{< /mermaid >}}

We can use this structure to define the recursive relationship. The idea is to add all the distances together, and use the minimum distance if there are two possibilities. This leads to to the recursive relationship

$$
f(i, j) = M[i, j] + \min\left(f(i + 1, j), f(i, j + 1)\right).
$$

Note that the boundary conditions have been left out, to keep it more simple. The last element at $(4, 4)$ is the base case, and in that case, the equation should give $M[4, 4]$. This also doesn't check if the values are within the bounds of the array, but to fix this, the value <code>int.MaxValue</code> is returned if we access a value outside of the matrix. This trick allows us to have a more simple algorithm. We could define the recursive relationship mathematically correct, but the extra notation won't make it easier to understand in my opinion.

We can visualize the recursive relationship by changing the diagram. Note that <code>int.MaxValue</code> is used when we access an element outside the bounds of the array, which is indicated with $\infty$.

{{< mermaid class="text-center" >}}
flowchart LR
  331 --> 956["956 + min(331, ∞)"]
  
  331 --> 37["37 + min(∞, 331)"]
  956 --> 111["111 + min(956 + min(331, ∞), ∞)"]
  
  956 --> 121["121 + min(37 + min(∞, 331), 956 + min(331, ∞))"]
  37 --> 121
  37 --> 524["524 + min(∞, 37 + min(∞, 331))"]
{{< /mermaid >}}

Of course we need to do this up to $M[0, 0]$, but for brevity, this has been left out of the diagram. I think by now you should get the gist of the approach that we are using to solve the problem.

## Solving the example problem

Before we will solve the 80 by 80 matrix, we will solve the example problem first. This is the 5 by 5 matrix. To do so, we will first create the matrix $M$.

{{< highlight csharp >}}
int[,] M = new int[,]
{
    { 131, 673, 234, 103,  18 },
    { 201,  96, 342, 965, 150 },
    { 630, 803, 746, 422, 111 },
    { 537, 699, 497, 121, 956 },
    { 805, 732, 524,  37, 331 }
};
{{< /highlight >}}

We will use the recursive relationship and create a recursive function to find the solution to the problem, however, doing so will lead to extra unnecessary computations. Note that if we start at $(i, j)$, the function calculates $(i + 1, j)$ and $(i, j + 1)$. Because of the recursion, $(i + 1, j)$ will result in calculating $(i + 2, j)$ and $(i + 1, j + 1)$. If we do the same thing for $(i, j + 1)$ we need to calculate $(i + 1, j + 1)$ and $(i, j + 2)$. Note that we are calculating $(i + 1, j + 1)$ two times! To prevent this, we will add memoization. First we define a data structure for the memo.

{{< highlight csharp >}}
using Memo = System.Collections.Generic.Dictionary<(int, int), int>;
{{< /highlight >}}

With that in place, we define the recursive method <code>int PathSum(int[,] M, Memo memo, int i, int j)</code> which results in the following implementation:

{{< highlight csharp >}}
int PathSum(int[,] M, Memo memo, int i, int j)
{
    // Base case.
    if (i == M.GetLength(0) - 1 && j == M.GetLength(1) - 1)
        return M[i, j];
    
    // Out of bounds.
    if (i >= M.GetLength(0))
        return int.MaxValue;
    
    // Out of bounds.
    if (j >= M.GetLength(0))
        return int.MaxValue;

    // Memoization.
    if (!memo.ContainsKey((i, j)))
        memo[(i, j)] = M[i, j] + Math.Min(PathSum(M, memo, i + 1, j), 
                                          PathSum(M, memo, i, j + 1));

    return memo[(i, j)];
}
{{< /highlight >}}

Which we can call in the following way to solve the example problem.

**Input**

{{< highlight csharp >}}
Memo memo = new();
int sum = PathSum(M, memo, 0, 0);
Console.WriteLine($"Solution = {sum}");
{{< /highlight >}}

**Output**

```
2427
```

We find that the minimum path sum of the 5 by 5 matrix is indeed 2427. Hooray, our algorithm is correct.

## Finding the answer

Now that we have validated our implementation. Let's load in the 80 by 80 matrix. The following code reads in the text file and converts it to a 2D array of integers.

{{< highlight csharp >}}
int[,] FromFile(string filepath)
{
    string[] lines = File.ReadAllLines(filepath);

    int m = lines.Length;
    int n = lines[0].Split(',').Length;
    int[,] matrix = new int[m, n];

    for (int i = 0; i < m; i++)
    {
        int[] elements = lines[i]
            .Split(',')
            .Select(v => int.Parse(v))
            .ToArray();

        for (int j = 0; j < n; j++)
        {
            matrix[i, j] = elements[j];
        }
    }

    return matrix;
}
{{< /highlight >}}

If we run the <code>PathSum(...)</code> method on the 80 by 80 matrix, it finds the correct solution in just 5 milliseconds! The time complexity of the DP approach with memoization is $\Theta(mn - 1)$, whereas a naive implementation where we evaluate all the routes is at worst $O(2^{m + n - 3})$. For the 80 by 80 matrix, this means that with the DP approach the recursive relationship is evaluated 6399 times (not counting the base cases and when it is out of bounds), whereas a naive implementation is absolutely exploding with a total of 182687704666362864775460604089535377456991567872 steps.
