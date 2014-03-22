---
layout: page
title: My future ex-web site!
---
{% include JB/setup %}

You've stumbled upon yet another attempt at a personal collection of facts and
rants and the like, with as much chance of surviving as previous versions,
except perhaps that this one is being versioned in a trusted server, so who
knows if it lasts a few years.

*I'm always looking for a future ex-web site.*

![Ian Malcolm, from Jurassic Park]({{ site.url }}/assets/malcolm.png)

## Blog Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

<!---
vim: syntax=markdown:
-->
