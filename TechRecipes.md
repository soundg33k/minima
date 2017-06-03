---
layout: page
title: Tech Recipes
permalink: /Tech Recipes/
---
3 liters of ideas, 1 teaspoon of sudo...
<ul>
  {% assign posts = site.posts | where_exp:"post","post.categories contains 'tech-recipe'"  %}
  {% for post in posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
