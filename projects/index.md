---
layout: default
title: "Projects"
header:
  image_fullwidth: "photo-1454165205744-3b78555e5572.jpeg"
permalink: "/projects/"

#layout: projectindex
#header:
#  image_fullwidth: header_unsplash_12.jpg
#widget1:
#  title: "Sleat"
#  url: '/projects/sleat.html'
#  image: widget-1-302x182.jpg
#  text: "A collection of scripts for collecting, parsing, and analyzing Windows Logon Events."
#widget2:
#  title: "Projects"
#  url: '/projects/'
#  image: widget-github-303x182.jpg
#  text: "Some of the projects I've been working on, including documentation and example usage."
#widget3:
#  title: "About"
#  url: '/about.html'
#  image: about_pic.jpg
#  text: 'Just a guy with a beard who likes to understand how things work. And tacos. Lots of tacos.'
#widget4:
#  title: "Four"
#  url: '/about.html'
#  image: about_pic.jpg
#  text: 'Four stuff text and description.'
---

<div id="project-index" class="row">
	<div class="small-12 columns t30">
		<h1>{{ page.title }}</h1>
		{% if page.teaser %}<p class="teaser">{{ page.teaser }}</p>{% endif %}

		<dl class="accordion" data-accordion>
			{% assign counter = 1 %}
			{% for post in site.projects limit:1000 %}
			<dd class="accordion-navigation">
			<a href="#panel{{ counter }}"><span class="iconfont"></span> {% if post.subheadline %}{{ post.subheadline }} › {% endif %}<strong>{{ post.title }}</strong></a>
				<div id="panel{{ counter }}" class="content">
					{% if post.meta_description %}{{ post.meta_description | strip_html | escape }}{% elsif post.teaser %}{{ post.teaser | strip_html | escape }}{% endif %}
					<a href="{{ site.url }}{{ post.url }}" title="Read {{ post.title escape_once }}"><strong>{{ site.data.language.read_more }}</strong></a><br><br>
				</div>
			</dd>
			{% assign counter=counter | plus:1 %}
			{% endfor %}
		</dl>
	</div>
</div>
