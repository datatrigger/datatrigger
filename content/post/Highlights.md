---
title: "Code highlighting with Hugo"
summary: "In this article we test the different ways of highlighting code with the static site generator Hugo."
date: 2020-04-26T13:41:21+02:00
draft: false
---

bla bla

# This is type 1 header

*this is italic*

**this is bold**

# Python / Code Fences

```python {linenos=table,hl_lines=[8,"15-17"],linenostart=199}
# Program to check if a number is prime or not

num = 407

# To take input from the user
#num = int(input("Enter a number: "))

# prime numbers are greater than 1
if num > 1:
   # check for factors
   for i in range(2,num):
       if (num % i) == 0:
           print(num,"is not a prime number")
           print(i,"times",num//i,"is",num)
           break
   else:
       print(num,"is a prime number")
       
# if input number is less than
# or equal to 1, it is not prime
else:
   print(num,"is not a prime number")
```

# R / Code Fences

```r {linenos=table,hl_lines=[8,"15-17"],linenostart=199}
# factor definition

Xqual<-factor( c(
    rep("0-10",1),
    rep("11-20",3),
    rep("21-30",5),
) )

# frequencies of different modalities

cat("\nfréquences :\n\n")
freq<-table(Xqual)/sum(table(Xqual))
print(freq)
p<-0.05

cat("\nmodsup :\n\n")
modsup<-mod[f
```
