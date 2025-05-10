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
  
  {% if category_name == "infra" %}
    <h3>Infrastructure & Platform Engineering</h3>
  {% elsif category_name == "monitoring" %}
    <h3>Monitoring & Observability</h3>
  {% elsif category_name == "best-practices" %}
    <h3>Software Engineering Best Practices</h3>
  {% elsif category_name == "system-design" %}
    <h3>System Design & Distributed Systems</h3>
  {% elsif category_name == "protocols" %}
    <h3>Web & Network Protocols</h3>
  {% elsif category_name == "journey" %}
    <h3>Personal Journey</h3>
  {% else %}
    <h3>{{ category_name }}</h3>
  {% endif %}

  <ul>
    {% for post in category[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}