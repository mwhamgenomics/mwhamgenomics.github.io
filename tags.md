---
layout: page
title: Tags
permalink: /tags
---

{% for t in site.tags %}
### {{ t[0] }} {#{{ t[0] }}}
  {% for p in t[1] %}
  - [{{ p.title }}]({{ p.url }}) ({{ p.date | date: '%D' }})
  {% endfor %}
{% endfor %}
