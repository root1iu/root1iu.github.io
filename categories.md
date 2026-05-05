---
layout: page
title: Categories
---

{% assign sorted_categories = site.categories | sort %}

{% if sorted_categories.size > 0 %}
  {% for category in sorted_categories %}
    {% assign category_name = category[0] %}
    {% assign category_posts = category[1] %}

## {{ category_name }} ({{ category_posts | size }})

<ul>
  {% for post in category_posts %}
    <li>
      <span>{{ post.date | date: site.theme_config.date_format }}</span>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
  {% endfor %}
{% else %}
No categories yet.
{% endif %}
