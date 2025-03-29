---
layout: default
title: Home
---

<div class="collections-grid">
  {% for collection in site.collections %}
    {% unless collection.label == "posts" %}
    <section class="collection-section">
      <h2>{{ collection.label | capitalize }}</h2>
      {% assign items = site[collection.label] | sort: 'date' | reverse %}
      {% for post in items limit:3 %}
        <article class="post-preview">
          <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
          <time>{{ post.date | date: "%B %d, %Y" }}</time>
        </article>
      {% endfor %}
      <a href="/collections/{{ collection.label }}" class="view-all">View all {{ collection.label }} posts →</a>
    </section>
    {% endunless %}
  {% endfor %}
</div>
