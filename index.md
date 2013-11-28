---
layout: index
title: heljoy
---
{% include heljoy/setup %}

{% for post in site.posts limit: 3 %}
  <h2 class="title"><a href="{{ post.url }}">{{ post.title }}</a></h2>
  <div class="post_info">
  <ul class="tag_box inline">
    <li id="post-date"> Posted on {{ post.date | date_to_string }}</li>
    {% unless no_category %}
      {% if post.category %}
        <li> in <a href="{{ BASE_PATH }}{{ site.heljoy.categories_path }}#{{ post.category }}-ref">
    	 {{ post.category }}
	</a>
        </li>
      {% endif %}
    {% endunless %}
  </ul>
  </div>
  <div class="archive">
    {{ post.content | split: '<!-- more -->' | first }}
  </div>
  <hr />
  {% endfor %}
