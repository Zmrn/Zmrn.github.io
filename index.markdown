---
layout: default
title: Home
nav_order: 1
---

欢迎来到 Anso 的文档站点。

## 文章列表

<ul>
{% for post in site.posts %}
	<li>
		<a href="{{ post.url }}">{{ post.title }}</a>
		<span style="font-size: 0.9em; color: #666;">{{ post.date | date: "%Y-%m-%d" }}</span>
	</li>
{% endfor %}
</ul>

