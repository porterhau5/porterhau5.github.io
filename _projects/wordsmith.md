---
layout: page-fullwidth
title: "Wordsmith"
teaser: "Crafting tailored wordlists for password cracking"
---
<div class="row">
<div class="medium-4 medium-push-8 columns" markdown="1">
<div class="panel radius" markdown="1">
**Table of Contents**
{: #toc }
*  TOC
{:toc}
</div>
</div><!-- /.medium-4.columns -->



<div class="medium-8 medium-pull-4 columns" markdown="1">
<!-- {% include _improve_content.html %} -->

## Source Code   {#source}

Latest development can be found on <a href="https://github.com/skahwah/wordsmith" target="_blank">GitHub</a>.

## Purpose  {#purpose}

The aim of Wordsmith is to assist with creating tailored, geolocation-based wordlists. As of v1.0, this is primarily based on U.S. states. Attributes for each state include roads, cities, colleges, area codes, zip codes, sports teams, and landmarks. Wordsmith also includes support for basic word mangling and filtering.

v2.0 will expand Wordsmith to include other countries, territories, and provinces.

## Presentation  {#presentation}
Wordsmith was introduced at BSidesLV 2016. A copy of the presentation slides from the talk can be found here: <a href="http://www.slideshare.net/SanjivKawa/crafting-tailored-wordlists-with-wordsmith" target="_blank">http://www.slideshare.net/SanjivKawa/crafting-tailored-wordlists-with-wordsmith</a>

## Usage  {#usage}

On first run, Wordsmith will unpack some files. This will take less than 5 seconds.
Alternatively you can run wordsmith.rb with the update option and download 175 MB of data from the internet.

{% highlight plaintext %}
$ ruby wordsmith.rb -h
wordsmith v1.0
Written by: Sanjiv "Trashcan Head" Kawa & Tom "Pain Train" Porter
Twitter: @skawasec & @porterhau5

Usage: ruby wordsmith.rb [options]
Main Arguments:
    -s, --state <states>     Comma-delimited list of US states
State Options:
    -a, --all                Grab everything for the specified state
    -c, --cities             Grab all city names for the specified state
    -f, --colleges           Grab all college sports for the specified state
    -l, --landmarks          Grab all landmarks for the specified state
    -p, --phone              Grab all area codes for the specified state
    -r, --roads              Grab all road names in the specified state
    -t, --teams              Grab all major sports teams in the specified state
    -z, --zip                Grab all zip codes for the specified state
Miscellaneous Options:
    -d, --domain DOMAIN      Set a URL for a web application that you want CeWL to scrape
    -e, --examples           Show some usage examples
    -i, --infile FILE        Supply a file containing multiple URLs that you want CeWL to scrape
    -n, --names              Grab the most common male, female, baby and last names in the USA
Output Options:
    -o, --output FILE        The name of the output file
    -q, --quiet              Don't show words generated, use with -o option
    -k, --length LEN         Minimum length of word to include
    -j, --lowercase          Convert all words to lowercase
    -w, --specials           Add words with special characters removed
    -x, --spaces             Add words with spaces removed
    -y, --split              Split words by space and add
    -m, --mangle             Add all permutations (-w, -x, -y)
Management:
    -u, --update             Update data from Internet sources
{% endhighlight %}

#### Command Examples
{% highlight plaintext %}
$ ruby wordsmith.rb -e
wordsmith v1.0
Written by: Sanjiv "Trashcan Head" Kawa & Tom "Pain Train" Porter
Twitter: @skawasec & @porterhau5

Grab all of the cities and towns for California
    ruby wordsmith.rb -s CA -c

Grab all of the cities for California, Montana, and Florida
    ruby wordsmith.rb -s CA,MT,FL -c

Grab all sports teams for California, mangle the output
    ruby wordsmith.rb -s CA -t -m

Grab all road names for California, mangle the output, convert to lowercase
    ruby wordsmith.rb -s CA -r -m -j

Grab all landmarks for California with a minimum character length of 8
    ruby wordsmith.rb -s CA -l -k 8

Grab everything for California, write to file named CA.txt
    ruby wordsmith.rb -s CA -a -o CA.txt

Create a mega wordlist containing all states with all options, quiet output, write to file named all.txt
    ruby wordsmith.rb -s all -m -q -o all.txt

Run CeWL against https://www.popped.io, mangle the output
    ruby wordsmith.rb -d https://www.popped.io -m

Run CeWL against list of URLs contained in urls.txt, write to file out.txt
    ruby wordsmith.rb -i urls.txt -m -o out.txt

Grab the most common male, female, baby and last names in the USA
    ruby wordsmith.rb -n
{% endhighlight %}


## Dependencies  {#dependencies}
A Gemfile has been included to simplify gem installation. These can be installed using `bundle install`. Alternatively, each gem can be installed manually using `gem install <gem>`.

Wordsmith uses data that's been compressed in data.tar.gz. On first run, Wordsmith will unpack this to a directory called "data/" in the current working directory. This can be circumvented manually using `tar -xf data.tar.gz`.

Two of Wordsmith's options, -d and -i, use CeWL to scrape words from user-supplied URLs. Wordsmith assumes the CeWL executable (cewl) is on the user's PATH. If cewl is not found, Wordsmith will skip the URLs and continue. Instructions for installing CeWL can be found in Robin Wood's CeWL repository: <a href="https://github.com/digininja/CeWL" target="_blank">https://github.com/digininja/CeWL</a>



</div><!-- /.medium-8.columns -->
</div><!-- /.row -->

 [1]: http://kramdown.gettalong.org/converter/html.html#toc
 [2]: {{ site.url }}/blog/
 [3]: http://srobbin.com/jquery-plugins/backstretch/
 [4]: #
 [5]: #
 [6]: #
 [7]: #
 [8]: #
 [9]: #
 [10]: #
