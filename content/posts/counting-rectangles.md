---
title: "Counting rectangles"
date: 2022-07-06T08:47:11+01:00
draft: false
---

<p>In this post we will work out a solution to the Project Euler problem #85 where we have to count rectangles.</p><p><b>Problem statement</b></p><p>By counting carefully it can be seen that a rectangular grid measuring 3 by 2 contains eighteen rectangles. Although there exists no rectangular grid that contains exactly two million rectangles, find the area of the grid with the nearest solution.</p><p><b>Naïve&nbsp;solution</b></p><p>As usual, I always try to define a naïve implementation first, and build from there. Let $(w, h)$ be the width and the height of the rectangle $R$, and $(m, n)$ the width and the height of the sub rectangle $S$, where $S \subset R$. Note that $w, h, m, n \in \mathbb{N}$. To count how many times $S$ fits in $R$, we first find that on the x-axis it fits $w - m + 1$ times, and on the y-axis $h - n + 1$ times. Multiplying these gives the count<br />$$<br />(w - m + 1)\cdot(h - n + 1).<br />$$</p><p>Of course this is only a single rectangle $S$ in $R$, and we need to count all the possible rectangles that fit in $R$. If we combine all of this, we get the following naïve implementation that counts the rectangles.</p>
{{< highlight csharp >}}
int CountingRectangles_Naive(int maxWidth, int maxHeight) 
{
    int count = 0;

    for (int width = 1; width <= maxWidth; width++)
    {
        for (int height = 1; height <= maxHeight; height++)
        {
            count += (maxWidth - width + 1) * (maxHeight - height + 1);
        }
    }

    return count;
}
{{< /highlight >}}

<p>
  By the looks of it, most of the variables are constants. If we write this as a sum, we can probably work this out algebraically.
</p><p><b>Optimizing</b></p><p>Let $C(w, h)$ be the count rectangles function, which we can define as<br />$$<br />C(w, h) = \sum\limits_{i=1}^w \sum\limits_{j=1}^h (w - i + 1) \cdot (h - j + 1).<br />$$</p><p>Remember that $\sum_{i=1}^n i = n (n + 1) / 2$, which we will use to define two shortcuts: $W = w(w+1)/2$ and $H = h(h+1)/2$. If we continue to work out the sum<br />$$<br />\begin{align}<br />C(w, h) &amp;= \sum\limits_{i=1}^w \sum\limits_{j=1}^h (w - i + 1) \cdot (h - j + 1) \\<br />&amp;= \sum\limits_{i=1}^w (w - i + 1) \sum\limits_{j=1}^h&nbsp; (h - j + 1) \\<br />&amp;= \left( \sum\limits_{i=1}^w (w - i + 1) \right) \left( \sum\limits_{j=1}^h h - \sum\limits_{j=1}^h j + \sum\limits_{j=1}^h 1&nbsp; \right) \\<br />&amp;= \left( \sum\limits_{i=1}^w (w - i + 1) \right) \left( h^2 - H + h \right) \\<br />&amp;= (w^2 - W + w)(h^2 - H + h).<br />\end{align}$$.</p><p>Alright, instead of evaluating the loops, we now know that<br />$$<br />C(w, h) = (w^2 - W + w)(h^2 - H + h),<br />$$</p><p>which gives the optimized function:</p>

{{< highlight csharp >}}
int CountingRectangles(int w, int h)
{
    int W = w * (w + 1) / 2;
    int H = h * (h + 1) / 2;
    return (h * h - H + h) * (w * w - W + w);
}
{{< /highlight >}}

<p>Note that dividing by two is the same as bitshifting once to the right:</p>

{{< highlight csharp >}}
int CountingRectangles(int w, int h)
{
    int W = w * (w + 1) >> 1;
    int H = h * (h + 1) >> 1;
    return (h * h - H + h) * (w * w - W + w);
}
{{< /highlight >}}

<p>Although I expect that the C# compiler will optimize this for you anyway.</p><p><b>Finding the answer</b></p><p>Now that we have an optimized function to count the number of rectangles, we can start solving the problem. We will define the problem as an optimization problem where we have the cost function<br />$$<br />F(w, h) = |\ C(w,h) - 2.000.000\ |<br />$$<br />with the following goal<br />$$\min F(w,h).$$<br /><br />A simple grid search will suffice to find the solution quickly. Notice that because of symmetry, we only have to check half of the grid. A rotated rectangle still has the same area. This gives the complete solution to the problem:</p>

{{< highlight csharp >}}
using System.Diagnostics;

int CountingRectangles(int w, int h)
{
    int W = w * (w + 1) >> 1;
    int H = h * (h + 1) >> 1;
    return (h * h - H + h) * (w * w - W + w);
}

Stopwatch sw = new();
sw.Start();

int limit = 2_000_000;
int min = int.MaxValue;
(int w, int h) minRectangle = (0, 0);

for (int i = 0; i < 1000; i++)
{
    for (int j = i; j < 1000; j++)
    {
        int n = CountingRectangles(i, j);
        if (Math.Abs(n - limit) < min)
        {
            min = Math.Abs(n - limit);
            minRectangle = (i, j);
        }
    }
}

sw.Stop();

Console.WriteLine(
    $"Solution = {minRectangle.w * minRectangle.h}, "
    + $"found in {sw.ElapsedMilliseconds} milliseconds."
);
{{< /highlight >}}

<p>This implementation finds the solution in a mere two milliseconds, good enough! :)</p>