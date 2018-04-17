---
layout: page
title: About
description: 美若黎明
keywords: Jayant Xie, 谢正尧
comments: true
menu: 关于
permalink: /about/
---

Jayant Xie，浙大大四电仪专业，毕业狗一只，近期忙毕设。

爱技术，爱摄影，爱阅读，爱健身......

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
