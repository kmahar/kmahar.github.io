---
layout: page
title: Archive
permalink: /archive/
---

{%- if site.posts.size > 0 -%}
<ul class="post-list">
  {%- for post in site.posts -%}
  <li>
    {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
    {{ post.date | date: date_format }}
      <a href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
      </a>
  </li>
  {%- endfor -%}
</ul>
{%- endif -%}