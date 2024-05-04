---
title: Archive
---

<h2>All posts <span class="badge">{{ site.posts | length }}</span></h2>

<ul>
{% for post in site.posts | reverse %}
    <li>
        <span>{{ post.metadata.human_readable_date }} -
        <a href="/{{ post.metadata.category }}">{{ post.metadata.category }}</a></span> -
        <a class="post-link" href="{{ post.url }}">{{ post.metadata.title }}</a>
    </li>
{% endfor %}
</ul>
