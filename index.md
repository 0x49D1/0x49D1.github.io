---
layout: default
title: "Home"
---

# Latest Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <small>{{ post.date | date: "%B %-d, %Y" }}</small>
      <a href="{{ post.url }}">
      <p>{{ post.excerpt }}</p>
      </a>
    </li>
  {% endfor %}
</ul>
