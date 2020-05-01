---
layout: page
title: Statistics
permalink: /statistics/
---

<table style="width: 100%">
  <tr>
    <th>Post</th>
    <th>Date</th>
    <th>Hits</th>
  </tr>
  {% for post in site.posts %}
    <tr>
      <td>{{ post.title }} </td>
      <td>{{ post.date | date: '%B %Y' }} </td>
      <td><img src="{{ site.counterUrl }}/nocount/tag.svg?url={{ site.url }}{{ post.url }}" alt="Hits"/></td>
    </tr>
  {% endfor %}
</table>
