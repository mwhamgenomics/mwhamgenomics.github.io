---
extends: post.html
title: Website updates 2020
category: programming
tags: ['web development']
date: 2020-04-27
external_links:
    - title: Jekyll official site
      link: https://jekyllrb.com
      description: Jekyll, the officially supported static site builder for GitHub Pages
    - title: Picnic CSS
      link: https://picnicss.com
      description: Picnic CSS, the styling framework I switched this site to
---

Current events have given me a lot of time in the evenings to catch up on side projects, one of which was an update of
this website. I'd been wanting to do this for a while, but pretty much since mid-2019 had been too busy. For the last
month or so I have been overhauling it behind the scenes and the end result is now live.


## Static site builders

Initially I built this site with [Jekyll](https://jekyllrb.com), but eventually grew tired of some of the boundaries I
had to work within:

- I wanted slicker syntax highlighting of code blocks. I didn't like the way Jekyll made me call it with HTML template
  syntax like `{% raw %}{% highlight python %}{% endraw %}`, rather than Markdown backquote syntax.
- I had to name the files for new posts with a date-stamp in front. This makes it easy to view them by date, but most of
  the time I want to view them by subject. If you write a post about, say, RVM, and want to find it later, you'd want to
  look for 'rvm' in the file name, not a datestamp for whatever date you wrote it on.
- Rather than a clean slate, there was a lot of pre-existing CSS and Sass code when I first got started with the Jekyll
  placeholders, and I didn't really know what I should and shouldn't edit.
- Jekyll is designed to do a lot for you, but on the flip side that means when there's a problem, it's not so easy to
  see whether the cause is a problem with the site source or some Jekyll behind-the-scenes trickery.
- I wanted something transparent that could build a tree of source files to be translated to an equivalent site map,
  but would also be easy for me to extend - I'm not enough of a Ruby programmer to start writing Jekyll plugins.

GitHub Pages is able to host any static files in the repo's master branch, so I could just add HTML files built by hand
or any static site builder. After a look at Jekyll alternatives, I asked myself the same question that any sane,
well-adjusted programmer does: how hard can it be to just roll my own? I figured that if I could build a combination of
Markdown processing, HTML templating and post metadata handling, not only would I be able to pare it down to just the
bits I needed, it would also be an interesting learning experience. The result is about 450 lines of site building code
using [Python-Markdown](https://python-markdown.github.io), [Jinja2](https://jinja.palletsprojects.com) and
[Pygments](https://pygments.org).


## Simpler CSS

I was also initially doing styling with Bootstrap, which is used for many large web projects but I've found it to be
overkill for small static sites. The source code of Bootstrap's CSS base runs to over 10,000 lines even before adding
Javascript interactivity. The heavy use of CSS classes and the verbose and repetitive language necessary for defining
even something like a small navbar was starting to get to me:

```html
<nav class="navbar navbar-expand-lg navbar-light bg-light">
  <a class="navbar-brand" href="#">A navbar</a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbar_content" aria-controls="navbar_content" aria-expanded="false" aria-label="Toggle nav">
    <span class="navbar-toggler-icon"></span>
  </button>

  <div class="collapse navbar-collapse" id="navbar_content">
    <ul class="navbar-nav mr-auto">
      <li class="nav-item active">
        <a class="nav-link" href="#">Home</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">This</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">That</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">Other</a>
      </li>
    </ul>
  </div>
</nav>
```

So at the same time as building my own static site generator, I was looking at
[alternative CSS frameworks](https://github.com/troxler/awesome-css-frameworks). While writing the site builder
happened fairly naturally, I spent a lot of time puzzling over how I wanted to style it. I wanted something relatively
lightweight but still responsive and mobile-friendly. I'm also not hardcore enough to build a website style from
scratch, so I was fine with the framework being 'opinionated' and guiding you towards doing things a certain way. After
trying a few frameworks and CSS resets, I decided on [Picnic](https://picnicss.com), which has a responsive
grid system but is otherwise relatively classless. This means that the HTML for defining a navbar is a bit nicer:

```html
<nav>
  <a href="#" class="brand">A Navbar</a>
  <input id="minimenu" type="checkbox" class="show">
  <label for="minimenu" class="burger pseudo button"><img class="icon_large" src="/img/menu.svg" /></label>

  <div class="menu">
    <a href="#">Home</a>
    <span class="divider"></span>
    <a href="#">This</a>
    <a href="#">That</a>
    <a href="#">Other</a>
  </div>

</nav>
```

This then is a two-part exercise in minimalism - my own small rebellion against the endless DOM tree nesting,
Javascript dependencies, myriad UI frameworks and other excesses of modern web development.

While I'm happy with the result, there are still some improvements I can make. The new site's workflow is definitely not
as polished as Jekyll's at the moment. The built files have to sit in master, so the source files have to be somewhere
else in a separate branch or repo. This means that whenever I want to update the site, I have to push changes twice.
This may or may not be fixable - on the face of it, GitHub Pages seems to be fairly tightly integrated with Jekyll and
there's no documentation on how to use anything other than Jekyll, but it may be possible to
[streamline the process with TravisCI](https://docs.travis-ci.com/user/deployment/pages). I'll see if I can get that to
work later.
