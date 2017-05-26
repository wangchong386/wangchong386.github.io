---
layout: archive
permalink: /hadoop/
excerpt: "HDFS and mapreduce"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'hadoop' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->
