---
layout: page
title: Tags 

---

<div class="page-content wc-container">
	<div class="post">
		<h1>Tags</h1>  
		<ul>
			{% for tag in site.tags %}
			<li><a href="{{ tag.url }}">{{ tag.name }}</a></li>
			{% endfor %}
		</ul>
	</div>
</div>