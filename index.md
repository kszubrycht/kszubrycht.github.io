---
title: /
layout: home
permalink: /
---

# Welcome

My name is Kamil Szubrycht. I'm software developer coding mainly in Ruby and JavaScript.

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
