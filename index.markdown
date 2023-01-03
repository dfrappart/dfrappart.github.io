---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---




## What's this site ?
Welcome on teknews blog 2.0 ^^
  
Old articles from the WordPress era are available as archive in pdf in the [Archive](https://blog.teknews.cloud/archive/) section.

## Pages
{% for topic in site.topics %}
- [{{ topic.title }}]({{ topic.url | relative_url }})
{% endfor %}

## Posts 
{% for post in site.posts %}
- {{ post.date | date_to_string }}: [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}