---
layout: page
title:  "Creating Conditional Statements with Cypher"
teaser: "How to hack together Neo4j's Cypher statements to conditionally execute code, along with examples of working with API response metadata."
categories:
    - blog
tags:
    - BloodHound
    - Neo4j
    - Cypher
header: no
image:
    title: cypher-conditional-banner.png
    thumb: cypher-conditional-thumb.png
author: porterhau5
---
This is another installment in a series of posts regarding modifications to BloodHound and lessons learned while working with Neo4j & Cypher. Other posts can be found here:

 * [Visualizing and Tracking Your Compromise](/blog/extending-bloodhound-track-and-visualize-your-compromise/)
 * [Representing Password Reuse in BloodHound](/blog/representing-password-reuse-in-bloodhound/)

This post will cover some advanced Neo4j concepts and how I hacked Cypher commands together to improve feedback on the <a href="https://github.com/porterhau5/BloodHound-Owned" target="_blank">BloodHound Owned extensions</a> project. I'll specifically cover how to create conditional statements in Cypher by combining a `CASE` expression and `FOREACH` clause. Although the examples are in context of BloodHound, I hope Neo4j & Cypher users in general find it helpful.

This post doesn't cover introductory Cypher concepts. I _highly_ recommend reading Rohan Vazarkar's <a href="https://blog.cptjesus.com/posts/introtocypher" target="_blank">Intro to Cypher</a> blog post as a primer for Cypher and its usage in BloodHound. One of the concepts covered can also be read about on this <a href="http://www.markhneedham.com/blog/2014/08/22/neo4j-load-csv-handling-empty-columns/" target="_blank">blog</a>.

 * [The Query Objective](#the-query-objective)
 * [Trial and Error with Response Metadata](#trial-and-error-with-response-metadata)
 * [Creating Conditional Statements with Cypher](#creating-conditional-statements-with-cypher)
 * [Providing Detailed Feedback](#providing-detailed-feedback)

### The Query Objective
This came about while I was trying to figure out how to provide granular feedback to Cypher queries issued by <a href="https://github.com/porterhau5/BloodHound-Owned/blob/master/bh-owned.rb" target="_blank">bh-owned.rb</a>. I wanted users to know exactly what happened on the backend after running a command, that way they can tweak their input if needed.

One of the script options, `-a`, adds custom properties to nodes in the Neo4j database. Three inputs are used for this:

 * `name` - name of the node, used to find the node we want to update
 * `owned` - method used to compromise the node, a property to add
 * `wave` - wave number, a property to add

There are three potential outcomes when adding custom properties that I want to track and report back to the user:

 1. The node doesn't exist (maybe because the 'name' is misspelled)
 2. The node already has the 'wave' property set, no changes are made
 3. The node doesn't have the 'wave' property set, so two custom properties are set

I want a single query I can POST to the Neo4j REST endpoint. I also want enough detail in the JSON response so I know which of these three outcomes actually transpired.

### Trial and Error with Response Metadata

Neo4j's REST API has a parameter called <a href="https://neo4j.com/docs/rest-docs/current/#rest-api-retrieve-query-metadata" target="_blank">includeStats</a> that you can tack on to requests. When added, it will include some metadata about the transaction in the JSON response. This includes statistics like the number of nodes created, relationships deleted, and properties set. We'll use this to help us determine what happened on the backend after a user POSTs a Cypher query.

For this example, let's say I want to add User "BLOPER@INTERNAL.LOCAL" to wave "1" via "LLMNR".

My first query attempt looked like this. The JSON response with 'includeStats' is shown below:
{% highlight plaintext %}
MATCH (n) WHERE (n.name = "BLOPER@INTERNAL.LOCAL")
SET n.owned = "LLMNR", n.wave = 1
RETURN 'BLOPER@INTERNAL.LOCAL','1','LLMNR'
---
{
  "results": [
    {
      "columns": [
        "'BLOPER@INTERNAL.LOCAL'",
        "'1'",
        "'LLMNR'"
      ],
      "data": [
        {
          "row": [
            "BLOPER@INTERNAL.LOCAL",
            "1",
            "LLMNR"
          ],
          "meta": [
            null,
            null,
            null
          ]
        }
      ],
      "stats": {
        "contains_updates": true,
        "nodes_created": 0,
        "nodes_deleted": 0,
        "properties_set": 2,
        "relationships_created": 0,
        "relationship_deleted": 0,
        "labels_added": 0,
        "labels_removed": 0,
        "indexes_added": 0,
        "indexes_removed": 0,
        "constraints_added": 0,
        "constraints_removed": 0
      }
    }
  ],
  "errors": [

  ]
}
{% endhighlight %}
Note that "properties_set" is 2. So this query sets the properties that I want set, but what happens if I run it again? It'll overwrite properties that already exist. I don't want that. I could fix this by adding an `AND` to filter out the nodes that I'd like to skip:
{% highlight plaintext %}
MATCH (n) WHERE (n.name = "BLOPER@INTERNAL.LOCAL")
AND not(exists(n.wave))
SET n.owned = "LLMNR", n.wave = 1
RETURN 'BLOPER@INTERNAL.LOCAL','1','LLMNR'
---
{
  "results": [
    {
      "columns": [
        "'BLOPER@INTERNAL.LOCAL'",
        "'1'",
        "'LLMNR'"
      ],
      "data": [

      ],
      "stats": {
        "contains_updates": false,
        "nodes_created": 0,
        "nodes_deleted": 0,
        "properties_set": 0,
        "relationships_created": 0,
        "relationship_deleted": 0,
        "labels_added": 0,
        "labels_removed": 0,
        "indexes_added": 0,
        "indexes_removed": 0,
        "constraints_added": 0,
        "constraints_removed": 0
      }
    }
  ],
  "errors": [

  ]
}

{% endhighlight %}
Great, this sets the two properties I want and it doesn't overwrite existing ones. Notice that "properties_set" was 0 because this node already had the custom properties set.

But what happens if we don't find the node? For example, if 'name' was misspelled or if a node with 'name' was never added?
{% highlight plaintext %}
{
  "results": [
    {
      "columns": [
        "'NLOPER@INTERNAL.LOCAL'",
        "'1'",
        "'LLMNR'"
      ],
      "data": [

      ],
      "stats": {
        "contains_updates": false,
        "nodes_created": 0,
        "nodes_deleted": 0,
        "properties_set": 0,
        "relationships_created": 0,
        "relationship_deleted": 0,
        "labels_added": 0,
        "labels_removed": 0,
        "indexes_added": 0,
        "indexes_removed": 0,
        "constraints_added": 0,
        "constraints_removed": 0
      }
    }
  ],
  "errors": [

  ]
}
{% endhighlight %}
It's the exact same response (except for the name being misspelled). There's no way to differentiate between a "node not found" error and a "properties already exists" scenario. That's not helpful to a user.

I want to conditionally set properties, but I need a way to discern between the three scenarios. Here's where we get creative.

### Creating Conditional Statements with Cypher
Cypher doesn't support full-blown conditional statements. We can't directly express something like `if a.x > 0, then SET a.y=1, else SET a.y=0, a.z=1`. We can get close with the `CASE` statement, which acts a lot like it does in the SQL world (example from <a href="https://neo4j.com/docs/developer-manual/current/cypher/syntax/expressions/" target="_blank">here</a>):
{% highlight plaintext %}
MATCH (n)
RETURN
CASE
  WHEN n.eyes = 'blue'
    THEN 1
  WHEN n.age < 40
    THEN 2
  ELSE 3
END AS result
{% endhighlight %}
The problem is that `CASE` is limited to returning a literal expression. We can't put clauses like `SET` or `MATCH` inside a `THEN`. That makes is difficult to do something like setting a property conditionally.

What we can do though is nest a `CASE` statement inside a `FOREACH` clause (<a href="https://neo4j.com/docs/developer-manual/current/cypher/clauses/foreach/" target="_blank">FOREACH documentation here</a>). `FOREACH` will loop through a list or a path and pass each matching element to a clause (like `SET`):
{% highlight plaintext %}
MATCH p =(begin)-[*]->(last)
WHERE begin.name = 'A' AND last.name = 'D'
FOREACH (n IN nodes(p) | SET n.marked = TRUE)
{% endhighlight %}
This is nifty, but we only want to perform clauses under certain conditions, like when a property does or doesn't exist.

<u><b>This is the hack</b></u>: we'll use a `CASE` statement to either return an empty list or a list with one element. That result is passed to a `FOREACH` loop. If the result is an empty list, then the `FOREACH` clause won't execute (because there's nothing to iterate over.) If the result is a list with one element, then the `FOREACH` clause will execute once (because it iterated over one element.)

Let's see it in context. This is how I ultimately ended up crafting the Cypher query (and here it is in <a href="https://github.com/porterhau5/BloodHound-Owned/blob/master/bh-owned.rb#L92" target="_blank">bh-owned.rb</a>):
{% highlight plaintext %}
1. MATCH (n) WHERE (n.name = 'BLOPER@INTERNAL.LOCAL')
2. FOREACH (ignoreMe in CASE
3.   WHEN exists(n.wave) THEN [1]
4.   ELSE [] END | SET n.wave=n.wave)
5. FOREACH (ignoreMe in CASE
6.   WHEN not(exists(n.wave)) THEN [1]
7.   ELSE [] END | SET n.owned = 'LLMNR', n.wave = 1)
8. RETURN 'BLOPER@INTERNAL.LOCAL','1','LLMNR'
{% endhighlight %}
A lot going on here, but let's take it line-by-line:

`1. MATCH (n) WHERE (n.name = 'BLOPER@INTERNAL.LOCAL')` : Match nodes where the 'name' property is equal to 'BLOPER@INTERNAL.LOCAL', store node in variable named 'n'.

`2. FOREACH (ignoreMe in CASE` : FOREACH will loop through a list or path. We're not passing in a list or path directly though -- we're passing in the result of a CASE statement. The result will either be a list with one element (if true) or an empty list (if false). As the name suggests, we can ignore the 'ignoreMe' variable because we'll never use it, we just need it there to satisfy syntax.

`3. WHEN exists(n.wave) THEN [1]` : Here's where we can specify the "if" conditional. If the 'wave' property already exists for our node, then pass a list with one element (`[1]`) back to the FOREACH clause. This means we want to execute the clause (`| SET n.wave=n.wave`) once. This is our "true" condition.

`4. ELSE [] END | SET n.wave=n.wave)` : If the 'wave' property doesn't exist for our node, then pass an empty list (`[]`) to the FOREACH clause. This is our "false" condition. With an empty list, no iteration happens. That means the clause `| SET n.wave=n.wave` won't execute. However, if the list is not empty (`[1]`) then we execute the clause once and set 'n.wave' equal to itself (I'll explain why later, just remember that we only set one property.)

`5. FOREACH (ignoreMe in CASE` : Again, we'll loop through the result of a CASE statement (which will return an empty list or list with one element).

`6. WHEN not(exists(n.wave)) THEN [1]` : If the 'wave' property doesn't exist for our node, then we'll return a list with one element and ensure we execute the clause.

`7. ELSE [] END | SET n.owned = 'LLMNR', n.wave = 1)` : If the 'wave' property does exist, then return an empty list (`[]`) and skip execution of the clause. However if we return a one-element list (`[1]`) then the 'wave' property doesn't exist and we'll execute `| SET n.owned = 'LLMNR', n.wave = 1`. In this example, we set two properties.

`8. RETURN 'BLOPER@INTERNAL.LOCAL','1','LLMNR'` : The primary goal of conditionally setting properties is already done by this point. We RETURN the inputs (name, method, wave) so we can make parsing the API response easier.

### Providing Detailed Feedback

With this query, we can now differentiate between the three different potential outcomes by inspecting "properties_set" in the API response:

 1. If "properties_set" is 0, then the node with 'name' wasn't found, so the FOREACH statements didn't execute. This happens when 'name' is misspelled.
 2. If "properties_set" is 1, then node was found and the 'wave' property already exists, but we didn't overwrite it. We need something to distinguish this scenario from the third one below, so we set 'n.wave=n.wave' which technically means we set one property.
 3. If "properties_set" is 2, then the node was found and the 'wave' property didn't exist, so we created the 'wave' and 'owned' properties.

Here's what each scenario looks like when using `bh-owned.rb`. Take note of the "properties_set" key in the JSON responses.

**Scenario 1** -- Misspelled node name. "properties_set" is 0. Don't run the follow-up query to find the spread of compromise for a node:
{% highlight plaintext %}
$ ruby bh-owned.rb -a example-wave.txt
[*] Using default username: neo4j
[*] Using default password: BloodHound
[*] Using default URL: http://127.0.0.1:7474/
[*] No previously owned nodes found, setting wave to 1
[-] Properties not added for 'NLOPER@INTERNAL.LOCAL' (node not found, check spelling?)
[-] Skipping finding spread of compromise due to "node not found" error
{
 "results": [
   {
     "columns": [
       "'NLOPER@INTERNAL.LOCAL'",
       "'1'",
       "'LLMNR'"
     ],
     "data": [

     ],
     "stats": {
       "contains_updates": false,
       "nodes_created": 0,
       "nodes_deleted": 0,
       "properties_set": 0,
       "relationships_created": 0,
       "relationship_deleted": 0,
       "labels_added": 0,
       "labels_removed": 0,
       "indexes_added": 0,
       "indexes_removed": 0,
       "constraints_added": 0,
       "constraints_removed": 0
     }
   }
 ],
 "errors": [

 ]
}
{% endhighlight %}

**Scenario 2** -- Node found and 'wave' property already exists. "properties_set" is 1:
{% highlight plaintext %}
$ ruby bh-owned.rb -a example-wave.txt
[*] Using default username: neo4j
[*] Using default password: BloodHound
[*] Using default URL: http://127.0.0.1:7474/
[*] Properties already exist for 'BLOPER@INTERNAL.LOCAL', skipping (overwrite with flag -w <num>)
[*] Finding spread of compromise for wave 2
[-] No additional nodes found for wave 2
{
  "results": [
    {
      "columns": [
        "'BLOPER@INTERNAL.LOCAL'",
        "'2'",
        "'LLMNR'"
      ],
      "data": [
        {
          "row": [
            "BLOPER@INTERNAL.LOCAL",
            "2",
            "LLMNR"
          ],
          "meta": [
            null,
            null,
            null
          ]
        }
      ],
      "stats": {
        "contains_updates": true,
        "nodes_created": 0,
        "nodes_deleted": 0,
        "properties_set": 1,
        "relationships_created": 0,
        "relationship_deleted": 0,
        "labels_added": 0,
        "labels_removed": 0,
        "indexes_added": 0,
        "indexes_removed": 0,
        "constraints_added": 0,
        "constraints_removed": 0
      }
    }
  ],
  "errors": [

  ]
}
{% endhighlight %}

**Scenario 3** -- Node found and 'wave' property doesn't exist. "properties_set" is 2:
{% highlight plaintext %}
$ ruby bh-owned.rb -a example-wave.txt
[*] Using default username: neo4j
[*] Using default password: BloodHound
[*] Using default URL: http://127.0.0.1:7474/
[*] No previously owned nodes found, setting wave to 1
[+] Success, marked 'BLOPER@INTERNAL.LOCAL' as owned in wave '1' via 'LLMNR'
[*] Finding spread of compromise for wave 1
[+] 2 nodes found:
DOMAIN USERS@INTERNAL.LOCAL
SYSTEM38.INTERNAL.LOCAL
{
  "results": [
    {
      "columns": [
        "'BLOPER@INTERNAL.LOCAL'",
        "'1'",
        "'LLMNR'"
      ],
      "data": [
        {
          "row": [
            "BLOPER@INTERNAL.LOCAL",
            "1",
            "LLMNR"
          ],
          "meta": [
            null,
            null,
            null
          ]
        }
      ],
      "stats": {
        "contains_updates": true,
        "nodes_created": 0,
        "nodes_deleted": 0,
        "properties_set": 2,
        "relationships_created": 0,
        "relationship_deleted": 0,
        "labels_added": 0,
        "labels_removed": 0,
        "indexes_added": 0,
        "indexes_removed": 0,
        "constraints_added": 0,
        "constraints_removed": 0
      }
    }
  ],
  "errors": [

  ]
}
{% endhighlight %}

And that's one way to do conditional statements with Cypher :D

<a href="{{ site.urlimg }}lolcypher.png" target="_blank"><img src="{{ site.urlimg }}lolcypher.png"></a>

(Sorry. I couldn't write about hacking, Cypher, and <u>Neo</u>4j without making at least one Matrix reference. <a href="http://www.cracked.com/article_19435_6-movie-plot-holes-you-never-noticed-thanks-to-editing.html" target="_blank">#CypherIsReallyTheOne</a>)

Thank you for reading!

#### Share
<ul class="share-buttons">
  <li><a href="https://www.facebook.com/sharer/sharer.php?u=http%3A%2F%2Fporterhau5.com%2Fblog%2Fcreating-conditional-statements-with-cypher%2F&t=Creating%20Conditional%20Statements%20with%20Cypher" title="Share on Facebook" target="_blank"><img alt="Share on Facebook" src="{{ site.urlimg }}flat_web_icon_set/black/Facebook.png"></a></li>
  <li><a href="https://twitter.com/intent/tweet?source=http%3A%2F%2Fporterhau5.com%2Fblog%2Fcreating-conditional-statements-with-cypher%2F&text=Creating%20Conditional%20Statements%20with%20Cypher:%20http%3A%2F%2Fporterhau5.com%2Fblog%2Fcreating-conditional-statements-with-cypher%2F" target="_blank" title="Tweet"><img alt="Tweet" src="{{ site.urlimg }}flat_web_icon_set/black/Twitter.png"></a></li>
  <li><a href="https://getpocket.com/save?url=http%3A%2F%2Fporterhau5.com%2Fblog%2Fcreating-conditional-statements-with-cypher%2F&title=Creating%20Conditional%20Statements%20with%20Cypher" target="_blank" title="Add to Pocket"><img alt="Add to Pocket" src="{{ site.urlimg }}flat_web_icon_set/black/Pocket.png"></a></li>
  <li><a href="http://www.reddit.com/submit?url=http%3A%2F%2Fporterhau5.com%2Fblog%2Fcreating-conditional-statements-with-cypher%2F&title=Creating%20Conditional%20Statements%20with%20Cypher" target="_blank" title="Submit to Reddit"><img alt="Submit to Reddit" src="{{ site.urlimg }}flat_web_icon_set/black/Reddit.png"></a></li>
</ul>
