---
layout: page
title: Statistics
permalink: /statistics
---

<table class="minimalistBlack">
  <thead>
  <tr>
    <th>Post</th>
    <th>Date</th>
    <th>Hits (From April 14, 2020)</th>
  </tr>
  </thead>
  <tbody>
  {% for post in site.posts %}
    <tr>
      <td>{{ post.title }} </td>
      <td>{{ post.date | date: '%B %Y' }} </td>
      <td><img src="{{ site.statisticsUrl }}/nocount/tag.svg?url={{ site.url }}{{ post.url }}" alt="Hits"/></td>
    </tr>
  {% endfor %}
  </tbody>
</table>