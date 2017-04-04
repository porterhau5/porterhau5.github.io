---
layout: page
title:  "Representing Password Reuse in BloodHound"
teaser: "How to integrate password reuse attacks into BloodHound with the 'SharesPasswordWith' relationship. Includes a new Custom Query, API response logic parsing, detailed query output, and more."
categories:
    - blog
tags:
    - BloodHound
    - pen-testing
header: no
image:
    title: spw-banner.png
    thumb: spw-banner-thumb.png
author: porterhau5
---
This is another installment in a series of posts regarding modifications to BloodHound and lessons learned while working with Neo4j & Cypher. Other posts can be found here:

 * [Visualizing and Tracking Your Compromise](/blog/extending-bloodhound-track-and-visualize-your-compromise/)
 * [Creating Conditional Statements with Cypher](/blog/creating-conditional-statements-with-cypher/)

This post will cover a new relationship added to the BloodHound Owned extensions: `SharesPasswordWith`. Check out the <a href="https://github.com/porterhau5/BloodHound-Owned" target="_blank">GitHub BloodHound-Owned</a> project for the latest code update. Here's what it looks like in action:

<a href="{{ site.urlimg }}spw.gif" target="_blank"><img src="{{ site.urlimg }}spw.gif"></a>

 * [Local Accounts, Lazy Users, and Password Reuse](#local-accounts-lazy-users-and-password-reuse)
 * [The 'SharesPasswordWith' Relationship](#the-sharespasswordwith-relationship)
 * [Other New Features](#other-new-features)
   * [Error Checking](#error-checking)
   * [Detailed Output](#detailed-output)
   * [The Reset Flag](#the-reset-flag)
 * [Next Steps](#next-steps)

### Local Accounts, Lazy Users, and Password Reuse
My workflow during a penetration test leverages some non-Active Directory techniques, as I'd imagine most pen testers' would. One of the first actions I take after compromising a system is to dump the hashes of the local accounts. We often find that those accounts exist on other machines in the environment with the exact same password. It's a common technique for lateral movement and it's highly effective. <a href="https://twitter.com/simakov_marina" target="_blank">Marina Simakov</a> and <a href="https://twitter.com/TalBeerySec" target="_blank">Tal Be'ery</a> delivered a great presentation at BlueHat IL 2017 that dives deeper into the risks associated with local accounts. I encourage you to <a href="https://youtu.be/HE7X7l-k-A4" target="_blank">check it out here</a>.

However, lateral movement like this is not something that's guaranteed to work by just inspecting users and attributes -- granted you can get a pretty good idea by comparing PwdLastSet dates. Local accounts aren't necessarily backed by a centralized configuration overlord like Active Directory, so each machine's local RID 500 account isn't guaranteed to have the same password or username. Heck, it's not even guaranteed to be active. Maybe the admins disabled "Administrator" and created "corp_adm" with RID 1000 as the new local admin. Lucky for us red teamers, many Domain Admins haven't adopted <a href="https://technet.microsoft.com/en-us/mt227395.aspx" target="_blank">LAPS</a> yet, so they instead manually set the password for the local admin account (or even worse, use GPP to set it.) And they set that same password on several machines.

__Where else do we see commonly see passwords being reused?__ Users with multiple accounts. For example, Bob may have a normal, everyday account (INTERNAL\\bob) and a privileged account for administrator-level tasks (INTERNAL\\bob_admin). Or perhaps Bob has an account on two different domains (INTERNAL\\bob and EXTERNAL\\bob). Bob doesn't like the hassle of remembering two different passwords, so he sets the same password for each account. That's a bad Bob.

Abusing password reuse between separate accounts is a useful technique for expanding access across the network, but it's something that BloodHound didn't have a representation for quite yet. No fault of BloodHound -- its data collection script isn't designed to test for reused passwords, and that's a sensible approach given the inherit risks of doing so without human oversight. However this knowledge is valuable information for our attack graph, so I created a means to integrate this into BloodHound via a new relationship.

### The 'SharesPasswordWith' Relationship
I've added a new relationship in BloodHound to represent password reuse -- I call it 'SharesPasswordWith'. It's meant to be used as a relationship (or an "edge") between the following types of nodes:

 * (:Computer)-[:SharesPasswordWith]->(:Computer)
 * (:User)-[:SharesPasswordWith]->(:User)

The first point would cover common local administrator passwords on Computers. <br> The second point would cover Users reusing the same password for multiple accounts.

Adding this relationship for two given nodes via Cypher looks like this:
{% highlight plaintext %}
MATCH (n {name:$node1}),(m {name:$node2})
WITH n,m CREATE UNIQUE (n)-[:SharesPasswordWith]->(m)
WITH n,m CREATE UNIQUE (n)<-[:SharesPasswordWith]-(m)
{% endhighlight %}
Neo4j doesn't support bidirectional relationships, however we can make a unidirectional relationship in both directions. We architect it this way to allow paths in either direction for escalation. Don't worry, Neo4j is smart enough to avoid cycles and to calculate shortest paths despite this configuration.

The use of `CREATE UNIQUE` guarantees that we don't create duplicate relationships between nodes, so there's no harm in attempting to re-add the same relationship multiple times.

The simplest way to add these relationships is by using the `-s` flag in <a href="https://github.com/porterhau5/BloodHound-Owned" target="_blank">bh-owned.rb</a>:
{% highlight plaintext %}
$ ruby bh-owned.rb
Usage: ruby bh-owned.rb [options]
    -u, --username <id>    Neo4j database username (default: 'neo4j')
    -p, --password <pass>  Neo4j database password (default: 'BloodHound')
    -U, --url <url>        URL of Neo4j RESTful host  (default: 'http://127.0.0.1:7474/')
    -n, --nodes            get all node names
    -a, --add <file>       add 'owned' and 'wave' property to nodes in <file>
    -s, --spw <file>       add 'SharesPasswordWith' relationship between all nodes in <file>
    -w, --wave <num>       value to set 'wave' property (override default behavior)
        --reset            remove all custom properties and SharesPasswordWith relationships
    -e, --examples         reference doc of customized Cypher queries for BloodHound
{% endhighlight %}
The file passed to `-s` should be newline-delimited with one node name per line. All nodes listed in the file will have a 'SharesPasswordWith' relationship created between them, essentially creating a small mesh network of relationships.

Let's say we found four Computers sharing a common local administrator password:
{% highlight plaintext %}
$ cat common-local-admins.txt
MANAGEMENT3.INTERNAL.LOCAL
FILESERVER6.INTERNAL.LOCAL
SYSTEM38.INTERNAL.LOCAL
DESKTOP40.EXTERNAL.LOCAL
{% endhighlight %}
For a file containing 4 nodes we'd create a total of _`P(4,2)=12`_ relationships (or, "Choose 2 nodes from group of 4 nodes".) The output below shows the 6 unique pairs of nodes, and our query is creating the relationship in both directions, so we have a total of 12 relationships created. Yay combinatorics:
{% highlight plaintext %}
$ ruby bh-owned.rb -s common-local-admins.txt
[*] Using default username: neo4j
[*] Using default password: BloodHound
[*] Using default URL: http://127.0.0.1:7474/
[+] Created SharesPasswordWith relationship: 'MANAGEMENT3.INTERNAL.LOCAL' and 'FILESERVER6.INTERNAL.LOCAL'
[+] Created SharesPasswordWith relationship: 'MANAGEMENT3.INTERNAL.LOCAL' and 'SYSTEM38.INTERNAL.LOCAL'
[+] Created SharesPasswordWith relationship: 'MANAGEMENT3.INTERNAL.LOCAL' and 'DESKTOP40.EXTERNAL.LOCAL'
[+] Created SharesPasswordWith relationship: 'FILESERVER6.INTERNAL.LOCAL' and 'SYSTEM38.INTERNAL.LOCAL'
[+] Created SharesPasswordWith relationship: 'FILESERVER6.INTERNAL.LOCAL' and 'DESKTOP40.EXTERNAL.LOCAL'
[+] Created SharesPasswordWith relationship: 'SYSTEM38.INTERNAL.LOCAL' and 'DESKTOP40.EXTERNAL.LOCAL'
{% endhighlight %}
With the relationships added, we can see how the graph in BloodHound changes with our <font color="#C900FF">owned</font> nodes. Here's the custom query "Find Shortest Paths from owned nodes to Domain Admins":
<a href="{{ site.urlimg }}spw-example.png" target="_blank"><img src="{{ site.urlimg }}spw-example.png"></a>

You can replicate this by adding the first 3 waves in the `example-files` directory via `-a`, and then adding the `common-local-admins.txt` file via `-s`.

To supplement this, I also created a Custom Query to highlight all instances of password reuse added to the database -- **Find Clusters of Password Reuse** (<a href="https://github.com/porterhau5/BloodHound-Owned/commit/1dca29cd6ec5c297fe19b7e19bc9827246ff9d06" target="_blank">source here</a>):

<a href="{{ site.urlimg }}password-reuse-query.png" target="_blank"><img src="{{ site.urlimg }}password-reuse-query.png"></a>

Pretty self-explanatory. The Cypher query used to make this graph looks like:
{% highlight plaintext %}
MATCH p=(n)-[r:SharesPasswordWith]->(m) RETURN p
{% endhighlight %}

The graph can become a little hard to interpret when working with a high volume of nodes. I think a future enhancement will be to add a third layout type (joining Directed and Hierarchical) that shows a radial graph output for this query. Doing so will require updating the BloodHound source and learning more about Linkurious and Sigma.

### Other New Features
The latest code pushed to the <a href="https://github.com/porterhau5/BloodHound-Owned" target="_blank">repo</a> contained a couple of smaller additions worth noting.

##### Error Checking
I've focused on the response parsing logic for API requests so that more specific details can be echoed back to the user. For example, here's what happens when you try to add a node with the name misspelled:
{% highlight plaintext %}
$ ruby bh-owned.rb -a 1st-wave.txt
[*] Using default username: neo4j
[*] Using default password: BloodHound
[*] Using default URL: http://127.0.0.1:7474/
[*] No previously owned nodes found, setting wave to 1
[-] Properties not added for 'BLOPER@INT.LOC' (node not found, check spelling?)
[+] Success, marked 'JCARNEAL@INTERNAL.LOCAL' as owned in wave '1' via 'NBNS wpad'
[-] Skipping finding spread of compromise due to "node not found" error
{% endhighlight %}
Since an error was found during the property-adding process (the `MATCH` query returned no matches due to a misspelled name), then it won't run the follow-up query to find the spread of compromise for that wave. Once the typo is fixed, re-run the command and add the `-w` flag and the desired wave number:
{% highlight plaintext %}
$ ruby bh-owned.rb -a 1st-wave.txt -w 1
[*] Using default username: neo4j
[*] Using default password: BloodHound
[*] Using default URL: http://127.0.0.1:7474/
[+] Success, marked 'BLOPER@INTERNAL.LOCAL' as owned in wave '1' via 'LLMNR wpad'
[+] Success, marked 'JCARNEAL@INTERNAL.LOCAL' as owned in wave '1' via 'NBNS wpad'
[*] Finding spread of compromise for wave 1
[+] 2 nodes found:
DOMAIN USERS@INTERNAL.LOCAL
SYSTEM38.INTERNAL.LOCAL
{% endhighlight %}

Getting this granularity of detail in the API response required some Cypher expression hacks (<a href="https://github.com/porterhau5/BloodHound-Owned/commit/55b5b763db3332c34a10b8d16b45ba98ce45ce2c#diff-621720f457a37e54436019d9d9b9a6abR92" target="_blank">shown here</a>), probably better suited for a different blog post. I plan to write about it next week.

##### Detailed Output
As seen with the above command, the script will now echo back all nodes found in a wave of compromise. Useful for a quick glance at the collateral damage when a node is <font color="#C900FF">owned</font>:
{% highlight plaintext %}
$ ruby bh-owned.rb -a 2nd-wave.txt
[*] Using default username: neo4j
[*] Using default password: BloodHound
[*] Using default URL: http://127.0.0.1:7474/
[+] Success, marked 'ZDEVENS@INTERNAL.LOCAL' as owned in wave '2' via 'Password spray'
[+] Success, marked 'BPICKEREL@INTERNAL.LOCAL' as owned in wave '2' via 'Password spray'
[*] Finding spread of compromise for wave 2
[+] 5 nodes found:
BACKUP3@INTERNAL.LOCAL
BACKUP_SVC@INTERNAL.LOCAL
CONTRACTINGS@INTERNAL.LOCAL
DATABASE5.INTERNAL.LOCAL
MANAGEMENT3.INTERNAL.LOCAL
{% endhighlight %}
We see above that two nodes were marked as `owned` for wave 2. After finding the spread of compromise, we found 5 additional nodes in the wave.

##### The Reset Flag
In the event that you want to remove any custom properties (`owned`, `wave`) or any custom relationships (`SharesPasswordWith`) from the database, you can now use the `--reset` flag:
{% highlight plaintext %}
$ ruby bh-owned.rb --reset
[*] Using default username: neo4j
[*] Using default password: BloodHound
[*] Using default URL: http://127.0.0.1:7474/
[*] Removing all custom properties and SharesPasswordWith relationships
{% endhighlight %}

Underneath the hood it's issuing the following Cypher queries:
{% highlight plaintext %}
MATCH (n) WHERE exists(n.wave) OR exists(n.owned) REMOVE n.wave, n.owned
MATCH (n)-[r:SharesPasswordWith]-(m) DELETE r
{% endhighlight %}

### Next Steps
I'll post updates here and on <a href="https://twitter.com/porterhau5" target="_blank">Twitter via @porterhau5</a> as I continue to tinker with ideas. Please reach out if you want to explore some of these together! I'd really appreciate some help from those of you who are skilled with front-end development :D Feedback in any form is always welcome.

 * Interrogate Computers for list of local, non-AD admin accounts. Add those accounts to database as User nodes with AdminTo corresponding Computer.
 * Instead of running custom queries through the Queries tab, have a slider on the main dashboard that can step through waves.
 * Define a "Critical Path to Compromise" (CPTC): the exact path taken through the network to go from A to Z. Add property to nodes indicating their involvement in the CPTC. Write custom query to show this path specifically and filter out the rest of the noise. Mostly useful for presenting/explaining to client.
 * Add more options when a node is right-clicked. For example, "Shortest Paths from Here", "Add to wave X", or "Created SharesPasswordWith relationship to node X".
 * Make "Owned in Wave" and "Owned via Method" values fillable from the UI. Ditch the need for an external script to ingest data.
 * General query optimizations to help with scalability & speed. The graphs can get a little messy when a DA account is obtained.

Thank you for reading!

#### Share
<ul class="share-buttons">
  <li><a href="https://www.facebook.com/sharer/sharer.php?u=http%3A%2F%2Fporterhau5.com%2Fblog%2Frepresenting-password-reuse-in-bloodhound%2F&t=Representing%20Password%20Reuse%20in%20BloodHound" title="Share on Facebook" target="_blank"><img alt="Share on Facebook" src="{{ site.urlimg }}flat_web_icon_set/black/Facebook.png"></a></li>
  <li><a href="https://twitter.com/intent/tweet?source=http%3A%2F%2Fporterhau5.com%2Fblog%2Frepresenting-password-reuse-in-bloodhound%2F&text=Representing%20Password%20Reuse%20in%20BloodHound:%20http%3A%2F%2Fporterhau5.com%2Fblog%2Frepresenting-password-reuse-in-bloodhound%2F" target="_blank" title="Tweet"><img alt="Tweet" src="{{ site.urlimg }}flat_web_icon_set/black/Twitter.png"></a></li>
  <li><a href="https://getpocket.com/save?url=http%3A%2F%2Fporterhau5.com%2Fblog%2Frepresenting-password-reuse-in-bloodhound%2F&title=Representing%20Password%20Reuse%20in%20BloodHound" target="_blank" title="Add to Pocket"><img alt="Add to Pocket" src="{{ site.urlimg }}flat_web_icon_set/black/Pocket.png"></a></li>
  <li><a href="http://www.reddit.com/submit?url=http%3A%2F%2Fporterhau5.com%2Fblog%2Frepresenting-password-reuse-in-bloodhound%2F&title=Representing%20Password%20Reuse%20in%20BloodHound" target="_blank" title="Submit to Reddit"><img alt="Submit to Reddit" src="{{ site.urlimg }}flat_web_icon_set/black/Reddit.png"></a></li>
</ul>
