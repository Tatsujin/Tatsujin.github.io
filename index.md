---
layout: page
title: Welcome To a world called DevOps! *whipcrack*
---
{% include JB/setup %}

## Run, Coward!

Welp, welcome to my shitty DevOps blog.  The focus here will be on my journey
from being a shitty hosting company SysAdmin to a marginally less shitty DevOps
type person. I hope that my crapposting on stuff like DevOps theory, IaaS,
Server config deployment and management, tools, etc help you in your journey as
well to a semi-competent computerman.

If you're wondering how I setup this blog, read up on GitHub Pages and Jekyll
Bootstrap.

[GitHub Pages](https://pages.github.com/)

[Jekyll Bootstrap](http://jekyllbootstrap.com)

## Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
