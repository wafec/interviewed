---
layout: default
title: SQL
---

# SQL

{% assign items = site.pages | where_exp: "p", "p.path contains 'src/sql/'" | where_exp: "p", "p.name != 'index.md'" | sort: "path" %}
{% if items.size == 0 %}
_No question sets yet — generate one with `/new-question sql`._
{% else %}
<ul>
{% for p in items %}
  <li><a href="{{ p.url | relative_url }}">{{ p.title | default: p.name }}</a></li>
{% endfor %}
</ul>
{% endif %}

[← Back to home]({{ "/" | relative_url }})
