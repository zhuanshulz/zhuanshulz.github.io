---
title: Pinter Declaration in C
date: 2022-03-04 12:26:29
tags:
---
# Pinter Declaration in C
<!--more-->

Not writing C for a period of time. I found a interesting gadget when declaring two pointer today.

Usually we will declare a pointer in C like this:
```C
TYPE* var;
```
and declare a list of varibles with same type like this:
```C
TYPE var1, var2, ..., varn;
```

So I try to declare two pointer with same type like this:
```C
TYPE* var1, var2;
```
Finally I found it wrong, typeof(var1) is `TYPE* `, but typeof(var2) is `TYPE`.

It is not working even if i use the following method
```C
(TYPE*) var1, var2;
```

As we can see, `TYPE*` does not denote a pointer TYPE, the declaration of pointer is divided into two part: First is declare a pointer`*val`, second identify the type of the object that this pointer point to`TYPE (*val)`. 

This problem is due to the writing habbit, we usually write `*` beside with `TYPE` which will cause the illusion of `TYPE*` is a pointer type. But we should write `TYPE *val` as a pointer(***first declare a pointer***) to a `TYPE` object(***second declare object type***).

Let's write with the right format to declare two same type pointers:
```C
TYPE *val1,*val2;
```
