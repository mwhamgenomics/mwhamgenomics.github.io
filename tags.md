---
title: Tags
url: /tags
---
Here you can browse posts by tags. Each tag may have many posts, and each post may have many tags.

{% for tagname, posts in site.tags | dictsort %}
<h2 id="{{ tagname }}">{{ tagname }} <span class="badge">{{ posts | length }}</span></h2>
<ul class="link-list">
  {% for post in posts | reverse %}
    <li>
      <span class="post-meta">{{ post.metadata.human_readable_date }}</span>
      <a class="post-link" href="{{ post.url }}">{{ post.metadata.title }}</a>
    </li>
  {% endfor %}
</ul>
{% endfor %}
