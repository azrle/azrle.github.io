---
layout: default
---
{% for post in site.posts %}
<div class="post-item no-decoration">
  <h3>
    <span class="date">{{ post.date | date: "%Y-%m-%d" }}</span>
    <br>
    <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
  </h3>
  <p class="preview">
    {{ post.excerpt }}
  </p> <a class="color-info" href="{{ BASE_PATH }}{{ post.url }}">...more</a>
</div>
{% endfor %}
