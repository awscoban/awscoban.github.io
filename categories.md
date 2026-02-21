---
layout: post
title: Chủ đề
---

<div class="tags-expo">
  <!-- <div class="tags-expo-list">
    {% for tag in site.categories %}
    <a href="#{{ tag[0] | slugify }}" class="post-tag">{{ tag[0] }}</a>
    <br>
    {% endfor %}
  </div>
  <hr/> -->
  <div class="tags-expo-section">
    {% for group in site.data.category_groups %}
    <h2 id="{{ group.name | slugify }}">{{ group.name }}</h2>
    
    <ul class="tags-expo-posts" style="margin-left: 20px;">
      {% assign parent_posts = site.categories[group.name] %}
      {% for post in parent_posts reversed %}
        {% assign in_sub = false %}
        {% for sub in group.subcategories %}
          {% if post.categories contains sub %}
            {% assign in_sub = true %}
            {% break %}
          {% endif %}
        {% endfor %}
        {% if in_sub == false %}
        <div>
          <span style="float: left;">
            <a href="{{ post.url }}">{{ post.title }}</a>
          </span>
          <span style="float: right;">
            {{ post.date | date_to_string }}
          </span>
        </div>
        <br>
        {% endif %}
      {% endfor %}
    </ul>

    {% for sub in group.subcategories %}
    <h4 style="margin-left: 20px;" id="{{ sub | slugify }}">{{ sub }}</h4>
    <ul class="tags-expo-posts" style="margin-left: 40px;">
      {% assign posts = site.categories[sub] %}
      {% for post in posts reversed %}
      <div>
        <span style="float: left;">
          <a href="{{ post.url }}">{{ post.title }}</a>
        </span>
       <span style="float: right;">
          {{ post.date | date_to_string }}
        </span>
      </div>
      <br>
      {% endfor %}
    </ul>
    {% endfor %}
    <hr>
    {% endfor %}
  </div>
</div>

