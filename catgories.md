---
layout: page
title: Categories
permalink: /categories
---

{% for c in site.categories %}
### {{ c[0] }} {#{{ c[0] }}}
  {% for p in c[1] %}
  - [{{ p.title }}]({{ p.url }}) ({{ p.date | date: '%D' }})
  {% endfor %}
{% endfor %}
