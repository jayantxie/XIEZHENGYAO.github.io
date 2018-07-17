---
layout: page
title: About
description: 美若黎明
keywords: Jayant Xie, 谢正尧
comments: true
menu: 关于
permalink: /about/
---

Jayant，从事Java后端与中间件开发方向，关注区块链技术。

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
