---
layout: default       # or "home", depending on your theme
title: "Concurrency Chronicles"
---

# Concurrency Chronicles

Welcome! Below you'll find my latest blog posts on concurrency, parallelism, and more.

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
