---
title: "Computing Ranges in Constant Time"
date: 2019-06-28T13:26:25+03:00
draft: false
tags: ["algorithms", "dynamic-programming", "c-sharp"]
useMath: true
summary: "Range Minimum Query with Sparse Table and Dynamic Programming."
description: "Solution for the Range Minimum Query problem with Sparse Tables and Dynamic Programming."
---
<style>
tr > th:first-child, td:first-child {
    border-right: 1px solid #000;
    font-weight: 500;
}
</style>

Suppose have some sequence of elements. We want to be able to answer questions about any of its ranges in time \\(O(1)\\). For example, we have the following sequence:

\\[ A = \\{ 5, 2, 4, 7, 6, 3, 1, 2 \\} \\]

* What is the minimum/maximum element in the range from index 0 to 3?
* What is the sum of the elements in the range from index 1 to 4?

A naive approach would simply iterate over the range and determine the result (\\(O(n)\\) search and space), or precalculate all of the possible queries (\\(O(1)\\) search and \\(O(n^2)\\) space). When speed and efficiency are essential, however, in cases when we're dealing with large datasets, we'll have to come up with something more clever.

In this article, we'll introduce the **Sparse Table** data structure and see how it, with a little bit of a preprocessing, lets us answer range queries in constant time.

## Intuition

The main idea is to precompute all of the answers for the range queries and store them in a data structure. The challenge is how to do it in an efficient way. We want to save as much space as we can thus retaining the ability to retrieve answers in constant time. Our target is \\(O(1)\\) search and \\(O(nlog_2n)\\) space and we can achieve it with **[dynamic programming](https://en.wikipedia.org/wiki/Dynamic_programming)** and some basic **arithmetic**.

We know that we can represent natural numbers as a unique decreasing sum of powers of two (yes we've just described binary). For example:

\\[ 11 = (1011)_2 = 1*2^3 + 0*2^2 + 1*2^1 + 1*2^0 = 8 + 0 + 2 + 1 \\]

We can use the same reasoning to represent a sequence as a finite union of ranges. Consider the sequence of natural numbers from 2 to 13. It can be represented in the following way:

\\[ [2 ... 12] = [2 ... 9] \cup [10 ... 11] \cup [12 ... 12] \\]

\\([2 ... 12]\\) has \\(11\\) elements and we broke it down to ranges of \\(8\\), \\(2\\) and \\(1\\) elements respectively. We can also observe that such union can consist of **at most** \\(log_2N\\) ranges where \\(N\\) is the length of the original sequence.

## Efficiently precomputing the results

We're going to compute range minima. Let's go back to our example sequence \\(A\\) and encode all the possible answers in the **sparse table**. We're going to represent it as a two-dimensional array \\(M\\) of size \\(N \times K\\), where \\(K = \lfloor {log_2N} \rfloor + 1\\). Every cell in this matrix will contain the index of the minimum in a particular range. Note that these ranges have **sizes of powers of \\(2\\)** which means that **we compute the minima only of those ranges**, hence the size \\(O(Nlog_2N)\\)

```csharp
var A = new[] { 5, 2, 4, 7, 6, 3, 1, 2 };
var N = A.Length;
var K = (int)Math.Floor(Math.Log(N, 2)) + 1;
var M = new int[N, K]; // The Sparse Table
```

### Basis

A range of length \\(1\\) is still a valid range. So the minimum of \\(A[1...1]\\) is exactly A[1] = 2, therefore, filling the first row of the table is trivial.

```csharp
for (int i = 0; i < N; i++)
    M[i, 0] = i;
```

In other words, we have computed the minima of all the ranges starting at index \\(i\\) of length \\(1 = 2^0\\).

### Iteration

This is where things get interesting. We introduce a general procedure for determining the minimum of a range of size \\(2^j\\), where \\( 1 \le j \le log_2N  \\). Assuming that we've already found the minima for all ranges of size \\(2^{j-1}\\), we're going to reuse those solutions to find the minima for \\(2^j\\). This is dynamic programming in its essence. We break down a problem into smaller sub-problems, solve the simplest case and work our way up.  

Before diving into the mathematics of this procedure, let's go through an example. This is our sequence:

\\[ A = \\{ 5, 2, 4, 7, 6, 3, 1, 2 \\} \\]

So our sparse table stores the indices of the minima in a certain range. After computing the basis, we end up with the following table:

| j\i   | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|:-----:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **0** | 5 | 2 | 4 | 7 | 6 | 3 | 1 | 2 |

Note that **this is not a 1 to 1 representation of the way we actually store the data**. Here for the sake of readability, I'm showing the actual values in the cells whereas in the implementation we store indices. Above are the already computed ranges of length 1. Let's see the next step.

| j\i   | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|:-----:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **0** | 5 | 2 | 4 | 7 | 6 | 3 | 1 | 2 |
| **1** | 2 | 2 | 4 | 6 | 3 | 1 | 1 |   |

Now we build our way up. We compute the ranges of length \\(2 = 2^1 \\). The way we interpret the values, \\(M[i, j]\\) is the minimum in the range from index \\(i\\) to \\(i + 2^j - 1\\) in \\(A\\). So at \\(M[0, 1]\\) we need to insert the minimum in the range from \\(0\\) to \\(1\\). We can split the range into two equal subranges. Looking at the table above, we've already computed them, so we take the smaller value. \\(M[0, 1] = Min(M[0, 0], M[1, 0]) \\).

The next step should be more representative of the power of dynamic programming. Now \\(j = 2\\) so we find the minima of the ranges of length \\(2^2 = 4\\).

| j\i   | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|:-----:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **0** | 5 | 2 | 4 | 7 | 6 | 3 | 1 | 2 |
| **1** | 2 | 2 | 4 | 6 | 3 | 1 | 1 |   |
| **2** | 2 | 2 | 3 | 1 | 1 |   |   |   |

So \\(M[i = 0, j = 2]\\) represents the smallest element in the range from 0 to 3 (the first four), which is indeed 2. We came up with the result by only looking at the row above. We can represent \\(A[0 ... 3] = A[0 ... 1] \cup A[2 ... 3]\\). We already know the minima of \\(A[0 ... 1]\\) and \\(A[2 ... 3]\\). They are located at \\(M[0, 1] = 2\\) and \\(M[2, 1] = 4\\) so we simply picked the smaller number. 

| j\i   | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|:-----:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **0** | 5 | 2 | 4 | 7 | 6 | 3 | 1 | 2 |
| **1** | 2 | 2 | 4 | 6 | 3 | 1 | 1 |   |
| **2** | 2 | 2 | 3 | 1 | 1 |   |   |   |
| **3** | 1 |   |   |   |   |   |   |   |

The last row of the table is computed in the same way. The minimum of the first 8 is the smaller between value between the minima of the first four and the next four elements. \\(M[0, 3] = Min(M[0, 2], M[4, 2])\\).

Formally we can describe the procedure as:

\\[
  M[i, j] =
    \begin{cases}
      M[i, j-1], & \text{if } A[M[i,j-1]] \le A[M[i + 2^{j-1}, j - 1]] \newline
      M[i + 2^{j-1}, j - 1], & \text{otherwise}
    \end{cases}
\\]

The range index calculation might seem a bit unintuitive at first but it's actually pretty straightforward.

\\[
A[i ... i + 2^j - 1] = A[i ... i + 2^{j-1} - 1] \cup A[i + 2^{j-1} ... i + 2^j - 1]
\\]

Both sub-ranges have a length of \\(2^{j-1}\\). This is how we turn this formal notation into code:

```csharp
for (int j = 1; j < K; j++) {
    // 1 << j = 2^j
    for (int i = 0; i + (1 << j) <= N; i++) {
        int left = M[i, j - 1];
        int right = M[i + (1 << (j - 1)), j - 1];
        M[i, j] = A[left] <= A[right] ? left : right;
    }
}
```

## The Range Query

Now our **sparse table** is constructed, we are ready to process queries. We've stored the minima for the ranges that are a power of two, but how do we compute minimum for arbitrary ranges?

The idea is to select two blocks that entirely cover this range. Suppose we have an arbitrary block \\(A[p ... q], \text{where } p < q \\) and we need to find the minimum. 

Let \\(k = \lfloor log_2(q - p + 1)]\rfloor \\), \\(2^k\\) is **the size of the largest block in the table that fits into the range** \\(A[p ... q]\\). Then we can compute the minimum by comparing the minima of the following blocks: \\( A[p ... p + 2^k] \text{ and } A[q - 2^k + 1 ... k] \\). Formally

\\[
  RangeMinimum(p, q) = Min(M[p, k], M[q - 2^k + 1, k])
\\]

Let's see an example. We're going to use the same **sparse table** that we computed in the previous seciton.

| j\i   | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|:-----:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **0** | 5 | 2 | 4 | 7 | 6 | 3 | 1 | 2 |
| **1** | 2 | 2 | 4 | 6 | 3 | 1 | 1 |   |
| **2** | 2 | 2 | 3 | 1 | 1 |   |   |   |
| **3** | 1 |   |   |   |   |   |   |   |

What is the range minimum of 1 and 5?

```
p = 1, q = 5
k = floor(log(5 - 1 + 1)) = 2
M[1, 2] = 2
M[5 - 2^2 + 1, 2] = M[2, 2] = 3
return 2 
```

The block \\(A[1 ... 5] \\) contains \\( { 2, 4, 7, 6, 3 }\\) so we got a correct answer in constant time! But what do these calculations actually mean?

1. We found the size of the largest block in A[1 ... 5], with size power of 2 by calculating \\(k\\). The size of this block is 4.
2. We already know the minima of all blocks with sizes of 4. 

Therefore we pick two **overlapping** ranges of this length. The first **starts at \\(p\\)** and the other **ends at \\(q\\)**. The whole range includes:

```
A[1 ... 5] = { 2, 4, 7, 6, 3 }
left = { 2, 4, 7, 6 }
right = { 4, 7, 6, 3 }
```

We have converted the question from something we don't know to something we know and thus can easily determine the result of the query in \\(O(1)\\).

```csharp
public int RangeMinimum(int[] A, int[,] M, int p, int q) {
    var k = (int)Math.Floor(Math.Log((q - p + 1), 2));
    var left = M[p, k];
    var right = M[q - (1 << k) + 1, k];
    
    return A[left] <= A[right] ? left : right;
}
```

This algorithm can be easily tweaked so it computes some other property like maximum for example.

## Range Sums
Let's see how to compute range sums in constant time using a sparse table. We need to slightly modify our precomputation procedure. The main difference is that instead of storing indexes to elements in the array, we store sums.

```diff
for (int i = 0; i < N; i++)
-    M[i, 0] = i;
+    M[i, 0] = A[i];
```

```diff
for (int j = 1; j < K; j++) {
    // 1 << j = 2^j
    for (int i = 0; i + (1 << j) <= N; i++) {
        int left = M[i, j - 1];
        int right = M[i + (1 << (j - 1)), j - 1];
-       M[i, j] = A[left] <= A[right] ? left : right;
+       M[i, j] = left + right;
    }
}
```

For computing a sum of an arbitrary range \\(A[p ... q]\\), we're going to use the observation that any range is a union of subranges with lengths of powers of \\(2\\). We start with the largest such subrange contained in \\(A[p ... q]\\) and continue by adding the sums of the subsequent smaller ones, but only if they are within the bounds of \\(A[p ... q]\\).

```csharp
public int RSQ(int[,] M, int p, int q) {
    var sum = 0;
    // The size of the table's second dimension
    var K = M.GetLength(1);

    for (int j = K; j >= 0; j--) {
        if ((1 << j) <= (q - p + 1)) {
            sum += M[p, j];
            p += 1 << j;
        }
    }
    return sum;
}
```

Note that the sum query will run in \\(O(log_2N)\\) so it is not constant time, but it's still pretty good.

## Conclusion

One can go a long way with some preprocessing. We've got a constant speed with just a little bit of overhead in terms of memory due to the logarithms. There's another price we pay for \\(O(1)\\) time though and that's immutability. If we modify our sequence, we'd have to run the precomputation procedure all over again.

To speed things up a bit more we can precalculate the logarithms. For a sequence of size \\(N\\) for all queries, we'll have \\(N\\) different log values. This can also be done with simple dynamic programming. You can check the complete implementations in the references below.

There's an \\(O(n)\\) space with \\(O(1)\\) time solution for the **RMQ** problem introduced by Farach-Colton and Bender in their [_"The LCA Problem Revisited"_](https://www.ics.uci.edu/~eppstein/261/BenFar-LCA-00.pdf) paper which builds on top of the one in this article, but is quite a bit more complex. If space efficiency is critical, then I'd recommend checking it out. The idea behind it is very clever too.

## References and Further Reading

* Code reference for [RMQ](https://gist.github.com/deniskyashif/bcdd96f1652f2f404c528886f104fee5) and [RSQ](https://gist.github.com/deniskyashif/53b17532aae120fcc1a2345f617a102b)
* [Sparse Tables on CP-Algorithms](https://cp-algorithms.com/data_structures/sparse-table.html)
* [Range Minimum Query (Wikipedia)](https://en.wikipedia.org/wiki/Range_minimum_query)
* [Farch-Colton, Bender, _"The LCA Problem Revisited"_](https://www.ics.uci.edu/~eppstein/261/BenFar-LCA-00.pdf) - linear space, constant time solution.
