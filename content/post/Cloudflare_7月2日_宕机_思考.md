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
> 这种正则在匹配的时候会造成太多回溯使时间复杂度飙升。解决方案是把正则转换成 NFA （非确定有限自动机）这样时间复杂度就是对于输入数据线性的。

#### 真正的解决方法

通过改写正则表达式本身消除回溯的方法并不能一劳永逸地解决问题，由于正则表达式本身可读性差，无法指望人工完全消除隐含的回溯。尽管存在相应检测工具，但其一般靠随机数据暴力穷举，费时费力。要完全消除回溯带来的性能问题需要从正则表达式引擎本身入手。

好在 1968 年，Ken Thompson 发表的一篇论文 [Programming Techniques: Regular expression search algorithm ](https://dl.acm.org/citation.cfm?doid=363347.363387)提出了解决办法。这篇论文阐述了一种将正则表达式转换为非确定性有穷自动机（NFA）的方法，然后根据在 NFA 中状态的转换，实现正则表达式的匹配。即使是带有回溯的正则表达式的匹配过程，这种算法的时间复杂度与字符串长度也仅呈线性关系。

下面仍以.*.*=.*匹配x=x为例。

先在每一个符号之间添加一个状态，匹配的过程就是状态之间的转移，画出 NFA 如下：
![NFA01](https://raw.githubusercontent.com/reticentfat/wangyuanfrank.com/master/static/images/NFA01.png)

状态 0 为初始状态，状态 4 为结束状态（接受状态）。若状态 m 和 n 之间存在没有符号的有向线段，则称它们之间存在 ε 转换，表示状态 m 无需任何输入即可转移到状态 n。那么对于正则表达式.*.*=.*的 NFA，一开始所处的状态可以是 0, 1 和 2：

![NFA02](https://raw.githubusercontent.com/reticentfat/wangyuanfrank.com/master/static/images/NFA02.png)

现在考虑输入的第一个字符x，读取x后，状态 0 能转移到状态 1，状态 1 能转移到状态 2，由于状态 2 到 3 需要一个=，无法完成转移，那么，转移后的状态就是 1 和 2。又因为状态 1 和 2 到状态 0 都存在 ε 转换，因此，读取了字符x后的状态仍然是 0, 1 和 2：

![NFA02](https://raw.githubusercontent.com/reticentfat/wangyuanfrank.com/master/static/images/NFA02.png)

现在考虑接下来的等号=，同理，状态 0 和 1 分别转移到状态 1 和 2，而状态 2 读取了=可以转移到状态 3，又由于状态 1, 2 到状态 0、状态 3 到状态 4 都有 ε 转换，因此，读取了字符=后的状态是 0, 1, 2, 3 和 4：

![NFA03](https://raw.githubusercontent.com/reticentfat/wangyuanfrank.com/master/static/images/NFA03.png)

末位的字符x的读取过程不再赘述，当所有字符读取完毕，可达的状态有 0, 1, 2, 3 和 4，其中包含了接受状态 4，即正则表达式能够成功匹配该字符串。

利用 NFA 来匹配x=x，过程中没有回溯，每个字符仅被处理了一遍，因此匹配所耗费的时间是与字符数量线性相关的。将正则表达式引擎改为此方法来匹配字符串，就可以避免因回溯带来的大量资源消耗，不重蹈 Cloudflare 的覆辙。

