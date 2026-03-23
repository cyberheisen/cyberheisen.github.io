---
layout: default
title: Notes
permalink: /notes/
---

<div class="category-header">
  <h1>notes</h1>
  <p>Cheat sheets, tool references, and methodology notes</p>
</div>

{% assign notes_posts = site.posts | where_exp: "p", "p.categories contains 'Notes'" %}
{% if notes_posts.size > 0 %}
  <div class="post-list">
    {% for post in notes_posts %}
      {% include post-card.html post=post %}
    {% endfor %}
  </div>
{% else %}
  <div class="empty-state">no notes yet</div>
{% endif %}
