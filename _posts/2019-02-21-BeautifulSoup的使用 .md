---
layout:     post
title:      BeautifulSoup的使用
subtitle:   Python抓包
date:       2019-02-21
author:     祝化林
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Python
    - 爬虫
---


### BeautifulSoup 
#### 简介
`BeautifulSoup` 库的名字取自刘易斯 ·卡罗尔在《爱丽丝梦游仙境》里的同名诗歌，`BeautifulSoup `尝试化平淡为神奇。它通过定位` HTML` 标签来
格式化和组织复杂的网络信息，用简单易用的 `Python` 对象为我们展现 `XML` 结构信息。

#### 安装
用PyCharm，创建的项目，自带独立的虚拟环境，在Terminal，中直接安装
```
pip install beautifulsoup4
```
#### 引入
```
from bs4 import BeautifulSoup
```

#### 常用的函数

##### beautifulSoup 初始化
```
html = requests.get("http://www.pythonscraping.com/pages/warandpeace.html")
print(html)
bsObj = BeautifulSoup(html.content,'lxml')
```
`html`对象是一个`response`类，需要用`html.content`初始化，，如果是用，urllib,要用`html.read()`

##### findAll函数
`findAll` 函数为最常用的函数，定义如下,返回值为一个`<class 'bs4.element.ResultSet'>`，`ResultSet`是一个`list`,
>"""A ResultSet is just a list that keeps track of the SoupStrainerthat created it."""
    
```
 def find_all(self, name=None, attrs={}, recursive=True, text=None,
                 limit=None, **kwargs):
```
name:为要查找的tag的名字，比如是span标签

```
nameList = bsObj.findAll("span", {"class":"green"})
```

attrs:是一个`tuple`,里面是限制附加条件，上面的方法表示的是查找`html`中`span`标签，且 `class`是`green` 的`sapn`标签。
如果要查找多个标签，可以

```
.findAll({"h1","h2","h3","h4","h5","h6"})
```

指定`TagName`多个限制条件

```
 .findAll("span", {"class":{"green", "red"}})
```
recursive:递归参数，默认是True
>如果 recursive 设置为 True，findAll 就会根据你的要求去查找标签参数的所有子标签，以及子 标签的子标签。如果 recursive 设置为 False，findAll 就只查找文档的一级标签。findAll 默认是支持递归查找的(recursive 默认值是 True);一般情况下这个参数不需要设置，除 非你真正了解自己需要哪些信息，而且抓取速度非常重要，那时你可以设置递归参数

text:文本标签，用标签的文本内容去匹配，比如要查，张三，出现了多少次

```
nameList = bsObj.findAll(text="张三") print(len(nameList))
```
limit:范围限制，只对获取的前几个标签感兴趣，

```
nameList = bsObj.findAll("span", {"class":"green"},limit=4)
```
keyword:选择那些具有指定属性的标签

```
allText = bsObj.findAll(id="text")
print(allText[0].get_text())
```
另外，keyword和attrs ,这两个是等价的

```
 bsObj.findAll(id="text")
 bsObj.findAll("", {"id":"text"})
```
##### find()函数

函数如下

```
find(tag, attributes, recursive, text, keywords)
```
`find`和`findAll`用法基本相同，`find`是`findAll`,`limit`为1的情况

##### 属性直接调用
比如你要获取`h1`标签的内容，可以如下直接调用

```
bsobj.h1 
```
##### get_text()
出去标签，直接获取标签里的文本内容


#### 几个常用的概念
标签与标签之间有，子标签（`.children`），后代标签(`.descendants`)，兄弟标签(`siblings`)，父标签(`parent`)
##### 子标签和 后代标签，
子标签是下级标签，比如,table的子标签，是两个<tr>标签，下一级。table的后代标签，是<tr>以及下面的<th>

```
<table>
    <tr>
        <th>
        <th>
    <tr>
        <th>
        <th>
</table>
```
`BeautifulSoup`默认处理的是，后代标签，

```
bsObj.body.h1 
```
这是是处理，`bsObj.body`标签后代里的第一个`h1`标签

```
bsObj.div.findAll("img")
```
这个是处理，第一个`div`标签下面的所有的`img`标签
##### 兄弟标签
兄弟标签是和自己同级的，但不包含自己的标签，
常用的函数有两个
`next_sibling` 和 `previous_siblings`

```
for sibling in bsObj.find("table",{"id":"giftList"}).tr.next_siblings: print(sibling)
```
会获取，table 标签下，tr标签后的，所有兄弟标签


