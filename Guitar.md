---
layout: page
title: Guitar
permalink: /Guitar/
---
Guitar video performances and transcriptions. [Lilypond](http://lilypond.org/) was used for the transcriptions. You can find lilypond source files [here](https://github.com/soundg33k/guitar-transcriptions).
<ul>
  {% assign posts = site.posts | where_exp:"post","post.categories contains 'guitar'"  %}
  {% for post in posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
