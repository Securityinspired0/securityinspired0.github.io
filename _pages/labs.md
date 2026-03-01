---
title: "Labs"
permalink: /labs/
layout: archive
author_profile: true
---

Hands-on security labs and step-by-step technical walkthroughs. Each post documents a real exercise — tools used, methodology, findings, and lessons learned.

{% assign category_posts = site.posts | where_exp: "post", "post.categories contains 'labs'" %}
{% if category_posts.size > 0 %}
  {% for post in category_posts %}
    {% include archive-single.html %}
  {% endfor %}
{% else %}
*No labs published yet — check back soon.*
{% endif %}
