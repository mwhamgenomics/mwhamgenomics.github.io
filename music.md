---
title: Music
url: /music
---

<h2>Music <span class="badge">{{ site.categories['music'] | length }}</span></h2>

<ul>
  {% for post in site.categories['music'] %}
    <li>
      <span>{{ post.metadata.human_readable_date }}</span>
      <a href="{{ post.url }}">{{ post.metadata.title }}</a>
    </li>
  {% endfor %}
</ul>
