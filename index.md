---
layout : layout
title : SiteName
---

<div class="post-list">
	<ul>
		{% for post in site.posts limit:10 %}
		<li>
			<a href="{{ post.url }}">{{ post.title }}</a>
			<span class="date">{{ post.date | date: "%e %B, %Y" }}</span>
		</li>			
		{% endfor %}
	</ul>
</div>

<!--
{% assign x = site.posts.size %}
{{ x }}

{% if x > 10 %}A{% else %}B{% endif %}
-->
