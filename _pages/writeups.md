---
title: "Write-ups"
permalink: /writeups/
layout: archive
author_profile: true
---

In-depth technical write-ups, vulnerability research, and advanced analysis. Long-form content for those who want to go deep.

{% assign category_posts = site.posts | where_exp: "post", "post.categories contains 'writeups'" %}
{% if category_posts.size > 0 %}
  {% for post in category_posts %}
    {% include archive-single.html %}
  {% endfor %}
{% else %}
*No write-ups published yet — check back soon.*
{% endif %}
