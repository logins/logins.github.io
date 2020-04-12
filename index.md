---
layout: default
---
# Blog posts
{% for post in site.posts %}

<div class="linkblocktext">
<!-- 
div tag will exclude markdown inside this section.
Be sure to put a new line between any html and any markdown!
-->
  <a href="{{ post.url | prepend: site.baseurl }}">
    <h2>{{ post.title }}</h2>
    {% if post.subtitle %}
      <blockquote>     
      <p>{{ post.subtitle }}</p>
      </blockquote>
    {% endif %}
  </a>
</div>

_{{ post.date | date: "%B %Y"}} - {{ post.categories[0] }}_

* * *

{% endfor %}
