---
layout: page
title: Tags 

---

<div class="page-content wc-container">
	<div class="post">
		<h1>Tags</h1>  
		<ul>
			{% for post_tag in site.tags %}
                {% assign tag = site.my_tags | where: "slug", post_tag %}
                {% if tag %}
                  {% assign tag = tag[0] %}
                  <li><a href="{{ tag.url }}">{{ tag.name }}</a></li>
                {% endif %}
			{% endfor %}
		</ul>
	</div>
</div>