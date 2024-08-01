---
layout: default
---
  {% for post in site.posts %}
   <h2>Posts</h2>
   <h3> {{ post.title }}</h3>
   <div class="post">
   {{ post.content }}
   </div>
  {% endfor %}