---
layout: page
title: Azure Arc
description: Posts about Azure Arc
permalink: /azurearc/
---
![AKS](/assets/AKS.png)
{% assign posts = site.posts | where_exp: "item", "item.categories contains 'Arc'" %}
{% for post in posts %}
  [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date_to_string }}
{% endfor %}