---
layout: page
title: The Mistery in Gecko
---
{% include JB/setup %}

I just put my observation from Gecko source code and try helping people (including myself) to get a clear view on the huge code base.

## Recent Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

