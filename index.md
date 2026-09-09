---
layout: default
title: boncheff
---

# Writing

Squeezing tokens out of rented GPUs. Notes from the experiments.

{% assign last_year = "" %}
{% for post in site.posts %}
  {% assign year = post.date | date: "%Y" %}
  {% if year != last_year %}
{% if last_year != "" %}{% endif %}
## {{ year }}
{% assign last_year = year %}
  {% endif %}
- [{{ post.title }}]({{ post.url | relative_url }}) <span style="opacity:0.6">— {{ post.date | date: "%d %b" }}</span>
{% endfor %}
