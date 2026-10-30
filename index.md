---
layout: default
title: 首页
---

# jimwitty

技术学习笔记。

## WRK 源码学习系列

按 Windows Research Kernel（WS03 SP1 代码线）源码的启动顺序，逐批拆解 NT 内核的点火全程，共 9 篇（第 0–8 批）。

<ul>
{% assign wrk_posts = site.categories.WRK | sort: "title" %}
{% for post in wrk_posts %}
<li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>
