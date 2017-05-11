---
layout: archive
title: "Latest Posts in *hadoop*"
excerpt: "Hadoop的框架最核心的设计就是：HDFS和MapReduce"

---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'hadoop' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->


