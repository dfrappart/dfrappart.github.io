---
layout: page
title: Kubernetes
description: Posts about Kubernetes
permalink: /kubernetes/
---
![Kubernetes](/assets/Kubernetes.png)
{% assign posts = site.posts | where_exp: "item", "item.categories contains 'Kubernetes'" %}
{% for post in posts %}
  [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date_to_string }}
{% endfor %}