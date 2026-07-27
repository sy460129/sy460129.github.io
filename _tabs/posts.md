---
layout: page
title: Posts
icon: fas fa-stream
order: 2
permalink: /posts/
---

<div id="post-list" class="flex-grow-1 px-xl-1">
{% for post in site.posts %}
  <article class="card profile-box mb-4">
    <div class="card-body">
      <h1 class="card-title my-2">
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h1>
      <div class="post-meta text-muted small my-2">
        <span>
          <i class="far fa-calendar fa-fw me-1"></i>
          {{ post.date | date: "%Y-%m-%d" }}
        </span>
        {% if post.categories.size > 0 %}
          <span class="ms-3">
            <i class="far fa-folder fa-fw me-1"></i>
            {{ post.categories | join: ', ' }}
          </span>
        {% endif %}
      </div>
      <div class="card-text content mt-3">
        <p>{{ post.excerpt | strip_html | truncate: 150 }}</p>
      </div>
    </div>
  </article>
{% endfor %}
</div>
