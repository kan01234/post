---
title: kan01234 - Software Engineer Notes
---

<!-- <ul>
  {% for post in site.posts %}
    <li>
      <a href="/post{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul> -->

{% for category in site.my-categories %}
  <h3>{{ category.title[0] }}</h3>
  <ul>
    {% for post in category.key[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}