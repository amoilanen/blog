---
layout: default
---

<p class="intro">A blog about software development, programming languages, and computer science. Exploring ideas from functional programming to language design, from algorithms to software architecture.</p>

{% assign posts_by_year = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}
{% for year in posts_by_year %}
<div class="post-list-year">{{ year.name }}</div>
<div class="post-list">
  {% for post in year.items %}
  <article>
    <h2><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></h2>
    <div class="meta">{{ post.date | date: "%B %-d, %Y" }}</div>
    <div class="excerpt">{{ post.content | strip_html | truncatewords: 40 }}</div>
  </article>
  {% endfor %}
</div>
{% endfor %}
