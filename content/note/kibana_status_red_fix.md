---
title: "kibana_status_red_fix"
author: "wangyuan"
date: '2019-06-07'
slug: technique
categories: ["technique"]
tags: ["技术笔记", "elk", "kibana"]
---

安装完elk发现报错![elasticsearch plugin is red](https://elasticsearch.cn/uploads/questions/20190402/fac402aa92982082e157d8850724ffbe.png)查看top命令发现es服务没有启动，于是执行

```
sudo chkconfig --add elasticsearch
```

再重启

```
sudo -i service elasticsearch start
```

发现正常