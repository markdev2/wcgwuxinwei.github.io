---
layout:     post
title:      "Shell脚本的简单编写以及sed的使用"
subtitle:   "Shell"
date:       2015-04-01 13:00::00
author:     "MarkWoo"
header-img: "img/home-bg.jpg"
---

前一阵子为了批量修改Web审计规则，故编写了一个`Shell`脚本，顺便使用了下`sed`，顺便把`正则表达式`也重新学习一遍，感觉还是需要总结下，不然对不起自己。

---
# Shell
## 变量
- shell的变量**很弱**,无需定义任何类型，
- 变量在赋值时，等号`=`两边必须不留任何`空格`，
- 变量在使用时可以使用`$`开头使用

## if条件判断
首先看代码

```bash

if [ ! -e "$website_dir" -o ! -e "$weblogin_dir" ]
then
	echo "$website_dir 不存在"
	echo "$weblogin_dir 不存在"
else
    ...
fi

```
    
- 这里需要重点指出一些格式问题，初学者比较容易碰到的，`if`,`then`,`else`必须**单独一行，如果想同一行请用**`;`**隔开**，不然会报错，再者，`if`后面的条件框`[]`，在两端必须留有**空格**，每次一个判断选项，和一个逻辑符号之间必须**留一个空格**，最后`fi`结尾
- `if`条件中的各种选项可以从其他搜索引擎中找到

## case条件选择

```bash
case $1 in
    replace)
        ...
        exit 1;;
    restore)
        ...
        exit 1;;
    *)
        echo "replace: 备份现有规则文件并替换规则文件"
        echo "restore: 恢复规则文件";;
esac
```

- `$1`指的是选择运行时的第一个输入参数，这里的输入参数指在terminal中输出的，这里固定`$0：运行脚本本本身文件名`，`$1：为其后的第一个参数`

# Sed
## Sed简介  
sed 是一种在线编辑器，它一次处理一行内容。处理时，把当前处理的行存储在临时缓冲区中，称为“模式空间”（pattern space），接着用sed命令处理缓冲区中的内容，处理完成后，把缓冲区的内容送往屏幕。接着处理下一行，这样不断重复，直到文件末尾。文件内容并没有 改变，除非你使用重定向存储输出。Sed主要用来自动编辑一个或多个文件；简化对文件的反复操作；编写转换程序等。**sed适合进行文本行的处理**

## 结合实例使用
首先sed的使用格式网络上都有比我详细的教程，各位可以随意google，这里我只专门将下我实际中遇到的一些比较棘手的问题

```bash
sed -i "/^[^#SUB].*WEBFORUM_/{s/\(.*\)CONTENT/\1REFERER=H24@P(7::),CONTENT/g}" $website_dir
```

- 这条语句的功能是：在一个文本行中，找到包含**WEBFORUM**但是不以`#`,`S`,`U`,`B`开头的文本行，r然后通过正则表达式中的backreferences方式替换`CONTENT`为`REFERER=H24@P(7::)`。这条语句中`sed`后面的`-i`选项表示在当前文本中替换，`{s/.../g}`这里加括号的意思表示这里是一条单独的sed语句，实际上整条规则去掉`{}`也是正确的，这里这样写是为了查看方便理解语义
- `{s/\(.*\)CONTENT/\1REFERER=H24@P(7::),CONTENT`在这条正则表达式中，`\(.*\)`表示任意文本，`\1`表示替换第一个匹配的文本（即CONTENT），具体backreferences的使用请参考_Classic Shell Scripting_的_Regular Expressions_章节

```bash
sed -i "/^[^#SUB].*WEBFORUM_/{s/$/;[COMPOSE]=URL=REFERER/g}" $website_dir
sed -i "s/^M//g" $website_dir
```

- 这条语句的功能是：在一个文本行中，找到包含**WEBFORUM**但是不以`#`,`S`,`U`,`B`开头的文本行，在行末尾添加`;[COMPOSE]=URL=REFERER`,`$`在这里表示行尾，这里有一个值得注意的问题，当只执行第一句时，末尾结束时会多出一个`^M`符号,这个是在`windows`下的一个换行符,由于拷贝过程中经过了`windows`，所以这个符号就存在了，但是这个符号会影响这个规则文件的解析，所以必须去掉

---
# 完整代码
- 由于涉及到一些比较敏感的东西，路径一律用`xxx`来表示

```bash
#########################################################################
# File Name: replace_web_site_rule.sh
# Author: MarkWoo
# mail: wcgwuxinwei@gmail.com
# Created Time: 2015年03月24日 星期二 09时59分58秒
#########################################################################
#!/bin/bash

website_dir='XXX/WebSite.rc'
backup_website_dir='XXX/WebSite.rc.bak'
weblogin_dir='XXX/weblogin_site.rc'
backup_weblogin_dir='XXX/weblogin_site.rc.bak'

if [ ! -e "$website_dir" -o ! -e "$weblogin_dir" ]
then
	echo "$website_dir 不存在"
	echo "$weblogin_dir 不存在"
else
	case $1 in
		replace)
			echo "正在备份原规则文件"
			touch $backup_website_dir
			touch $backup_weblogin_dir
			cat $website_dir > $backup_website_dir
			cat $weblogin_dir > $backup_weblogin_dir
			echo "正在进行规则替换"
			sed -i "/^[^#SUB].*WEBFORUM_/{s/\(.*\)CONTENT/\1REFERER=H24@P(7::),CONTENT/g}" $website_dir
			sed -i "/^[^#SUB].*WEBFORUM_/{s/$/;[COMPOSE]=URL=REFERER/g}" $website_dir
			sed -i "s/^M//g" $website_dir
			sed -i "/^[^#SUB].*WEBFORUM_/{s/\(.*\)CONTENT/\1REFERER=H24@P(7::),CONTENT/g}" $weblogin_dir
			sed -i "/^[^#SUB].*WEBFORUM_/{s/$/;[COMPOSE]=URL=REFERER/g}" $weblogin_dir
			sed -i "s/^M//g" $weblogin_dir
			exit 1;;
		restore)
			if [ ! -e "$backup_website_dir" -o ! -e "$backup_weblogin_dir" ]
			then
				echo "找不到备份文件"
			else
				echo "正在恢复原始规则文件"
				cat	$backup_website_dir > $website_dir
				cat $backup_weblogin_dir > $weblogin_dir
			fi
			exit 1;;
		*)
			echo "replace: 备份现有规则文件并替换规则文件"
			echo "restore: 恢复规则文件";;
	esac
fi
```

# 最后的总结
首先`正则表达式`是一个很强大的工具，对于有规律的文本要进行处理，这个是个极好的辅助工具，sed对于一行一行的文本处理极为方便

# 参考资料
- _Classic Shell Scripting_, Aronld Robbins, Nelson H.F.Beebe O'REILLY Media,Inc
