---
layout: page
title: Terraform
description: Posts about Terraform
permalink: /terraform/
---
![terraform](/assets/terraform.png)
{% assign posts = site.posts | where_exp: "item", "item.categories contains 'aks'" %}
{% for post in posts %}
  [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date_to_string }}
{% endfor %}