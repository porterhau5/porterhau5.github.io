---
#
# Use the widgets beneath and the content will be
# inserted automagically in the webpage. To make
# this work, you have to use › layout: frontpage
#
layout: frontpage
header:
  image_fullwidth: "photo-1454165205744-3b78555e5572.jpeg"
widget1:
  title: "About"
  url: '/about/'
  image: WWHF1-303x182.jpg
  text: "Just a guy with a beard who enjoys reading, researching, solving, and sharing. And tacos."
widget2:
  title: "Blog"
  url: '/blog/'
  image: "photo-unsplash-kitsune-3-frontpage.jpg"  
  text: "Ramblings about red teaming, penetration testing, and the miscellaneous things I learned the hard way."
widget3:
  title: "Projects"
  url: '/projects/'
  image: widget-github-303x182.jpg
  text: "Some of the projects I've been working on, including documentation and example usage."
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
<meta name="twitter:image" content="https://porterhau5.com/images/twitter-meta-homepage.png">
