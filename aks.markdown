---
layout: page
title: AKS
description: Posts about AKS
permalink: /aks/
---
![AKS](/assets/AKS.png)
{% assign posts = site.posts | where_exp: "item", "item.categories contains 'AKS'" %}
{% for post in posts %}
  [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date_to_string }}
{% endfor %}