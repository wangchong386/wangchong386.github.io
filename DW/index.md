---
layout: archive
permalink: /DW/
title: " *DW*"
excerpt: "It is a system used for reporting and data analysis, and is considered a core component of business intelligence"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'DW' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->
