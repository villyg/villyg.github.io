---
layout: page
title: Golf
# permalink: /projects/gunvault/index
---
<div id="tags-list">
  {% assign posts = site.posts | where_exp: "item", "item.tags contains 'golf'" %}
    <ul class="post-list post-list-narrow">
    {% for post in posts limit:5 %}
        <li>
          {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
          <b>
            <a href="{{ post.url | relative_url }}">
              {{ post.title | escape }}
            </a>
          </b> - <i>{{ post.date | date: date_format }}</i>
        </li>   
    {% endfor %}
  </ul>
</div>

