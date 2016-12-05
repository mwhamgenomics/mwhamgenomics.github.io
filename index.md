---
layout: default
---

Hello and welcome to {{ site.title }}, a combined technical blog on programming, bioinformatics and music.

In my job as a bioinformatician and software developer, I often encounter periods of head-scratching, trying to solve a problem or get something to work. I usually then find a solution and move on, but then might come across the same problem again months later. Hopefully this website will be useful to me and to others encountering the same problems.

Since I work for a genomics institute, most of the programming I do is to solve problems that are biological at heart. Bioinformatics and computational biology is constantly evolving, always faced with some degree of uncertainty, and often involves highly complex processing of huge quantities of input data. Documented here are some of my experiences in bioinformatics, tools that I've used, and problems that I've encountered and attempted to solve.

Finally, in deference to my main spare-time interest, there are also some posts relating to music - harmonic theory, chord extensions, and similar.

Hope you find this site useful.

M.

### About me

- I am a biologist by qualification, programmer by occupation and musician by night. You can find me on GitHub [here](https://github.com/mwhamgenomics).
- This website was built using [Jekyll](http://jekyllrb.com), styled with [Bootstrap](http://getbootstrap.com) and hosted on [GitHub Pages](https://pages.github.com).
- Jekyll is a Ruby app for generating static websites from html and [markdown](http://daringfireball.net/projects/markdown/)/[kramdown](http://kramdown.gettalong.org).
- The inline vector icons found here are from {% include icon-fontawesome.html href="http://fontawesome.io" icon="fa-flag" msg="Font Awesome" %}

### Posts

Have a look around categories and tags, or see a list of all posts here.

<ul class="post-list">
{% for post in site.posts %}
  <li>
    <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}  -
        {% include comma_sep_links.html links=post.categories type='categories' %}
    </span>
    <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>
