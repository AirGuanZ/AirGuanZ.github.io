---
layout: post
title: 关于makefile中的依赖文件自动生成
key: 20180427
tags:
  - makefile
---

<!--more-->

## 不正确的makefile

今天在改善我的玩具 Parser Generator [AGZParserGen](https://github.com/AirGuanZ/AGZParserGen) 的时候，我修改了一个头文件，然而当我敲下`make`的时候，它竟然告诉我目标已经是最新了。显而易见，这是写makefile的时候文件间的依赖关系出了问题。我不禁惊出一身冷汗来，因为在过去相当长的一段时间内，我都是使用同样的方式来自动生成文件依赖。以某个简单的C++项目为例：

{% highlight makefile linenos %}
CC = clang++
CC_INCLUDE_FLAGS = -I ./Include/
CC_FLAGS = -std=c++11 -O2 $(CC_INCLUDE_FLAGS) -Werror -Wall

CPP_SRC_FILES = $(shell find ./Source/ -name "*.cpp")
CPP_OBJ_FILES = $(patsubst %.cpp, %.o, $(CPP_SRC_FILES))
CPP_DPT_FILES = $(patsubst %.cpp, %.d, $(CPP_SRC_FILES))

DST = Output

$(DST): $(CPP_OBJ_FILES)
	$(CC) $^ -o $@

%.o: %.cpp
	$(CC) $(CC_FLAGS) -c $< -o $@

%.d: %.cpp
	@set -e; \
	rm -f $@; \
	$(CC) -MM $(CC_FLAGS) $< $(CC_INCLUDE_FLAGS) > $@.$$$$.dtmp; \
	sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$.dtmp > $@; \
	rm -f $@.$$$$.dtmp

-include $(CPP_DPT_FILES)

clean:
	rm -f $(DST)
	rm -f $(CPP_OBJ_FILES)
	rm -f $(CPP_DPT_FILES)
	rm -f $(shell find ./Source/ -name "*.dtmp")

run:
	make
	$(DST)
{% endhighlight %}

这里面隐藏了一个致命的错误，我很好奇为什么我以前都没有踩到这个大坑，但既然今天被我发现了，自然要拖出来批判一番。

## makefile解析

文件思路很简单，用`shell find`找到代码目录下所有的`.cpp`文件名（`CPP_SRC_FILES`），并通过后缀替换得到对应的`.o`文件名（`CPP_OBJ_FILES`）和`.d`文件名（`CPP_DPT_FILES`）。目标文件依赖于所有的`.o`，每个`.o`又依赖于自己的`.cpp`。

当然，`.o`仅仅依赖于`.cpp`是不够的，还应该依赖于该`.cpp`中包含以及间接包含的所有文件。这一依赖关系就存放在`.d`文件中，且这些`.d`在第24行被引入该makefile中。那么`.d`文件有何来头呢？它们是第17~22行代码生成的，而这几行代码我是从网上直接抄的（数个博客中都是这一段代码，我真是日了哈士奇了），它就是导致依赖关系不正确的万恶之源。下面逐行地解析这段有问题的的代码。

{% highlight makefile linenos %}
%.d: %.cpp
	@set -e; \
	rm -f $@; \
	$(CC) -MM $(CC_FLAGS) $< $(CC_INCLUDE_FLAGS) > $@.$$$$.dtmp; \
	sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$.dtmp > $@; \
	rm -f $@.$$$$.dtmp
{% endhighlight %}

第一行`%.d: %.cpp`是makefile的语法，意思是形如“%.d”的文件依赖于一个名为“%.cpp”的文件，这个没什么好说的。

第二行`@set -e;`的含义是一旦出现错误，立即放弃执行后续的命令。加上这行是为了保证我看得到生成`.d`时的错误消息，而不是被后面一连串命令的输出掩盖。

第三行`rm -f $@;`是删除已经存在的`.d`文件，其中`$@`是makefile的特殊语法，表示要生成的目标文件。

第四个命令使用了gcc/clang都支持的一个特殊功能——加上`-MM`参数表示将源文件包含以及间接包含的文件（标准库除外）以依赖项的形式输出到标准输出流。这行命令末尾把编译器的输出重定向到一个`.dtmp`的文件中，其中`$@`是目标文件名，四个美元符表示当前进程号。`.dtmp`文件只是一个临时文件，稍后会被自动删除。

第五个命令调用了`sed`，它是一个灵活的字符串操作工具，在这里用来将`.d`文件的依赖项插入到依赖文件内容中，并输出为需要的`.d`。这么说不太清晰，稍后举例即可看到。

最后一个命令删除了之前临时存放依赖关系的文件。至此，`.d`文件的生成就大功告成了。

举个简单的例子，我们在某个空目录下创建一个`src`目录，在其中放置`a.cpp`，`a.h`和`b.h`三个文件，它们包含如下内容：

{% highlight c++ %}
/* In a.h */
#include "b.h"

/* In b.h */
#include <iostream>

/* In a.cpp */
#include "a.h"

int main(void)
{

}
{% endhighlight %}

现在要把`src/a.cpp`编译为`src/a.o`，我们期望的依赖是`src/a.o src/a.d: src/a.cpp src/a.h src/b.h`。假设我们使用上述的makefile来构建这个“项目”，那么clang++输出的临时文件会是：

```
a.o: src/a.cpp src/a.h src/b.h
```

这一输出和我们期望的输出有两个差异，一个是该输出不包含`.d`文件的依赖关系，第五个命令中的`sed`就是用来添加这个关系的；另一个差异则是目标文件名——`a.o`和`src/a.o`完全不是一回事！很不幸，第二个差异是致命的，它不仅导致这个依赖关系无效（`a.o`根本不在`$(DST)`的依赖列表中），还导致`sed`命令无法查找到`src/a.o`进而无法添加`.d`文件的依赖关系。可以说，这样的依赖关系生成方案只能应对所有的`.cpp`都和makefile在同一个目录下的情况，一旦某个`.cpp`在别的目录下（比如这里的`src/a.cpp`），就会无法生成正确的依赖文件。

说到这我就来气，网络上这么多博主都在博文中如是构建依赖关系，大部分人恐怕都只是你抄我我抄你罢了……

## 错误修正

要修复这个问题也很简单，那就是修改`sed`命令的参数，让它把这两个差异都处理掉。修改后的第五个命令如下：

{% highlight shell %}
sed 's,\(.*\)\.o\:,$*\.o $*\.d\:,g' < $@.$$$$.dtmp > $@;
{% endhighlight %}

其中，`\(.*\)\.o\:`会匹配形如`src/a.o:`这样的字符串，并将其替换为`$*\.o $*\.d\:`。这里`$*`是makefile的特殊语法，表示的是`%.d`中“`%`”所代表的字符串（即去掉了后缀的文件路径）。这一行命令能够正确地将clang++输出的

```
a.o: src/a.cpp src/a.h src/b.h
```

替换为

```
src/a.o src/a.d: src/a.cpp src/a.h src/b.h
```
