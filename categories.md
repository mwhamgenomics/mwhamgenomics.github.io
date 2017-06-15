---
layout: page
title: Categories
permalink: /categories
---
Here you can browse posts in the categories of bioinformatics, programming or music. Each post belongs to precisely one category.

{% for mapping in site.categories %}
<h3 id="{{ mapping[0] }}">{{ mapping[0] }} <span class="badge">{{ mapping[1].size }}</span></h3>
<ul class="link-list">
  {% for post in mapping[1] %}
    <li>
      <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
      <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
{% endfor %}
