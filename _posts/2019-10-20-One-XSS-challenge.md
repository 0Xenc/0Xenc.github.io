---
title: 一个XSS challenge 
tags: XSS,Web安全
atthor: Xenc
Date: Oct 20th,2019
grammar_cjkRuby: true
---


我在推特上看到了这样的一个XSS挑战：[Uncomment Me](https://walma.re/uncomment-me/?s=)

它是基于DOM XSS，我们看源代码即可发现
```
<title>Uncomment Me</title>

<h1>Uncomment Me</h1>
<p>XSS challenge by <a href="https://twitter.com/DrStache_">DrStache</a></p>

<script>
	s = new URL(window.location.href).searchParams.get('s');

	if (!s.includes('!') && !s.includes('-')) {
		t = document.createElement('template');
		t.innerHTML = s;
		document.write('<!-- ' + t.innerHTML + ' -->');
	}
</script>
```
很显然这是一个DOM类型的XSS **`document.write('<!-- ' + t.innerHTML + ' -->')`** 将会输出到我们的电脑屏幕上，我们还要负责去绕过一些过滤

它们是：
`!s.includes('!') && !s.includes('-')` 我们不能包含 **!** 、**-** 否则if这个条件语句进不去无法XSS

既然不可以添加，那么我们就使用编码去绕过它！这里还需要一些HTML的标签闭合优先权还有2个解析器(JS,HTML)的知识，我在这将解释HTML的标签闭合优先权，这里引用seebug的文章
>这个特性出现的原因、可能是源于浏览器对DOM树的特殊处理、而在某些XSS攻击的场景下、这一特性可能导致意想不到的结果。

>一些标签的特殊的魔力。他的闭合优先权高于双引号的完整性的优先级、高于嵌套在内层的标签的闭合优先权

可能不好理解，来个演示就知道了

```
<script>
var xss="</script><svg/onload=alert(1)>";
</script>
```
是否能成功XSS我们一探究竟：
![1](http://console-log.cn/img/xss-challenge_1_1.jpg)

浏览器没有将xss变量的值当成数据来看待，而是`</script>`直接闭合了标签使得XSS可以执行，这就是闭合优先级，当然不是所有的标签都有那么强大的优先级，以下是标签的优先级高于引号的优先级。
labels：
>`<!--`
>`<iframe>`
>`<noframes>`
>`<noscript>`
>`<script>`
>`<title>`
>`<style>`
>`<textarea>`
>`<xmp>`

了解到了这些，我们再看XSS challenge，

 1. 不能存在!、-
 2. 我们必须闭合注释才可以执行XSS

那我们利用这个闭合优先级来绕过这个挑战

先把 `-->` 进行实体编码然后放入引号当中，首先要插入歌标签
eg:
`<a a="-->的实体编码">`

实体编码后：&#45;&#45;&#62;
将实体编码转url
变成：`<a a="%26%2345%3B%26%2345%3B%26%2362%3B">` 这个payload将闭合注释！

![2](http://console-log.cn/img/xss-challenge_1_2.jpg)

成功的闭合了注释，然后我们插入xss弹窗：

![3](http://console-log.cn/img/xss-challenge_1_3.jpg)

很不错成功了

实体编码是为了绕过if、url编码是因为实体编码有`#` 浏览器会把它当成瞄（我打不出那么字，语文太差）

注释实体编码为什么会被浏览器解析呢? 

Refer：[https://mp.weixin.qq.com/s/LQweGgjUuBwsXgNQg0RP9A
](https://mp.weixin.qq.com/s/LQweGgjUuBwsXgNQg0RP9A)