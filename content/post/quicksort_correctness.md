---
title: "Why quick-sort works"
summary: "Prooving the correctness of quick-sort algorithms with high-school maths."
date: 2023-01-07
tags: ["algorithms", "computer science", "sorting", "theory"]
draft: false
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
    array[l], array[end] = array[end], array[l]

    # Recursive calls on left and right partitions
    quick_sort(array, start, l - 1)
    quick_sort(array, l + 1, end)

```

*Note*: *l* stand for 'left pointer' and *p* stands for 'pivot'

# Why the partitioning part works

The partitioning step (lines 12-18 above) selects a pivot *p*, say the last element of the input array. Then, the array is rearranged as follows: [ *elements < p* ..., **p**, ... *elements > p* ]. Most likely, the partitions on 'each side' of *p* are not ordered. However, *p* is already in 'the right place': when the entire array will be sorted, *p* will be at exactly the same index.

How come the partition works? Let's show the following **invariants** hold throughout the for loop:

At the **beginning** of each $i \in \\{ start, ..., end - 1 \\}$,

$$
\mathcal{P} \(i\): 
\begin{cases}
1 \) \\, array[k] < p \\, , \\, start \leq k < l_i \\\\[6pt]
2 \) \\, array[k] \geq p \\, , \\, l_i \leq k < i \\\\
\end{cases}
$$

## Intuition

$\mathcal{P} \(i\)$ means that at the beginning of step ```i``` in the for loop:

1) All the elements strictly on the left of pointer ```l``` are < ```p```
2) The elements on the right of pointer ```l``` (included) are ≥ ```p```, up until index ```i-1```

These properties are important because the mean that the array is almost partitioned as we want it to be, up to index ```i-1```. The exception is the pivot, which stays at the end of the array throughout the loop. That is why it is swapped with ```array[l]``` after the loop is completed.

Before diving into the proof of these two statements, how can we have an intuition that they are true?

1) The pointer ```l``` only moves to the right after an element < ```p``` is found at step ```i```, and put at index ```l```. So, the elements on the left of ```l``` can only be < ```p```.

2) Initially, ```l``` = ```i```. ```i``` is incremented at each step of the for loop, while ```l``` can be incremented or not. So, ```l``` ≤ ```i```. When does ```l``` < ```i```? When ```i``` moves further away but not ```l```, which happens when ```array[i]``` ≥ ```p```. So, the elements in between ```l``` and ```i``` (if any) can only be ≥ ```p```.

## Proof

The following is not a proof by induction per say because the input *array* is finite. The logic is similar though: we are going to chain a (finite) number of implications together.

### Base case

At the beginning of the first step of the for loop, $i = s = l_i$, so statements 1) and 2) are vacuous truths (no $k$ exists in both cases). Statement 3) is trivially true as we've just defined $p=array[end]$ just before entering the loop. Therefore, $\mathcal{P} \(start\)$.

### 'Induction'

Let $i \in \\{ start, ..., end - 1 \\}$ and assume $\mathcal{P} \(i\)$. Now, let's enter inside the loop and see what happens at step $i$:

&nbsp;

* **A.** If $array[i] \geq p$.

Nothing happens, the pointer ```l``` is not moved to right: $l_{i+1} = l_i$.  
  

**Statement 1)**: We can replace $l_i$ by $l_{i+1}$ at no cost: statement 1) still holds at step $i+1$.  
  
**Statement 2)**: same thing, $array[k] \geq p \\, , \\, l_{i+1} \leq k < i$ since $l_{i+1} = l_i$. Moreover, since $array[i] \geq p$, then $array[k] \geq p \\, , \\, l_{i+1} \leq k < i+1$, i.e statement 2) holds at step $i+1$

&nbsp;

* **B.** Else if $array[i] < p$. Then:

**Before the swap**: $array[i] < p$ (**B.**) and $array[l_i] \geq p$ as per statement 2) of $\mathcal{P} \(i\)$ with $k=l_i$.

So **after the swap**, $array[i] \geq p$ and $array[l_i] < p$
  
Also, the pointer ```l``` is moved to right: $l_{i+1} = l_i+1$.  
  
**Statement 1)**: $array[l_i] < p$ (after the swap), hence statement 1) is valid for $start \leq k \leq l_i$, which is equivalent to $start \leq k < l_{i+1}$ since $l_{i+1} = l_i+1$  
  
**Statement 2)**: which elements are greater than $p$ since the swap? Not $array[l_i]$ anymore. But elements at index $l_i + 1$, ..., $i-1$ (if any) are still $\geq p$ as per statement 2) (step $i$). Moreover, $array[i]$ has now joined the club, since the swap. Then $array[k] \geq p \\, , \\, l_i + 1 \leq k < i + 1$, i.e. $array[k] \geq p \\, , \\, l_{i+1} \leq k < i + 1$

In both possible cases **A.** and **B.**, $\mathcal{P} \(i+1\)$ holds.

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

The above is true at the beginning of step $i=end$, or more precisely at the end of step $i=end-1$ since the loop stops before $i=end$. At this point, one more instruction is executed: swapping $array[l_{end}]$ and $array[end]=p$. So, after the final swap, $array[l_{end}] = p$ and $array[end] \geq p$ (from $\mathcal{P} \(end\)$ - Statement 2). Finally:

$$
\begin{cases}
array[k] < p \\, , \\, start \leq k < l_{end} \\\\[6pt]
array[l_{end}] = p \\\\[6pt]
array[k] \geq p \\, , \\, l_{end} \leq k \leq end \\\\
\end{cases}
$$

Q.E.D.

# Why the whole algorithm works

Now that the correctness of the partitioning part is established, we need to prove that recursively calling ```quick_sort(array)``` sorts ```array```. This part is much easier than the correctness of the partition, and more documented as well.

* When the size of the input array is 0 or 1, ```quick_sort()``` does nothing and the array is sorted
* Suppose that ```quick_sort()``` adequately sorts any array of size $k \in \\{ 0, 1, ..., n \\}, n \geq 1$
* Then, let's see what happens upon calling ```quick_sort()``` on an array of size $n+1$. First, the array will be partitioned in three parts: ```[left_part, pivot, right_part]```. We have show in the previous section that ```pivot``` is already at the right index. Since all the elements of ```left_part``` are less than ```pivot```, if the recursive call ```quick_sort(left_part)``` sorts the left part, then the job is done on the left side. Same logic for the right part. Well, the size of ```[left_part, pivot, right_part]``` is $s_l+1+s_r=n+1$, so $s_l+s_r \leq n$: ```left_part``` and ```left_part``` cannot be more than $n$ elements! So, by the inductive hypothesis, ```quick_sort(left_part)``` and ```quick_sort(left_part)``` do sort the arrays, and the array ```[left_part, pivot, right_part]``` of size $n+1$ ends up being sorted. The inductive statement holds at step $n+1$.

The above proves by strong induction that ```quick_sort``` sorts an array of any size $n \geq 0$.

# References

* [Vassar College](https://www.cs.vassar.edu/~cs241/Lectures/6-quick-sort-post.pdf)
* [McGill School of Computer Science](https://www.cs.mcgill.ca/~dprecup/courses/IntroCS/Lectures/comp250-lecture17.pdf)
* [Stanford university](https://youtu.be/Colb_4jAy8A)