---
layout: default       # or "home", depending on your theme
title: "Concurrency Chronicles"
---

# Concurrency Chronicles

Welcome! Below you'll find my latest blog posts on concurrency, parallelism, and more.

{% comment %}
  If your chosen theme doesn't automatically provide styling for the post list,
  you can wrap them in your own HTML. For example:
{% endcomment %}

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">
        {{ post.title }}
      </a>
      <span> - {{ post.date | date_to_string }}</span>
    </li>
  {% endfor %}
</ul>

*Check back often for new posts!*
