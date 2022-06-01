# Make元编程 1. 逻辑行计算和行扩展

这是一系列讨论GNU Make元编程技术文章的第一篇。

所谓元编程，是指程序在运行中产生程序或者改变自身的能力。GNU Make有许多可以考虑为“元编程”的功能，并且这些功能非常的强大。它们可以在创建极少维护的编译环境时，还能提供极大的弹性。在这一系列文章里，我会由易到难探索这些不同的功能。


## 逻辑行计算和行扩展

Make每次只会读一个Make文件的一个逻辑行。所谓的逻辑行可以是一行或者多行，结束的标志是两个反斜杠（/)或者两个换行符。读完一个逻辑行，Make就会开始计算这个逻辑行的值。

逻辑行计算涉及到Makefile的解析，以及Makefile内容转存到内部数据结构，这些结构可以用来执行编译。

例如下面这个make文件
```makefile
.PHONY: objs
objs: foo.o \
        bar.o
```
+
Make首先会计算第一行，然后读取接下来的两个物理行作为一个逻辑行并进行计算。结果是，一个名为objs、需求条件为foo.o和bar.o的目标被定义，并且声明objs是虚假的文件目标。


This is the first in a series of posts discussing metaprogramming techniques in GNU make.
Metaprogramming is the ability of a program to generate a program or to modify itself while running. It turns out that GNU make has a number of facilities that can be considered “metaprogramming”, and these capabilities can be incredibly powerful in creating build environments that require a minimal amount of upkeep and maintenance, while providing a significant amount of flexibility. In a series of posts I’ll explore these different facilities, starting from the simplest to most complex.

Evaluation and Expansion
Before we can begin to discuss metaprogramming techniques in make, it’s critical to clearly fix in our minds the distinction between evaluation and expansion. We also need to understand the order in which make performs evaluation and expansion: in some situations expansion happens before evaluation, and in other situations it happens after evaluation. Understanding these rules is key to writing correct makefiles in general, and to using metaprogramming in particular.

Evaluation
Evaluation involves parsing a makefile and storing its content into internal structures which can be used to perform the build. Make will read in a makefile one logical line at a time and once it’s fully read in, make will evaluate that line. A logical line may consist of one or more physical lines, where all but the last physical line ends with a backslash/newline pair. For a makefile such as:

.PHONY: objs
objs: foo.o \
      bar.o
make will read and evaluate the first line, then read and evaluate the next two physical lines as one logical line. The result of this will define a single target objs with prerequisites foo.o and bar.o, and declare the target to be phony.

Expansion
Expansion involves replacing some text with other text that has been previously stored. Make will expand text that is in the form of a macro reference. A macro reference is a string that starts with a single dollar sign ($) followed by the name of the macro. The macro name is either the next single character, or a string enclosed by matching parenthesis or curly braces. If the next single character is also a dollar sign, this will expand to a single dollar sign (an escape for literal dollar signs).

Given the following makefile:

F = an f
FOO = a foo
the following strings would expand as follows:

STRING	MACRO NAME	EXPANSION
$F	F	an f
$FOO	F	an fOO
$(FOO)	FOO	a foo
${FOO}	FOO	a foo
$$FOO	N/A	$FOO
Expansion is Single-Pass
It’s important to understand that make will walk every string exactly one time during expansion. As it walks the string it will expand any macro reference it discovers (which may, itself, require further expansion), then continue. When the end of the string is reached, the expansion is complete. Sometimes people believe that make will expand the string completely once without “recursing” to expand more values, then go back to the beginning of the string and start over, until there is nothing left to expand: that’s not how expansion works in make (luckily: it would be extremely inconvenient if that were the case).

To Expand or Not To Expand…
One of the more confusing things about makefiles is that the parser evaluates different parts of the makefile differently. Of interest to us here, there are two different ways expansion is handled: it can be either “immediate” or “deferred”. Immediate means that the expansion happens before evaluation of the line, so that the results of the expansion are evaluated. Deferred means that expansion happens sometime later, after evaluation of this line.

In the following set of makefile constructs I show which parts of each line are immediate or deferred:

immediate
immediate = deferred
immediate := immediate
immediate ?= deferred
immediate += immediate-or-deferred
immediate : immediate
        deferred
immediate : immediate : immediate
        deferred

include immediate
ifeq/ifneq/ifdef/ifndef immediate
The first line in the example refers to a line which make can’t parse as being any particular type of line: if the result of the expansion is not empty you’ll get a “missing separator” error here. The append assignment operator (+=) is special in that the right-hand-side is evaluated immediately if the variable has already been defined to be “simple” (using :=) or deferred otherwise.

One other thing to be aware of is that certain GNU make functions use expansion in special ways as well. For example, the $(if ...) function has the form $(if cond,then,else): first the conditional cond is expanded immediately; depending on the result either then or else is expanded, but not both. Other functions also may have special treatment of expansion: see the documentation.

Next…
In the next post, I’ll discuss the concept of recursive expansion which is the basis of all metaprogramming in GNU make.