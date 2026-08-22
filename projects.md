---
layout: default
title: Projects
permalink: /projects/
---

<ul class="post-list">
{% for item in site.projects reversed %}
  <li>
    <span class="date">{{ item.date | date: "%Y-%m-%d" }}</span>
    <a href="{{ item.url | relative_url }}">{{ item.title }}</a>
  </li>
{% endfor %}
</ul>
