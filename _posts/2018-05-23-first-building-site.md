---
layout: post
title:  "Python 浅拷贝和深拷贝"
date:   2018-05-23 12:56:36 +0800
categories: jekyll update
---
> 本文出自 [田小计划](http://www.cnblogs.com/wilber2013/p/4645353.html)
Python 中，对象的赋值，拷贝（深/浅拷贝）之间是有差异的，如果使用的时候不注意，就可能产生意外的结果。
下面本文就通过简单的例子介绍一下这些概念之间的差别。
#对象赋值
直接看下面的代码：
{% highlight ruby %}
will = ["Will", 28, ["Python", "C#", "JavaScript"]]
wilber = will
print id(will)
print will
print [id(ele) for ele in will]
print id(wilber)
print wilber
print [id(ele) for ele in wilber]

will[0] = "Wilber"
will[2].append("CSS")
print id(will)
print will
print [id(ele) for ele in will]
print id(wilber)
print wilber
print [id(ele) for ele in wilber]
{% endhighlight %}