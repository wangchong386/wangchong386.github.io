---
layout: archive
permalink: /DataWarehouse/
title: "Latest Posts in *DataWarehouse*"
excerpt: "It is a system used for reporting and data analysis, and is considered a core component of business intelligence"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'DataWarehouse' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->
