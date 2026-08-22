---
layout: default
title: Notes
permalink: /notes/
---

<ul class="post-list">
{% for item in site.notes reversed %}
  <li>
    <span class="date">{{ item.date | date: "%Y-%m-%d" }}</span>
    <a href="{{ item.url | relative_url }}">{{ item.title }}</a>
  </li>
{% endfor %}
</ul>
