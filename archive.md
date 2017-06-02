---
layout: page
title: Archive
permalink: /archive
---

<ul class="link-list">
{% for post in site.posts %}
    <li>
        <span class="post-meta">{{ post.date | date: "%-d %b %Y" }} -
        <a href="/categories/{{ post.category }}">{{ post.category }}</a></span> -
        <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
    </li>
{% endfor %}
</ul>
