---
# You don't need to edit this file, it's empty on purpose.
# Edit theme's home layout instead if you wanna make some changes
# See: https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: default
---
{% for post in site.posts %}
  <div>
    <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
    <p><strong>{{ post.date | date: "%B %e, %Y" }}</strong> in {{ post.category }}</p>
    {{ post.excerpt }}
  </div>
{% endfor %}	
