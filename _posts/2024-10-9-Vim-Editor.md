---
layout: 	post  				   
title: 		Vim Editor  				
subtitle:	from missing semester(MIT)
date:       2024-10-9 				
author:     Sirin 						
header-img: img/universe.jpg
catalog: 	true 				
tags:						
    - Vim
    - Tools
---

## Vim

### Intro

vim由四种模式组成

- **Normal Mode**

默认进入的模式，在其他模式下按`esc`就会回到normal mode

- **Insert Mode**

在Normal Mode下输入`i`来进入

- **Replace Mode**

用于对文本进行替换，Normal mode下输入`r`来进入

- **Visual Mode**

包括了plain, line, block三种子模式，用于选择相应的文本内容

按`v`进入plain，`shift+v`进入line，`ctrl+v`进入block

比较特殊的是**Command-line**模式，输入`:`来进入，用于执行相应的命令

### 相关操作

Vim本身就自带了`vimtutuor`用于练习和学习，因此这里只简单记录各种操作对应的方法

**上下左右移动光标**	`hjkl`

```
	^
	k
< h	  l >	
	j
	v
```

**保存文件**	`w`

**退出文件(丢弃修改)**	`q`

**强制退出**	`q!`

**保存修改并退出**	`wq`

**删除单个字符**	`x`

**进入insert mode**	`i`

**在当前位置后添加文本**	`a`

**删除从光标处开始的单词后续部分**	`dw`

**移动到行首**	`0`

**移动到本行首个非空字符**	`^`

**移动到行末**	`$`

**删除从光标处到行末的所有文本**	`d$`(由此类推其他位置上的删除操作)

**移动到下一个单词**	`w`

**移动到单词末尾**	`e`(end of word)

**移动到单词的开头**	`b`(beginning of word)

**在动作前输入数字来指定执行次数**	`<num><motion>`

```
e.g.
2e	4j	5k
```

**连续删除多个单词**	`d<num><motion>`

```
e.g.
d2w	d4w d2e
```

**删除整行**	`dd`

**删除多行**	`<num>dd`

**撤销一次操作**	`u`(undo)

**撤销所有操作让被修改的文本回到最初状态**	`shift+u`

**恢复操作**	`ctrl+r`

**寄存器粘贴**	`p`(paste)

```
这个需要展开讲一下，执行删除操作的时候，被删除的内容会被存在vim的寄存器里方便后续恢复。
使用p操作就可以把寄存器里的内容粘贴上去。
注意，dd删除得到的整行内容，在粘贴时光标应该位于想要粘贴位置的上方一行。
即，此时粘贴会导致换行。
```

**单字符替换**	`r<char>`

```
将光标处的字符替换为<char>
```

**修改内容**	`c<num><motion>`

```
例如cw就是修改光标后的词语部分，也就是先删除再顺便切换为insert模式
```

**查看当前信息**	`ctrl+g`

**跳转文档最后一行**	`G`

**跳转文档第一行**	`gg`

**跳转指定行**	`<num>G`

**跳转回之前位置**	`ctrl+o`

**跳转回新的位置**	`ctrl+i`

**查找某个字符串**	`/<char>`

```
该查找方式默认从上到下
输入n来顺序查找下一个，输入shift+n来回到上一个
如果想逆序查找，从下到上，用?替换/
```

**查找配对符号()[]{}**	`%`

```
需要将光标移动到要查找配对的符号上
```

**替换命令**	`s/<old_str>/<new_str>`

```
该命令会将整行中的第一个匹配<old_str>的字符串替换为<new_str>

直接替换一整行的所有匹配串
s/<old_str>/<new_str>/g

替换行号#line-h和#line-t之间(包括这两行)的所有匹配串
<#line-h>,<#line-t>s/<old_str>/<new_str>/g

替换整个文件中的匹配串
%s/<old_str>/<new_str>/g

找到整个文件中所有的匹配串并依次询问是否要修改
%s/<old_str>/<new_str>/gc
```

**在vim内执行外部shell命令**	`:!<command>`

**另存当前文件**	`:w <filename>`

**选取文本**	`v`,按行选取`shift+v`，按块选取`ctrl+v`

**输入文件、命令内容**	`r <filename>`	`r !<command>`

**在某行下方新起一行**	`o`

**在某行上方新起一行**	`shift+o`

**替换文本**	`shift+r`

```
输入后进入替换模式，此时输入会直接替换掉光标处的文本
```

**复制选中的文本到buffer中**	`y`

```
同样的，复制支持yw ye yb y$这样的操作
```

**粘贴buffer中的文本**	`p`

**查找文本的相关操作**	

```
/abc可以用于寻找该词位置，使用n和N来前后移动光标，但通常来说这是大小写敏感的
通过输入:set ic
即Ignore case来忽略大小写
:set noic可以禁用这种忽略

:set hls is可以高亮匹配项
:set nohlsearch关闭高亮
```

**启动帮助系统**	`:help`

```
使用ctrl+w来在窗口之间移动
输入:q来退出
:help <Arg>来获取对于特定参数的帮助文档
```

**关闭Vim兼容模式以使用命令补全**	`:set nocp`

```
ctrl+d来查看一个命令的所有补全
tab来补全命令
```

**快速移动**	`ctrl+u`(up)	`ctrl+d`(down)

**快速到达当前窗口的位置**	顶部`shift+h`  中间`shift+m`  底部`shift+l`

**在本行快速查找**

```
从前到后快速到某个字符位置f<char>	(find)
从后向前快速到某个字符位置F<char>

从前到后快速到某个字符旁边位置t<char>		(till)
从后向前快速到某个字符旁边位置T<char>	

使用 , ; 来左右移动查找匹配项
```

**快速切换当前字符大小写**	`~`









