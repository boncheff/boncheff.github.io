---
layout: default
title: boncheff
---

<h1 class="writing-heading">Writing</h1>

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) <span style="opacity:0.6">— {{ post.date | date: "%d %b %Y" }}</span>
{% endfor %}
