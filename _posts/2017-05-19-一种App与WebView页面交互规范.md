---
layout: post
title: 一种App与WebView页面交互规范
---
# {{ page.title }}

为了加快业务需求的开发进度，技术选型上往往选择H5作为业务需求的承载，如此不可避免的需要支持App原生代码与页面JavaScript脚本的相互调用。

本文设计了一种接口调用的规范，有效简化业务各方的沟通和联调成本，仅供参考。

规范具体如下：

 1、App在WebView注入一个对象window.appJavaScriptBridge。

 2、页面JavaScript调用Native

window.appJavaScriptBridge.callNative(command, argument);
如果需要App回调，需将callback函数挂在window.appJavaScriptBridge，函数名自定义，如callbackName，并在argument中添加字段'callback':'callbackName'；
其中callback函数定义为 function callbackName(command, argument)；

 3、Native调用页面JavaScript

 window.appJavaScriptBridge.callbackName(command, argument);

 4、参数说明

 command为 String，标记需要调用的方法；
 argument为 String，内容为JSON对象，调用方法需要的参数；App回调时为JSON对象；

 5、版本兼容

如果command不支持，则回调页面，command原样返回，argument为{'errorCode':100,'errorMessage':'unknown command'};

代码实现请点击阅读原文或访问以下网址：
https://github.com/linzhiman/ATWebViewJavaScriptBridge
 

