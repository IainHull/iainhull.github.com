---
layout: page
title: Iain's Blog
tagline: Not another software engineering blog
---
{% include JB/setup %}

I am interested in large distributed systems and the architecture required to support them. [More](about.html)

## Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
