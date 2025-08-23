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

{% assign categories = site.categories %}

{% for category in categories %}
  {% assign category_name = category[0] %}
  <h3>
  {% elsif category_name == "system-design" %}
    System Design
  {% elsif category_name == "datastore" %}
    Datastore
  {% elsif category_name == "java" %}
    Java
  {% if category_name == "infra" %}
    Infrastructure & Platform Engineering
  {% elsif category_name == "monitoring" %}
    Monitoring & Observability
  {% elsif category_name == "protocols" %}
    Web & Network Protocols
  {% elsif category_name == "journey" %}
    Personal Journey
  {% else %}
    {{ category_name }}
  {% endif %}
  </h3>

  <ul>
    {% for post in category[1] %}
      <li><a href="/post{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}