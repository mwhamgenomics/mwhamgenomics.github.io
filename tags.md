---
layout: page
title: Tags
permalink: /tags
---
Here you can browse posts by tags. Each tag may have many posts, and each post may have many tags.

{% for mapping in site.tags %}
<h3 id="#{{ x[0] }}">{{ mapping[0] }} <span class="badge">{{ mapping[1].size }}</span></h3>
<ul class="link-list">
  {% for post in mapping[1] %}
    <li>
      <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
      <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
{% endfor %}
