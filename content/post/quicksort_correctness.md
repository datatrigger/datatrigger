---
title: "Why quick-sort works"
summary: "Prooving the correctness of quick-sort algorithms with high-school maths."
date: 2023-01-07
tags: ["algorithms", "computer science", "sorting", "theory"]
draft: true
unsafe: true
---

# The quick-sort algorithm

Quick-sort is an efficient (in-place, unstable) sorting algorithm. Although its worst-case time complexity is $O(n^{2})$, the average complexity is $O(nlog(n))$. On real-world (random) inputs, quick-sort is even more efficient than other sorting algorithms with worst-case $O(nlog(n))$ time complexity like merge-sort.

I am currently reviewing my sorting algorithms and after all these years, quick-sort still baffles me. It is the less intuitive sorting algorithms that I know of. Now, there is a ton of content about the way quick-sort works and how to implement it. But this time, I wanted to *prove* to myself that it does work, and I had the most trouble finding a complete and rigorous correctness proof. In this post, I will formalize and gather the elements I have found [here and there](#references).

Here is the implementation we're working with:

```python

def quick_sort(array, start, end):
    '''
    Sorts input `array` between indexes `start` and `end`.
    Returns None, sorting is made in-place.
    '''
    # Base case: array of 0 or 1 elements can only be sorted
    if start >= end:
        return None
    
    # Partition input array
    l = start
    p = array[end]
    for i in range(start, end):
        if array[i] < p:
            array[l], array[i] = array[i], array[l]
            l += 1
    array[l], array[end] = array[end], array[left]

    # Recursive calls on left and right partitions
    quick_sort(array, start, l - 1)
    quick_sort(array, l + 1, end)

```

*Note*: *l* stand for 'left pointer' and *p* stands for 'pivot'

# Why the partitioning part works

They key is to show the following *invariants* hold throughout the for loop:

For each i \in \\{ start, ..., end - 1 \\}$, the following holds at the **beginning** of step i:

$$
\begin{cases}
array[k] < p \\, , \\, start < k < l_i \\\\[6pt] 
array[k] \geq p \\, , \\, l_i < k < i \\\\[6pt] 
array[k] = p \\, , \\, k = p \\\\[6pt] 
\end{cases}
$$

# Why the whole algorithm works

# References