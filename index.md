---
layout: page
title: Unconventional Development
tagline: Designing from Experience
---

<div class="posts">
    {% for post in site.posts %}
        <article class="post">
            <header>
                <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
                <span>{{ post.date | date_to_string }}</span>
            </header>
            <div class="excerpt">
                {{ post.content | truncatewords: 200 }} ...
            </div>
        </article>
    {% endfor %}
</div>