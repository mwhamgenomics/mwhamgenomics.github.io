---
title: Archive
---

<h2>All posts</h2>

<ul class="link-list">
{% for post in site.posts %}
    <li>
        <span class="post-meta">{{ post.metadata.human_readable_date }} -
        <a href="/{{ post.metadata.category }}">{{ post.metadata.category }}</a></span> -
        <a class="post-link" href="{{ post.url }}">{{ post.metadata.title }}</a>
    </li>
{% endfor %}
</ul>
