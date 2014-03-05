---
layout: page
title: Welcome!
---
{% include JB/setup %}

You've stumbled upon yet another attempt at a web site.

*I'm always looking for an ex-web site.*

![Ian Malcolm, from Jurassic Park]({{ site.url }}/assets/malcolm.png)

## Sample Posts

Here's an empty "posts list" (even though there are no posts yet). Clever, uh?

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

<!---
vim: syntax=markdown:
-->
