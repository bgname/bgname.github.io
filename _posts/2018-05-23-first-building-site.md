---
layout: post
title:  "Python 浅拷贝和深拷贝"
date:   2018-05-23 12:56:36 +0800
categories: jekyll update
---
> 本文出自    [田小计划](http://www.cnblogs.com/wilber2013/p/4645353.html)

Python 中，对象的赋值，拷贝（深/浅拷贝）之间是有差异的，如果使用的时候不注意，就可能产生意外的结果。

下面本文就通过简单的例子介绍一下这些概念之间的差别。

# 对象赋值

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

代码的输出为：

![copy_1](/assets/2018/05/23/copy_1.jpg)

下面来分析一下这段代码：

- 首先，创建了一个名为will的变量，这个变量指向一个list对象，从第一张图中可以看到所有对象的地址（每次运行，结果可能不同）

- 然后，通过will变量对wilber变量进行赋值，那么wilber变量将指向will变量对应的对象（内存地址），也就是说”wilber is will”，”wilber[i] is will[i]”

可以理解为，Python中，对象的赋值都是进行对象引用（内存地址）传递

- 第三张图中，由于will和wilber指向同一个对象，所以对will的任何修改都会体现在wilber上

这里需要注意的一点是，str是不可变类型，所以当修改的时候会替换旧的对象，产生一个新的地址39758496

![copy_2](/assets/2018/05/23/copy_2.jpg)

# 浅拷贝

下面就来看看浅拷贝的结果：

{%  highlight ruby %}
import copy
 
will = ["Will", 28, ["Python", "C#", "JavaScript"]]
wilber = copy.copy(will)
 
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

代码结果为：

![copy_3](/assets/2018/05/23/copy_3.jpg)

分析一下这段代码：

- 首先，依然使用一个will变量，指向一个list类型的对象

- 然后，通过copy模块里面的浅拷贝函数copy()，对will指向的对象进行浅拷贝，然后浅拷贝生成的新对象赋值给wilber变量

浅拷贝会创建一个新的对象，这个例子中”wilber is not will”
但是，对于对象中的元素，浅拷贝就只会使用原始元素的引用（内存地址），也就是说”wilber[i] is will[i]”

- 当对will进行修改的时候

由于list的第一个元素是不可变类型，所以will对应的list的第一个元素会使用一个新的对象39758496

但是list的第三个元素是一个可不类型，修改操作不会产生新的对象，所以will的修改结果会相应的反应到wilber上

![copy_4](/assets/2018/05/23/copy_4.jpg)

总结一下，当我们使用下面的操作的时候，会产生浅拷贝的效果：

- 使用切片[:]操作

- 使用工厂函数（如list/dir/set）

- 使用copy模块中的copy()函数