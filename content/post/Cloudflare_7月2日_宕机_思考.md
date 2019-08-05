---
title: "Cloudflare_7月2日_宕机_思考"
author: "wangyuan"
date: '2019-08-05'
slug: Cloudflare
categories: ["技术"]
tags:  ["Cloudflare", "正则表达式", "宕机"]
---
上个月Cloudflare发生了一次宕机，原因后来官方给出了[解释](https://new.blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/#appendix-about-regular-expression-backtracking)与[中文解释](https://blog.cloudflare.com/zh/details-of-the-cloudflare-outage-on-july-2-2019-zh) 。这件事与失事一样，并不是单一问题导致的，而是一连串原因导致保险措施失效。
作为中断核心诱因的正则表达式是 
```
(?:(?:\"|'|\]|\}|\\|\d|(?:nan|infinity|true|false|null|undefined|symbol|math)|\`|\-|\+)+[)]*;?((?:\s|-|~|!|{}|\|\||\+)*.*(?:.*=.*)))
```
核心代码是
```
.*(?:.*=.*)
```
(?:和)表示它们中间是一个非捕获组，意思是两个括号间的内容需要匹配，但在匹配之后会被扔掉。由于我们关注的是匹配的过程，此处的(?:和)也可以被忽略，最终化简后的表达式是
```
.*.*=.*
```
它从表面看上去非常简单，.表示匹配一个任意字符，.*即表示匹配零个或多个任意字符。因此上述表达式先匹配零个到多个任意字符，然后又匹配零个到多个任意字符，再匹配一个等于符号=，最后再匹配零个到多个任意字符。如下图所示。

![CloudflareNFA01](https://raw.githubusercontent.com/reticentfat/wangyuanfrank.com/master/static/images/CloudflareNFA01.png)

你可以到 [Perl Regexp::Debugger](https://metacpan.org/pod/Regexp::Debugger) 直观地查看正则表达式引擎如何回溯。下图展示了x=x的匹配过程。

![backtrackingSteps](https://raw.githubusercontent.com/reticentfat/wangyuanfrank.com/master/static/images/backtrackingSteps.gif)

如果我们将正则表达式末尾添加一个分号，改写为.*.*=.*;，或申明懒惰匹配的写法.*?.*?=.*?;，两者在处理匹配失败的字符串时所需要的步数是相同的，如x=x都需要耗费 90 步才能知道匹配失败，而x=后面有 20 个 x时，从原来 555 步增加到了惊人的 5353 步。

下图展示了引擎是如何走了 5353 步才发现x=xxxxxxxxxxxxxxxxxxxx匹配失败的。
![5353Steps](https://raw.githubusercontent.com/reticentfat/wangyuanfrank.com/master/static/images/5353.gif)
这种正则在匹配的时候会造成太多回溯使时间复杂度飙升。解决方案是把正则转换成 NFA （非确定有限自动机）这样时间复杂度就是对于输入数据线性的。
