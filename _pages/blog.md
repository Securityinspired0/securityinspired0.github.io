---
title: "Blog"
permalink: /blog/
layout: archive
author_profile: true
---

Thoughts, opinions, and commentary on security topics — lessons from the field, industry observations, and things worth talking about.

{% assign category_posts = site.posts | where_exp: "post", "post.categories contains 'blog'" %}
{% if category_posts.size > 0 %}
  {% for post in category_posts %}
    {% include archive-single.html %}
  {% endfor %}
{% else %}
*No posts published yet — check back soon.*
{% endif %}
