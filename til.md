---
layout: page
title: TIL
permalink: /til/
---

Daily learning notes (category: til).

{% assign til_posts = site.posts | where_exp: "post", "post.categories contains 'til'" %}
{% for post in til_posts %}
- {{ post.date | date: "%Y-%m-%d" }} — [{{ post.title }}]({{ post.url }})
{% endfor %}
