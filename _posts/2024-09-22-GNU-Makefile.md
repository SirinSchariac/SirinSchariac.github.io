---
layout: 	post  				   
title: 		GNU Makefile      				
subtitle:	make and cmake
date:       2024-12-07 				
author:     Sirin 						
header-img: img/view.jpg
catalog: 	true 				
tags:						
    - Makefile
    - Tools
---

## Makefile

### 1. Makefile规则

```makefile
target ... : prerequisites
	command
	...
	...
```

- `target`是这个makefile指定的操作所要产生的目标，可以是一个目标文件object file，也可以是一个可执行文件，也可以是一个标签label
- `prerequisites`是生成`target`所需要的依赖文件
- `command`是生成该文件所需要的命令

> 注意
>
> 1. 当`prerequisites`中的文件比`target`更新时间更晚的话，则需要重新执行`command`对应的指令
> 2. `command`新起一行之后，需要开头用tab缩进

### 2. 书写Makefile规则

#### 2.1 example

```makefile
foo.o : foo.c defs.h
	cc -c -g foo.c
```

如果你不想另起一行写`command`，也可以：

```makefile
foo.o : foo.c defs.h ; cc -c -g foo.c
```

这个规则的含义：

- **文件的依赖关系**，`foo.o`作为`target`需要依赖于`foo.c`和`def.h`
- **如何生成target**，也就是`cc -c -g foo.c`命令的作用

如果命令很长，可以使用`\`作为换行符。

一般来说，make使用UNIX的标准Shell(`/bin/sh`)来执行命令。

#### 2.2 规则通配符

make支持三种通配符`*`, `?`, `~`。

下面以`*`为例进行介绍。

`*.c`表示了一系列`.c`后缀的文件，如果文件名中存在`*`则需要进行转义`\*`来表示名称中的`*`，一个使用`*`通配符的例子如下

```makefile
clean:
	rm -f *.o
```

注意，在变量中使用`*`并不会展开，例如

```makefile
objects = *.o
```

这种情况下，objects的值就是`*.o`而并非所有`.o`后缀的文件，如果想要让objects的值是所有`.o`后缀的文件，则应该使用如下的定义方式

```makefile
objects := $(wildcard *.o)
```

#### 2.3 文件搜索

Makefile中使用`VPATH`特殊变量来指名文件所在的目录

```makefile
VPATH = src : ../headers
```

不同目录之间使用`:`进行分隔，例如上面就给出了两个目录`src`和`../headers`，当然Makefile还是会最优先搜索当前目录

一种更灵活的方式是使用`vpath`关键字函数，该函数根据参数有三种使用方法

```makefile
vpath <pattern> <directories>
```

​	对于所有满足模式`<pattern>`的文件，指定搜索目录`<directories>`

```makefile
vpath <pattern>
```

​	清除满足模式`<pattern>`的文件的搜索目录

```makefile
vpath
```

​	清除所有被设置的搜索目录

对于`<pattern>`，使用`%`来进行匹配（同样的，如果文件名字中`%`则需要用`\%`来进行转义），比如：

```makefile
vpath %.h ../headers
```

该语句表示make会在`../headers`中寻找所有满足`.h`后缀的文件。（如果有文件在当前目录没有找到的话）

```makefile
vpath %.c foo:dog
vpath %   cat
```

vpath进行的顺序查找，在上例中，会按照`foo->dog->cat`的顺序查找

#### 2.4 伪目标

一般来说在make生成了大量的编译文件，需要相应的make clean操作来删除这些文件，以用于完整地重新编译。

```makefile
.PHONY : clean
clean:
	rm *.o temp
```

这里显然clean不是一个target，通过`.PHONY`可以显式地指定这一目标是伪目标，不与任何文件关联。

为什么要有`.PHONY`？

make实际上就是一种自动化的文件系统，伪目标存在的意义就是可以在这种系统下独立的去运行一些功能而不需要考虑文件的影响，比如，如果你真的有一个文件叫做`clean`，那么`.PHONY`就帮助make避免了混淆。

一般来说，伪目标没有依赖文件，但也可以为其指定依赖文件，并将其**放在第一个的位置**，就可以让伪目标作为“默认目标”来工作。例如，如果你想用一次make来生成多个目标文件，那么就可以使用伪目标的特性

```makefile
all : prog1 prog2 prog3
.PHONY : all

prog1 : prog1.o utils.o
	cc -o prog1 prog1.o utils.o
prog2 : prog2.o
	cc -o prog2 prog2.o
prog3 : prog3.o sort.o utils.o
	cc -o prog3 prog3.o sort.o utils.o
```

可以看到，在这里目标可以成为伪目标的依赖，那么同样的，伪目标也可以作为依赖。

```makefile
.PHONY : cleanall cleanobj cleandiff

cleanall : cleanobj cleandiff
	rm program

cleanobj : 
	rm *.o

cleandiff :
	rm *.diff
```

在这里`cleanobj`和`cleandiff`类似于`cleanall`的子程序，两者可以用于特定类型的清除操作，而`cleanall`则实现了清除所有指定的文件。

#### 2.5 多目标

如果说多个目标的依赖文件都是同一个，并且生成规则大体相同，那么就可以使用自动化变量`$@`来表示当前目标规则中所有的目标的集合，例如

```makefile
aout bout : text.g
	generate text.g -$(subst out,,$@) > $@
```

上述规则等价于

```makefile
aout : text.g
	generate text.g -a > aout
bout : text.g
	generate text.g -b > bout
```

这里函数的作用是替换字符串，`$@`是对象集合，类似于一个数组，从中取出目标然后依次执行。

#### 2.6 静态模式

静态模式的语法

```makefile
<targets ...> : <target-pattern> : <prereq-pattern ...>
	<commands>
	...
```

`targets`用于定义一系列的目标文件，可以用通配符，是目标的一个集合。

`target-pattern`是指明了`targets`的模式，也就是目标集模式。

`prereq-pattern`是目标的依赖模式，是对于`target-pattern`形成的模式再一次进行依赖目标的定义。

下面结合一个例子来解释

```makefile
objects = foo.o bar.o
all : $(object)

$(object) : %.o : %.c
	$(CC) -c $(CFLAGS) $< -o $@
```

这里我们指明了目标文件从`$object`中获取。

`%.o`表示这里是要所有以`.o`结尾的目标，也就是`foo.o`和`bar.o`，这样就指明了目标集合的模式。

依赖模式`%.c`中的缺省是取`%.o`中的`%`代表的值，即`foo`和`bar`，然后加上`.c`的后缀。

`$<`代表第一个依赖文件，`$@`表示目标集合(`foo.o bar.o`)

上述规则展开后就是

```makefile
foo.o : foo.c
	$(CC) -c $(CFLAGS) foo.c -o foo.o
bar.o : bar.c
	$(CC) -c $(CFLAGS) bar.c -o bar.o
```

#### 2.7 自动生成依赖性

有时候一个文件里可能会有多个头文件，这样子显式地在Makefile里给出依赖就很麻烦。

在编译器中，为了避免这种事情，可以使用`-M`选项来自动搜寻源文件中包含的头文件，例如

```makefile
cc -M main.c
```

有时候GNU的编译器会涵盖一些标准库的头文件，此时可以用`-MM`选项。

但这是编译器，如何在Makefile中实现这样的功能呢？GNU组织的标准中建议把编译器为每一个源文件自动生成的依赖关系放到一个文件中，为每一个 `name.c` 的文件都生成一个 `name.d` 的Makefile文件， `.d` 文件中就存放对应 `.c` 文件的依赖关系。

因此，可以写出`.c`文件和`.d`文件的依赖关系，并且让make自动更新或者生成`.d`文件，将其包含在Makefile中，从而自动化生成依赖关系。这一模式规则可以如下给出：

```makefile
%.d : %.c
	@set -e; rm -f $@; \
	$(CC) -M $(CPPFLAGS) $< > $@.$$$$; \
	sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g < $@.$$$$ > $@; \
	rm -f $@.$$$$
```

1. `@set -e;`：
   - 使得如果命令失败（返回非零状态），则整个脚本将会中止。`@` 符号在这里用于隐藏命令本身的输出。
2. `rm -f $@;`：
   - 删除目标文件 `$@`（在这个上下文中就是 `.d` 文件）如果它已经存在，`-f` 选项表示强制删除而不提示。
3. `$(CC) -M $(CPPFLAGS) $< > $@.$$$$;`：
   - 使用编译器 `$(CC)` 来生成依赖关系。其中 `-M` 选项使得编译器输出源文件的依赖信息。
   - `$(CPPFLAGS)` 是编译器的附加选项。
   - `$<` 是第一个前提条件（在这个情况下就是 `.c` 文件）。
   - `> $@.$$$$` 将输出重定向到一个临时文件（通常是以 `.$$` 开头的文件名，`$$` 在 Makefile 中表示当前进程的 PID，能保证生成的文件名唯一）。
4. `sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@;`：
   - `sed` 命令用于处理临时文件的内容。
   - `s,\($*\)\.o[ :]*,\1.o $@ : ,g` 这条命令是对依赖关系格式的一次转换。具体来说，它会把原来的 `.o` 文件依赖的格式替换为 Makefile 的目标依赖格式，使得生成的 `.d` 文件可以直接被 Makefile 解析。
5. `rm -f $@.$$$$`：
   - 删除临时文件。

通过这一模式规则，编译器生成的依赖中会加入`.d`文件的依赖，使得`.d`文件也会自动生成与更新。

### 3. 书写Makefile命令

#### 3.1 关于命令

在Makefile中所有以`tab`缩进开头的语句会被认为是一个命令(即使是空行)，注释使用`#`

#### 3.2 显示命令

通常，make会将要执行的命令行打印出来(在命令执行之前)，如果在命令行前加入`@`符号，那么就不会打印这一命令

例如，不加入`@`符号时，语句对应的输出信息

```bash
echo compiling foo.c

output:
$ echo compiling foo.c
$ compiling foo.c
```

加入`@`符号后的输出信息

```bash
@echo compiling foo.c

output:
$ compiling foo.c
```

如果在make时加入参数`-n`或者`--just-print`，那么make只会显示命令，不会执行命令，有助于检查命令的执行顺序。

而make参数`-s`、`--silent`、`--quiet`则是全面禁止命令的显示。

#### 3.3 执行命令

命令在换行后，执行结果是不继承的，如果想要下一条命令继承上一条命令的结果，要卸载同一行并用`;`分割。

例如，如果我们要进入到指定的路径并打印相应的路径信息。

- <font color = red>错误示例</font>

  ```bash
  exec : 
  	cd /home/usr
  	pwd
  ```

- <font color = green>正确示例</font>

  ```bash
  exec :
  	cd /home/usr; pwd
  ```

在第二个示例中才会正确打印出进入的路径信息。

#### 3.4 命令出错

默认规则下，一条命令出错，整个规则就会被终止执行，但一些情况下(比如mkdir)命令出错并不代表有问题，因此需要忽略这类错误。

方法就是在命令前加`-`

```makefile
clean :
	-rm -f *.o
```

此外，也可以给make加上参数`-i`或者`--ignore-errors`，这样Makefile就会忽略所有命令的错误。

如果想要区分不同的规则，只忽略指定的规则，可以使用`.IGNORE`作为目标，例如

```makefile
.IGNORE : cleanobj
cleanobj :
	rm -f *.o
cleandiff :
	rm -f *.diff
```

此外，参数`-k`或者`--keep-going`可以在某条规则中的命令出错后，终止该规则的执行，同时继续执行其他规则。

#### 3.5 定义命令包

在Makefile中经常出现相同的命令序列，这时候就可以定义一个变量来代替这些相同的命令序列。

这些命令包以`define`开始，以`endef`结束

```makefile
define run-yacc
yacc $(firstname $^)
mv y.tab.c $@
endef
```

首先第一行运行Yacc程序并生成固定的y.tab.c文件

第二行就单纯是对文件改名，比如在如下的例子中

```makefile
foo.c : foo.y
	$(run-yacc)
```

此时上述命令包中的`$^`就代表`foo.y`，`$@`则代表`foo.c`

### 4.变量

#### 4.1 变量的声明与使用

变量是大小写敏感的，推荐使用驼峰法

变量中不能含有`:` `#` `=`和空字符

声明变量时要给予初值

```makefile
objects = program.o foo.o utils.o
```

使用时，需要用`()`或者`{}`将变量包裹起来，这样使用变量更加安全和清晰

```makefile
program : $(objects)
	cc -o program $(objects)
	
$(objects) : defs.h
```

实际上变量的作用激励就是在使用处展开和替换

#### 4.2 变量中的变量

在Makefile中，可以使用变量来构造变量的值，一共有两种方式

<font color = red>递归的定义方式</font>

```makefile
foo = $(bar)
bar = $(dog)
dog = wof
```

好处是可以把变量的真实值放到后面去定义，坏处就是这种递归形式的定义可能会导致死循环

<font color = skyblue>更好的定义方式</font>

```makefile
x := foo
y := $(x) bar
x := later
```

这种定义方式等价于

```
y := foo bar
x := later
```

相比上述递归的定义，这种方式定义的变量必须被先声明才能使用，因此可以避免上述的问题

下面是一个例子，其中`MAKELEVEL`表示嵌套执行make时后当前Makefile的调用层数

```makefile
ifeq (0, ${MAKELEVEL})
cur-dir := $(shell pwd)
whoami := $(shell whoami)
host-type := $(shell arch)
MAKE := ${MAKE} host-type=${host-type} whoami=${whoami}
endif
```

注意，在定义变量的时候，注释符`#`代表着变量定义的终止，也就是说

```makefile
nullStr := 
spaceChar := $(nullStr) #end of the line
```

这种定义下`nullStr`是一个空变量，`spaceChar`的定义中，因为`$(nullStr)`和`#`有一个空格，此时`spaceChar`变量就相当于一个空格了。

再举一个例子

```makefile
dir := /usr/bin    # dir of bin
```

这种情况下`bin`后面还会带有四个空格，在使用`$(dir)/clang`之类的指定操作时会造成错误。

此外一个比较有用的操作符是`?=`

```
var ?= newDef
```

这条语句的含义是如果`var`之前没有被定义，那么`var`就会被定义为`newDef`

如果`var`之前被定义过了，那么这条语句不会执行任何操作

#### 4.3 变量替换

通过`$(var:a=b)`可以把`var`中所有以字符串`a`结尾的部分换成字符串`b`，例如

```makefile
foo := a.o b.o c.o
bar := $(foo:%.o=%.c)
```

注意这里的模式中必须要有`%`来进行匹配。

#### 4.4 将变量值作为变量

这个特性比较绕，下面是一个例子

```makefile
kiana_kaslana = k423
fn = kiana
ln = kaslana
id = $($(fn)_$(ln))
```

此时`$(id)`就是`k423`

#### 4.5 追加变量值

通过`+=`来给变量追加值，比如

```makefile
foo = x.c y.c
foo += z.c
```

#### 4.6 override指令

如果一个变量的值是通过make中的命令行参数获取，那么Makefile中该变量的赋值会被忽略。

通过override指令，可以在Makefile中指定这些参数的值，比如

```makefile
override <var>; := <val>;
```

#### 4.7 define指令

define可以用于定义一系列的操作，形成一个命令包，例如

```makefile
define printVar
echo foo
echo $(bar)
endef
```

 #### 4.8 目标变量Target-specific Variable

在Makefile中可以为某个目标设置局部变量，这样该变量的作用范围只会限制在这条规则以及连带规则中，而不会影响其他规则中的变量的值，因而可以出现同名，例如

```makefile
prog : CFLAGS = -g
prog : prog.o foo.o bar.o
	$(CC) $(CFLAGS) prog.o foo.o bar.o
	
prog.o : prog.c
	$(CC) $(CFLAGS) prog.c

foo.o : foo.c
	$(CC) $(CFLAGS)	foo.c

bar.o : bar.c
	$(CC) $(CFLAGS) bar.c
```

在这个例子中，不管全局的`CFLAGS`定义为了什么值，在`prog`对应的规则中它就是`-g`

#### 4.9 模式变量Pattern-specific Variable

类似于目标变量，模式变量是针对符合某种模式的所有目标上，例如

```makefile
# Generic rule for .c files  
%.o: %.c  
    CFLAGS = -O2          # This CFLAGS applies only to .c to .o conversion  
    $(CC) $(CFLAGS) -c $< -o $@  

# Generic rule for .cpp files  
%.o: %.cpp  
    CXXFLAGS = -O3       # This CXXFLAGS applies only to .cpp to .o conversion  
    $(CXX) $(CXXFLAGS) -c $< -o $@  
```

在这种情况下，`$(CFLAGS)`就只针对第一类模式(即编译.c文件)起作用，而`$(CXXFLAGS)`则只针对第二类模式(即编译.cpp文件)起作用。

### 5. 条件判断

#### 5.1 An Example

```makefile
ifeq ($(CC), gcc)
	libs = $(gcc_libs)
else
	libs = $(norm_libs)
endif

foo : $(objects)
	$(CC) -o foo $(objects) $(libs)
```

在这个例子中，通过条件判断`$(CC)`变量的值是否是`gcc`，从而决定下面的规则中使用哪种库。

除了上面的括号形式的表达，条件语句也可以写为

```makefile
ifeq "arg1" "arg2"
ifeq 'arg1' 'arg2'
```

同时`ifneq`、`ifdef`和`ifndef`也是条件关键字，例如

```makefile
ifneq (arg1, arg2)
	<do sth>
endif 

ifdef foo	#check if foo has values
	<do sth>
endif

ifndef foo
	<do sth>
endif
```

注意，make是在读取Makefile时就计算条件表达式的值并根据相应的值来选取语句，因此，条件表达式中不要用`$@`之类的自动化变量，因为这些变量在运行时才会有。

### 6. 函数

#### 6.1 函数的调用

函数调用的语法为`$(<function> <arg1>,<arg2>,<arg...>)`，或者把`()`替换为`{}`也可以。

其中`<function>`指函数名，后面是参数。函数和参数之间间隔一个空格，参数之间用`,`分隔。

例如

```makefile
comma := ,
empty := 
space := $(empty) $(empty)
foo := a b c
bar := $(subst $(space),$(comma),$(foo))
```

这个函数的作用是把`$(foo)`中的空格换成`,`

#### 6.2 字符串处理函数

> ##### subst 字符串替换函数

```makefile
$(subst <str_old>,<str_new>,<text>)
```

将`<text>`中的`<str_old>`替换为`<str_new>`



> ##### patsubst 模式字符串替换函数

```makefile
$(subst <pattern>,<replace>,<text>)
```

查找`<text>`中所有满足模式`<pattern>`的项，用`<replace>`替换

**e.g.**

```makefile
$(patsubst %.c,%.o,x.c.c bar.c)
```

替换后的结果即为

```makefile
x.c.o bar.o
```



> ##### strip 去除空格函数

```makefile
$(strip <str>)
```

去除`<str>`中开头和结尾的空格

**e.g.**

```makefile
$(strip a b c  )
```

返回值为`a b c`



> ##### findstring 查找字符串函数

```makefile
$(findstring <tar_str>,<text>)
```

在`<text>`中寻找`tar_str`，找到的话返回`<tar_str>`，否则返回空串。

**e.g.**

```makefile
$(findstring a,a b c)	# output : a
$(findstring a,b c)		# output : 
```



> ##### filter 过滤函数

```makefile
$(filter <pattern...>,<text>)
```

返回`<text>`中符合保留模式`<pattern>`的字串

**e.g.**

```makefile
sources := foo.c bar.c baz.s ugh.h
foo : $(sources)
	cc $(filter %.c %.s,$(sources)) -o foo
```

在这个例子中，函数筛选出所有后缀为`.c`和`.s`的文件(即`foo.c bar.c baz.s`)



> ##### filter-out 反过滤函数

```makefile
$(filter-out <pattern...>,<text>)
```

返回`<text>`中不满足模式`<pattern>`的字串

**e.g.**

```makefile
objects := main.o foo.o bar.o
mains := main.o
$(filter-out $(mains),$(objects)) #return foo.o bar.o
```



> ##### sort 排序函数

```makefile
$(sort <list>)
```

给`<list>`中的字串按照字典序升序排列，同时会去重

**e.g.**

```makefile
$(sort ba cd aa)	# return aa ba cd
```



> ##### word 取词函数

```makefile
$(word <n>,<text>)
```

从`<text>`中取出第`n`个单词(范围为1...n)

**e.g.**

```makefile
$(word 2, foo bar baz) # return bar
```



> ##### wordlist 取单词串函数

```makefile
$(wordlist <st>,<ed>,<text>)
```

从单词串中取得`[st, ed]`区间的单词串

**e.g.**

```makefile
$(wordlist 2,3,foo bar baz) # return bar baz
```



> ##### words 单词数统计函数

```makefile
$(words <text>)
```

统计`<text>`中的单词数目

**e.g.**

```makefile
$(words foo bar baz) # return 3
```



> ##### firstword 取首词函数

```makefile
$(firstword <text>)
```

返回`<text>`中的第一个单词



#### 6.3 文件名操作函数

> **dir 取目录函数**

```makefile
$(dir <pathName...>)
```

从`<pathName>`中取出目录部分，即第一个反斜杠之前的部分，如果没有，返回`./`

**e.g.**

```makefile
$(dir src/foo.c hacks)	#return src/ ./
```



> **notdir 取文件函数**

```makefile
$(notdir <pathName...>)
```

从`<pathName>`中取出非目录部分，即最后一个反斜杠之后的部分

**e.g.**

```makefile
$(notdir src/foo.c hacks) #return foo.c hacks
```



> **suffix 取后缀函数**

```makefile
$(suffix <pathName...>)
```

从`<pathName>`中取出各个文件名的后缀，如果没有，返回空字符串

**e.g.**

```makefile
$(suffix src/foo.c src-1.0/bar.c hacks) #return .c .c
```



> **basename 取前缀函数**

**e.g.**

```makefile
$(basename src/foo.c src-1.0/bar.c hacks) #return src/foo src-1.0/bar hacks
```

*ps.其实比较奇怪的是这个不是叫prefix而是叫basename*



> **addsuffix 加后缀函数**

**e.g.**

```makefile
$(addsuffix .c,foo bar) #return foo.c bar.c
```



> **addprefix 加前缀函数**

**e.g.**

```makefile
$(addprefix src/,foo bar) #return src/foo src/bar
```



> **join 连接函数**

```makefile
$(join <list1>,<list2>)
```

将`<list2>`中的单词加到`<list1>`单词的后面

- 如果`<list1>`中的单词更多，则多出来的单词会保持原样
- 如果`<list2>`中的单词更多，则这单词会连接到一个空字符串上

**e.g.**

```makefile
$(join aaa bbb,111 222 333) #return aaa111 bbb222 333
```



#### 6.4 foreach函数

```makefile
$(foreach <var>,<list>,<text>)
```

该函数会将`<list>`中的单词逐一取出放入`<var>`指定的变量中，然后执行`<text>`所包含的表达式。每一次`<text>`会返回一个字符串，返回过程中`<text>`所返回的每个字符串会以空格分隔，最后`<text>`中的字符串就是该函数的返回值。

**e.g.**

```makefile
names := a b c d

files := $(foreach n,$(names),$(n).o)

# $(files) = a.o b.o c.o d.o
```

在foreach函数中，`<var>`只是一个局部变量，其作用域只在该函数内。



#### 6.5 if函数

```makefile
$(if <condition>,<then-part>)

$(if <condition>,<then-part>,<else-part>)
```

- `<condition>`为真，函数返回值为`<then-part>`
- `<condition>`为假或者空字符串，函数返回值为`<else-part>`



#### 6.6 call函数

```makefile
$(call <expression>,<para1>,<para2>,...)
```

该函数会将`<expression>`参数中的变量比如`$(1) $(2)`等用参数`<para1> <para2>`依次取代，最终`<expression>`的返回值就是`call`函数的返回值。

**e.g.**

```makefile
reverse = $(2) $(1)

foo = $(call reverse,a,b)

# return foo = b a
```

注意，call在处理参数时，第二个及其后面参数中的空格会被保留，因此在使用时，最好去掉所有多余的空格以避免问题。



#### 6.7 origin函数

```makefile
$(origin <varName>)
```

**注意！<varName\>是变量的名字，而不是引用，因此不要使用\$字符**

origin用于判断一个变量的定义与类型情况，返回值有如下这些

```makefile
default 		#默认定义，例如CC
undefined 		#未定义过
environment 	#环境变量，且Makefile执行时未使用-e参数
file 			#该变量被定义在Makefile中
command line	#该变量由命令行来定义
override 		#该变量是被override指示符重新定义的
automatic		#自动化变量
```

一个例子，根据bletch是否是环境变量来决定是否对其进行重定义操作

```makefile
ifdef bletch
    ifeq "$(origin bletch)" "environment"
        bletch = barf, gag, etc.
    endif
endif
```



#### 6.8 shell函数

shell函数的作用就是在makefile中执行shell命令，与反引号“ ` ”的作用相同，例如

```makefile
contents := $(shell cat foo)
file := $(shell echo *.c)
```

但是执行这个函数会生成一个新的shell程序来执行命令，因此不要大量使用这个函数，会有损性能。



#### 6.9 make控制函数error & warning

```makefile
$(error <err_info>)
```

产生一个fatal error并输出错误信息`<err_info>`，该函数可以被定义在变量里，例如

```makefile
ifdef ERR_01
	$(error Fatal error : $(ERR_01))
endif
```

```makefile
$(warning <warning_info>)
```

该函数会输出一段警告信息，但不会像`error`那样终止Makefile运行。

















