---
layout : layout
title : Michał Sakowicz
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