---
#
# Use the widgets beneath and the content will be
# inserted automagically in the webpage. To make
# this work, you have to use › layout: frontpage
#
layout: frontpage
header:
#  image_fullwidth: header_unsplash_12.jpg
  image_fullwidth: "photo-1454165205744-3b78555e5572.jpeg"
widget1:
  title: "Blog"
  url: '/blog/'
#  image: widget-1-302x182.jpg
#  image: "photo-39982a21-frontpage.jpeg"
#  image: "photo-6HJtLkJSDqjqlE2NipEu_macbook-frontpage.jpg"
  image: "photo-unsplash-kitsune-3-frontpage.jpg"  
#  image: "photo-1453928582365-b6ad33cbcf64-frontpage.jpeg"
  text: "Ramblings about penetration testing, scripts that make my life easier, and how bad the Braves will be this season."
widget2:
  title: "Projects"
  url: '/projects/'
  image: widget-github-303x182.jpg
  text: "Some of the projects I've been working on, including documentation and example usage."
widget3:
  title: "About"
  url: '/about/'
  image: about_pic.jpg
  text: 'Just a guy with a beard who likes to understand how things work. And tacos. Lots of tacos.'
#
# Use the call for action to show a button on the frontpage
#
# To make internal links, just use a permalink like this
# url: /getting-started/
#
# To style the button in different colors, use no value
# to use the main color or success, alert or secondary.
# To change colors see sass/_01_settings_colors.scss
#
#callforaction:
#  url: https://tinyletter.com/feeling-responsive
#  text: Inform me about new updates and features ›
#  style: alert
permalink: /index.html
#
# This is a nasty hack to make the navigation highlight
# this page as active in the topbar navigation
#
homepage: true
---
