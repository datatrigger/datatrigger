---
title: "Why quick-sort works"
summary: "Prooving the correctness of quick-sort algorithms with high-school maths."
date: 2023-01-07
tags: ["algorithms", "computer science", "sorting", "theory"]
draft: true
---

# The quick-sort algorithm

Quick-sort is an efficient (in-place, unstable) sorting algorithm. Although its worst-case time complexity is $O(n^{2})$, the average complexity is $O(nlog(n))$. On real-world (≈ random) inputs, quick-sort is even more efficient than other sorting algorithms with worst-case $O(nlog(n))$ time complexity like merge-sort.

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

The partitioning step (lines 12-18 above) selects a pivot *p*, say the last element of the input array. Then, the array is rearranged as follows: [ *elements < p* ..., **p**, ... *elements > p* ]. Most likely, the partitions on 'each side' of *p* are not ordered. However, *p* is already in 'the right place': when the entire array will be sorted, *p* will be at exactly the same index.

How comes the partition works? Let's show the following **invariants** hold throughout the for loop:

At the **beginning** of each $i \in \\{ start, ..., end - 1 \\}$,

$$
\mathcal{P} \(i\): 
\begin{cases}
1 \) \\, array[k] < p \\, , \\, start \leq k < l_i \\\\[6pt]
2 \) \\, array[k] \geq p \\, , \\, l_i \leq k < i \\\\
\end{cases}
$$

## Proof

The following is not a proof by induction per say because the input *array* is finite. The logic is similar though: we are going to chain a (finite) number of implications together.

### Base case

At the beginning of the first step of the for loop, $i = s = l_i$, so statements 1) and 2) are vacuous truths (no $k$ exists in both cases). Statement 3) is trivially true as we've just defined $p=array[end]$ just before entering the loop. Therefore, $\mathcal{P} \(0\)$.

### 'Induction'

Let $i \in \\{ start, ..., end - 1 \\}$ and assume $\mathcal{P} \(i\)$. Now, let's enter inside the loop and see what happens at step $i$:

&nbsp;

* a) If $array[i] \geq p$.

Nothing happens, the pointer ```l``` is not moved to right: $l_{i+1} = l_i$.  
  

**Statement 1)**: We can replace $l_i$ by $l_{i+1}$ at no cost: still holding at step $i+1$.  
  
**Statement 2)**: same thing, $array[k] \geq p \\, , \\, l_{i+1} \leq k < i$ since $l_{i+1} = l_i$. Moreover, since $array[i] \geq p$: $array[k] \geq p \\, , \\, l_{i+1} \leq k < i+1$.

&nbsp;

* b) Else if $array[i] < p$. Then:

$array[i]$ and $array[l_i]$ are swapped: now $array[l_i] < p$ (assumed in current case b) and $array[i] \geq p$ by statement 2) with $k=i$.  
  
The pointer ```l``` is moved to right: $l_{i+1} = l_i+1$.  
  
**Statement 1)**: $array[i]$ and $array[l_i]$ have just been swapped, so now $array[l_i] < p$. Hence statement 1) is valid for $start \leq k \leq l_i$, which is equivalent to $start \leq k < l_{i+1}$ since $l_{i+1} = l_i+1$  
  
**Statement 2)**: at the beginning of the current step, $array[l_i] \geq p$ held as per statement 2) at index $i$. After swapping $array[i]$ and $array[l_i]$, now $array[i] \geq p$ holds. So, $array[k] \geq p \\, , \\, l_i \leq k < i+1$. This implies $array[k] \geq p \\, , \\, l_i+1 \leq k < i+1$, i.e. $array[k] \geq p \\, , \\, l_{i+1} \leq k < i+1$

We can conclude that $\mathcal{P} \(i+1\)$ holds.

### Conclusion

We have proven:

* $\mathcal{P} \(0\)$
* $\mathcal{P} \(0\) \implies \mathcal{P} \(1\)$
* $\mathcal{P} \(1\) \implies \mathcal{P} \(2\)$
* ...
* $\mathcal{P} \(end-1\) \implies \mathcal{P} \(end\)$

From the above, we can conclude that $\mathcal{P} \(end\)$ holds:

$$
\mathcal{P} \(end\): 
\begin{cases}
1 \) \\, array[k] < p \\, , \\, start \leq k < l_{end} \\\\[6pt]
2 \) \\, array[k] \geq p \\, , \\, l_{end} \leq k < end \\\\
\end{cases}
$$

The above is true at the beginning of step $i=end$, or more precisely at the end of step $i=end-1$ since the loop stops before $i=end$. At this point, one more instruction is executed: swapping $array[l_{end}]$ and $array[end]=p$. So, $array[l_{end}] = p$ and $array[end] \geq p$ from $\mathcal{P} \(end\)$ - Statement 2. Finally:

$$
\begin{cases}
array[k] < p \\, , \\, start \leq k < l_{end} \\\\[6pt]
array[l_{end}] = p \\\\[6pt]
array[k] \geq p \\, , \\, l_{end} \leq k \leq end \\\\
\end{cases}
$$

Q.E.D


# Why the whole algorithm works



# References