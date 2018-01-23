---
layout: article
title:  "学习算法"
categories: linux
toc: true
---

> 算法是求解一个问题所需要遵循的，被清楚指定的简单指令的集合。如果某种算法背确定是正确的，那么最重要的一步就是确定时间和空间的问题

* O()表示算法的时间复杂度
* O(1)表示常数阶复杂度
* O(n)表示线性阶复杂度
* O(n^2)表示平方阶复杂度
* 时间复杂度：在计算机科学中，算法的时间复杂度是一个函数，它定量描述了该算法的运行时间。这是一个关于代表算法输入值的字符串的长度的函数。
* 时间复杂度常用大O符号表述，不包括这个函数的低阶项和首项系数。
* 算法除了时间复杂度以外，还有空间复杂度，他们共同构成了算法复杂度。

## 如何估计一个程序所需要的时间
一般评估所需要的时间可能凭借经验进行计算。但是如果两个程序花费大致相同的时间，要确定哪个程序更快的最好方法当然就是使用编程并运行。

* 一般法则：

(1) 法则一--for循环

一个for循环最多是该for循环内的语句所运行的时间乘以它迭代的次数

(2)法则二--嵌套的or循环

一般对于嵌套的for循环，都是从里向外的进行分析，嵌套内部的一条语句的总运行时间为该语句运行的时间乘以该组所有for循环的大小乘积
例如：大O(N^2)

for(int i=0;i<n;i++)
    for(int j=0;j<n;j++)
  K++

(3)法则三--顺序语句

各个的运行时间求和即可：

例如：
for(int i=0;i<n;i++)
a[0] = 0;
for(int i=0;i<n;i++)
    for(int j=0;j<n;j++)
    a[i] += a[j] +i +j


--- 该程序是先花费了大O(N),然后嵌套循环花费了大O(N^2),所以总的也是花费了大O(N^2)

(4)法则四：--if/else语句

if (condition)   s1  else s2
 
 一个if/else语句的运行时间从不超过判断的的运行时间再加上s1和s2中运行时长者的总运行时间
