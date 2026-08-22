---
layout: default
title: About
permalink: /about/
---

I am a software engineer by trade, currently working on the windows kernel-mode driver at AMD. 

Check out what I am currently exploring [here](/notes) and stuff I've made [here](/projects).

<div class="quest-log">
{% for entry in site.data.timeline %}
  <div class="quest-entry{% if entry.status == 'current' %} current{% endif %}">
    <div class="quest-meta">
      <span class="quest-year">{{ entry.year }}</span>
    </div>
    <div class="quest-org">{{ entry.org }}</div>
    {% if entry.role and entry.role != "" %}<div class="quest-role">{{ entry.role }}</div>{% endif %}
    <ul class="quest-desc">
      {% for line in entry.description %}<li>{{ line }}</li>
      {% endfor %}
    </ul>
  </div>
{% endfor %}
</div>