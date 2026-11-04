---
layout: default
title: 首页
---

# jimwitty

技术学习笔记。

## WRK 源码学习系列

按 Windows Research Kernel（WS03 SP1 代码线）源码的启动顺序，逐批拆解 NT 内核的点火全程，共 9 篇（第 0–8 批）。

<ul>
{% assign wrk_posts = site.categories.WRK | sort: "series_order" %}
{% for post in wrk_posts %}
<li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

## Win 内核驱动学习系列

按微软官方《内核模式驱动程序体系结构设计指南》逐批研读 Windows 内核驱动的体系结构，共 12 篇（第 1–12 批）。

<ul>
{% assign wk_posts = site.categories["windows-kernel"] | sort: "series_order" %}
{% for post in wk_posts %}
<li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

## Uninformed 全译系列

Uninformed 期刊（uninformed.org）经典论文的中文全译：PatchGuard 三部曲（第 3 / 6 / 8 卷）与 FUTo（第 3 卷），共 4 篇，按系列顺序排列。

<ul>
{% assign uninf_posts = site.posts | where: "series", "uninformed" | sort: "series_order" %}
{% for post in uninf_posts %}
<li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

## 《新机器的灵魂》精读系列

逐章精读 Tracy Kidder 的《The Soul of a New Machine》（1981）：Data General 一支年轻团队从零造出 Eclipse MV/8000 的全过程，普利策奖与美国国家图书奖双奖之作。严格按原书章节推进（序章 + 16 章 + 尾声），全 18 篇已完结。

<ul>
{% assign soul_posts = site.categories["新机器的灵魂"] | sort: "series_order" %}
{% for post in soul_posts %}
<li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

## 《帝国的叛逆者》精读系列

逐章精读 Michael Drummond 的《Renegades of the Empire》（1999）：DirectX 三人组如何在没有许可证的情况下造出 DirectX、又如何带着 Project Chrome 一头撞向帝国的围墙。严格按原书章节推进（前言 + 19 章及尾声），全 20 篇已完结。

<ul>
{% assign renegades_posts = site.categories["帝国的叛逆者"] | sort: "series_order" %}
{% for post in renegades_posts %}
<li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

## 独立文章

<ul>
{% assign essay_posts = site.categories["essay"] | where_exp: "post", "post.series != 'uninformed'" | sort: "title" %}
{% for post in essay_posts %}
<li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>
