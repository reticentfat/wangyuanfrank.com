---
title: "Cloudflare_7月2日_宕机_思考"
author: "wangyuan"
date: '2019-08-05'
slug: Cloudflare
categories: ["技术"]
tags:  ["Cloudflare", "宕机", "思考"]
---
上个月Cloudflare发生了一次宕机，原因后来官方给出了个[解释](https://new.blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/#appendix-about-regular-expression-backtracking)与[中文解释](https://blog.cloudflare.com/zh/details-of-the-cloudflare-outage-on-july-2-2019-zh) 。这件事与失事一样，并不是单一问题导致的，而是一连串原因导致保险措施失效。
作为中断核心诱因的正则表达式是 
`(?:(?:\"|'|\]|\}|\\|\d|(?:nan|infinity|true|false|null|undefined|symbol|math)|\`|\-|\+)+[)]*;?((?:\s|-|~|!|{}|\|\||\+)*.*(?:.*=.*)))`
