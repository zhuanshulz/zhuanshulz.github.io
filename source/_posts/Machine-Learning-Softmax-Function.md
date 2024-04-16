---
title: 'Machine Learning: Softmax Function'
date: 2022-01-05 23:06:26
tags:
---
# Softmax Function

cite: [wikipedia_Softmax_function](https://en.wikipedia.org/wiki/Softmax_function) [DeepAI_Softmax_function](https://deepai.org/machine-learning-glossary-and-terms/softmax-layer)

The softmax function is a function that turns a vector of K real values into a vector of K real values that sum to 1. The input values can be positive, negative, zero, or greater than one, but the softmax transforms them into values between 0 and 1, so that they can be interpreted as probabilities. If one of the inputs is small or negative, the softmax turns it into a small probability, and if an input is large, then it turns it into a large probability, but it will always remain between 0 and 1.

<!--more-->

意思就是说将输入的一组数转换成概率,输入的数越大,输出的概率越大,总的输出值的和为1,输出值均大于0。

The softmax formula is as follows:
![softmax_function](img.png)

其中Zj为每一个单独的输入向量元素,K为向量中元素的个数。