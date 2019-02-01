---
layout:     post
title:     iOS-LaTex,KaTex的加载
subtitle:  LatexWebView
date:       2019-02-01
author:     祝化林
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    -   iOS
    - 开发技巧
---

## iOS-LaTex,KaTex的加载
### 前言
最近项目里，需要一个控件加载HTML字符串，加载的控件百度上一堆， `YYLabel`，`DTCoreText`等都可以加载，实在不行还可以用`UIWebView`比较厚重的控件来实现。
`YYLabel`,`DTCoreText` 根据文本内容直接可以同步计算出需要的高度,相较于`UIWebView`需要通过`delegate`的方法来实现,实在不要太爽。
然鹅，最后发现 `$Na_{2}CO_{3}$` 这样的格式。不能正确的显示出来，即  **Latex**

### 什么是LaTex？
>LaTeX（LATEX，音译“拉泰赫”）是一种基于ΤΕΧ的排版系统，由美国计算机学家莱斯利·兰伯特（Leslie Lamport）在20世纪80年代初期开发，利用这种格式，即使使用者没有排版和程序设计的知识也可以充分发挥由TeX所提供的强大功能，能在几天，甚至几小时内生成很多具有书籍质量的印刷品。对于生成复杂表格和数学公式，这一点表现得尤为突出。因此它非常适用于生成高印刷质量的科技和数学类文档。这个系统同样适用于生成从简单的信件到完整书籍的所有其他种类的文档

简单点就是新的排版系统，主要用于数学公式等，化学公式等等，作为一个教学类的APP，不免会出现这样的格式。躲是躲不过了！

### 怎么办？
查找资料发现，**Latex**这种格式的文本，无论是iOS还是在Android上，底层都是通过一个Web的方式来渲染。
- 方法1，使用YYLable加载文本，然后在`$Na_{2}CO_{3}$` **LaTex**的地方，通过WebKit,加载出来，整体上是`YYLabel+WKWebView`的方式来实现，`YYLabel`负责加载富文本，`WKWebView`负责加载**LaTex**,如下图![图片](https://raw.githubusercontent.com/MRZHL/LatexWebView/master/LatexExample/WechatIMG5295.jpeg)
外面是`YYLabel` 中间是`WKWebView`,可以参考[SPMathKit](https://github.com/CodingSha/SPMathKit)的实现。
- 方法2，将**LaTex**做成**PDF**，然后加载到LaTex的地方，和方法一基本上雷同。

我采用的是第一种方法，然鹅，出其不意，富文本里面竟然有`table`

### table的加载
无论是`YYLable` 还是`DTCoreText`都无法显示完整的Table,表格视图，最后只能用最笨重的方法实现，使用**UIWebView**来加载**Latex**和**Table**,**Table**可以不添加`script`就能实现,`Latex`需要添加JS，
实现思路，在HTML的字符串中给 **Latex** 加上一个class,然后包在一个HTML网页中，加载这个网页，

### 最后的实现
新建一个类继承于UIWebView
处理字符串中，含有**LaTex**的部门加上class, 

```
-(void)dealWithHTMLString:(NSString *)html{
    NSString* htmlPath = [[NSBundle mainBundle] pathForResource:@"index" ofType:@"html"];
    NSString* appHtml = [NSString stringWithContentsOfFile:htmlPath encoding:NSUTF8StringEncoding error:nil];
    // 是否有$ ，有表示的含有LaTex
    if([html rangeOfString:@"$"].location != NSNotFound) { // Expression inside text.
        NSRange r;
        BOOL intex = NO; // 判断是前面的$还是后面的$，需要一对
        while ((r = [html rangeOfString:@"$"]).location != NSNotFound) {
            html = [html stringByReplacingCharactersInRange:r withString:(intex ? @"</span>" : @"<span class=\"tex\">")];
            intex = !intex;
        }
        if (intex) NSLog(@"没有闭合");
    }
    // 将处理后的字符串，嵌套在index.html中
    appHtml = [appHtml stringByReplacingOccurrencesOfString:@"$LATEX$"
                                                 withString:html];
    NSURL *baseURL = [NSURL fileURLWithPath:htmlPath];
    [self setDelegate:self];
    [self loadHTMLString:appHtml baseURL:baseURL];;
}
```
 
Index.html

```
<!DOCTYPE html>
<!-- saved from url=(0028)http://khan.github.io/KaTeX/ -->
<html class="wf-proximanova-i6-active wf-proximanova-n3-active wf-proximanova-n4-active wf-proximanova-n6-active wf-active"><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<head>
<link rel="stylesheet" type="text/css" href="katex.min.css">
<script src="katex.min.js" type="text/javascript"></script>
</head>

<body>
<div>$LATEX$</div>

<script type="text/javascript">
// 对 clsss = tex 进行处理
window.startup = function() {
	function onload() {
		var texes = document.getElementsByClassName('tex');
        for(var i=0; i<texes.length; i++) {
            katex.render(texes[i].innerHTML, texes[i]);
        }
	}
	onload();
}
startup();
</script>
</body>
</html>
```
以上. 
[DEMO](https://github.com/MRZHL/LatexWebView)



