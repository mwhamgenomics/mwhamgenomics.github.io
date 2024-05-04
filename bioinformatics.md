---
title: Bioinformatics
url: /bioinformatics
---

<h2>Bioinformatics <span class="badge">{{ site.categories['bioinformatics'] | length }}</span></h2>

<ul>
  {% for post in site.categories['bioinformatics'] | reverse %}
    <li>
      <span>{{ post.metadata.human_readable_date }}</span>
      <a href="{{ post.url }}">{{ post.metadata.title }}</a>
    </li>
  {% endfor %}
</ul>
