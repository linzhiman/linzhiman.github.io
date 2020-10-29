---
layout: post
title: Sublime Text使用技巧
---
# {{ page.title }}

[下载传送门](http://www.sublimetext.com/3)

##日志过滤分析

###基础版：

	打开需要处理的文件，按【Ctrl + F】，输入目标文本，选择【Find All】，在文件中你会看到我们想选的内容被选中了

	选择菜单栏的【Selection】的下拉列表中的【Expand Selection to Line】，你会看到我们要查找的数据所在的行被选中

	按【Ctrl + C】拷贝数据，最后新建文件粘贴数据，这样我们所需要的数据就被提取出来了

###进阶版：

	安装插件管理器，【Tools】->【Install Package Control…】

	安装插件，【Tools】->【Command Palette…】-> 输入Install Package并点击进入-> 输入Filter Lines并点击进入

	使用插件，【Tools】->【Command Palette…】-> 输入Filter 选择【Filter Lines: Include Lines With Regex】

	在底部输入框输入目标文本即可

##Markdown预览

	安装 MarkdownPreview 插件

	使用插件，【Tools】->【Command Palette…】-> 输入Filter 选择【Markdown Preview: Preview in Browser】

	选择任一解析器即可

