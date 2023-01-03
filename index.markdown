---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

## Welcome on teknews blog 2.0 ^^
  
This is my blog about IT stuff, mainly in the Cloud.
It's called **2.0** because I used to have a wordpress, but not anymore ^^.

Old articles from the WordPress era are available as archive in pdf in the [Archive](https://blog.teknews.cloud/archive/) section.

## Topics
{% for topic in site.topics %}
- [{{ topic.title }}]({{ topic.url | relative_url }})
{% endfor %}
