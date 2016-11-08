---
layout: page
title: Tags 

---

<div class="page-content wc-container">
	<div class="sidebar">
		<h1>Tags</h1>  
		<ul>
			{% for tag in site.my_tags %}
              {% assign tag = tag[0] %}
              <li><a href="{{ tag.url }}">{{ tag.name }}</a></li>
			{% endfor %}
		</ul>
	</div>
</div>