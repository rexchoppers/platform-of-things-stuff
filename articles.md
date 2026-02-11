---
layout: default
title: Articles
nav_order: 2
---

# Articles

Browse all articles below.

## Projects
{% for post in site.posts %}{% if post.category == "Projects" %}
- [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endif %}{% endfor %}

## Code/Tech
{% for post in site.posts %}{% if post.category == "Code/Tech" %}
- [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endif %}{% endfor %}

## Tutorials/Reviews
{% for post in site.posts %}{% if post.category == "Tutorials/Reviews" %}
- [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endif %}{% endfor %}

## Personal
{% for post in site.posts %}{% if post.category == "Personal" %}
- [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endif %}{% endfor %}
