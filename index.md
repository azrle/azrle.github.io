---
layout: page
---
{% include JB/setup %}


{% for post in site.posts %}
  <div class="page-header">
      <div style="float: right; clear: both;">{{ post.date | date_to_long_string }}</div>
      <h3><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></h3>
  </div>
{% endfor %}
