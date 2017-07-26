---
layout: archive
permalink: /kafka/
excerpt: "kafka"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'kafka' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->
