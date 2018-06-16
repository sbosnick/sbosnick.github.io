---
---
# Posts

{% for post in site.posts %}
{{post.date | date_to_string }}
### [{{post.title}}]({{post.url | relative_url}})
{{post.excerpt | strip_html | truncatewords: 40}}
{% endfor %}
