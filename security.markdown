---
layout: page
title: Security
description: all posts about Security stuffs, ofter in Azure
permalink: /security/
---

![Security](/assets/Security.jpg)
{% assign posts = site.posts | where_exp: "item", "item.categories contains 'Security'" %}
{% for post in posts %}
  [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date_to_string }}
{% endfor %}