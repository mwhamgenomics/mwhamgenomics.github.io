---
title: Programming
url: /programming
---

<h2>Programming <span class="badge">{{ site.categories['programming'] | length }}</span></h2>

<ul>
  {% for post in site.categories['programming'] %}
    <li>
      <span>{{ post.metadata.human_readable_date }}</span>
      <a href="{{ post.url }}">{{ post.metadata.title }}</a>
    </li>
  {% endfor %}
</ul>

