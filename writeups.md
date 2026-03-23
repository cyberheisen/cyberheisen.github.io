---
layout: default
title: Writeups
permalink: /writeups/
---

<div class="category-header">
  <h1>writeups</h1>
  <p>CTF walkthroughs and machine write-ups</p>
</div>

{% assign ctf_posts = site.posts | where_exp: "p", "p.categories contains 'CTF'" %}
{% if ctf_posts.size > 0 %}
  <div class="post-list">
    {% for post in ctf_posts %}
      {% include post-card.html post=post %}
    {% endfor %}
  </div>
{% else %}
  <div class="empty-state">no writeups yet</div>
{% endif %}
