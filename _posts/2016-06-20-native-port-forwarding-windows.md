---
layout: page
title:  "Pivoting in Windows Using Native Port Forwarding"
teaser: "Harness the power of netsh interface portproxy to natively pivot on Windows and stay under the AV radar during a penetration test."
categories:
    - blog
tags:
    - windows
    - pen-testing
    - pivoting
header: no
image:
    title: photo-protractor.jpeg
    thumb: photo-protractor-thumb.jpeg
author: porterhau5
---
Ever wanted to route traffic through a compromised Windows host without going through the hassle of setting up a Meterpreter session or uploading files which may get flagged by endpoint protection? `netsh interface portproxy` is baked right into Windows and has everything you need to set up a pivot point. It works by directing incoming traffic on a specified host:port to a destination host:port. Very simple, and very effective.

### Command

{% highlight html %}
C:\>netsh interface portproxy add v4tov4 listenport=<lport>
listenaddress=<lhost> connectport=<rport> connectaddress=<rhost>

<lport> - local port to listen on
<lhost> - local address to bind to
<rport> - remote port
<rhost> - remote host
{% endhighlight %}

Example:
{% highlight html %}
C:\>netsh interface portproxy add v4tov4 listenport=1194
listenaddress=0.0.0.0 connectport=8080 connectaddress=10.10.10.20

{% endhighlight %}

This would bind locally to `0.0.0.0` (all network interfaces) on `1194/tcp` and route incoming connections to remote host `10.10.10.20:8080`. This particular example might be used to reach a web application that's sitting behind a restrictive firewall.

### Why this technique?

This pivoting technique is an example of living off the land to expand access with the added benefit of stealth. It comes in handy when options are limited or when there's a genuine concern of being noticed by AV or HIPS sitting on the endpoint. Using `netsh` to temporarily modify firewall tables is much less likely to be flagged by one of these products than a malicious or unknown executable, and it requires no uploading of software to the target host.

From a detection standpoint, this technique demonstrates the need for baselining. Comparing network traffic and system behavior against a solid baseline in order to detect anomalous behavior should be part of any proper defense-in-depth strategy. Unfortunately, this is a concept foreign to many security teams. A <a href="https://twitter.com/pfizzell" target="_blank">colleague of mine</a> and I are currently researching and developing a simple tool to help organizations capture these kind of baselines - more info to come as we develop.

### Scenario

To give this technique a bit more context, here's how I utilized this feature during a recent engagement:

I had obtained admin creds and had access to a Domain Controller via SMB. An Nmap scan showed that the network firewall allowed me to also get to 88/tcp, 389/tcp, and some of the high-range TCP ports (49152, 49153, etc.) on the DC as well. However, the network firewall didn’t allow me to reach back to my attack host, so reverse connections were out of the question. I couldn’t reach RDP, and all quick attempts to get a Meterpreter session going failed. I tried altering the local firewall, but it got me nowhere due to the restrictive network firewall rules. I really wanted to use this DC as a pivot point to get to some juicy targets that I suspected had RDP exposed.

Within my <a href="https://sourceforge.net/projects/winexe/" target="_blank">winexe</a> session, I ran netstat on the DC (10.10.10.10) to figure out which high-range TCP port would be available for me to bind to:
{% highlight html %}
C:\Windows\system32>netstat -an

Active Connections

  Proto  Local Address        Foreign Address      State
  TCP    0.0.0.0:88           0.0.0.0:0            LISTENING
  TCP    0.0.0.0:111          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:135          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:389          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:445          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:464          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:593          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:610          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:636          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:1688         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:3268         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:3269         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:3389         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:5722         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:6677         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:9389         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:9898         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:47001        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49152        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49153        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49154        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49155        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49158        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49159        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49160        0.0.0.0:0            LISTENING
{% endhighlight %}

I decided to bind to `0.0.0.0:49162`, seeing as it was unused and I could most likely reach it through the network firewall. I used the `netsh interface portproxy` command to forward my traffic to a database server located at `10.10.10.20:3389`:

{% highlight html %}
C:\Windows\system32>netsh interface portproxy add v4tov4 listenport=49162
listenaddress=0.0.0.0 connectport=3389 connectaddress=10.10.10.20

C:\Windows\system32>netstat -an

Active Connections

  Proto  Local Address        Foreign Address      State
  TCP    0.0.0.0:88           0.0.0.0:0            LISTENING
  TCP    0.0.0.0:111          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:135          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:389          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:445          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:464          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:593          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:610          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:636          0.0.0.0:0            LISTENING
  TCP    0.0.0.0:1688         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:3268         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:3269         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:3389         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:5722         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:6677         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:9389         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:9898         0.0.0.0:0            LISTENING
  TCP    0.0.0.0:47001        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49152        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49153        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49154        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49155        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49158        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49159        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49160        0.0.0.0:0            LISTENING
  TCP    0.0.0.0:49162        0.0.0.0:0            LISTENING <-- pivot
{% endhighlight %}

The pivot point was set on the DC. I could now connect xfreerdp to my newly-created listener and have my traffic forwarded to the secondary host:
{% highlight html %}
root@kali:~ # xfreerdp /d:DOM /u:admin /p:password /v:10.10.10.10:49162
connected to 10.10.10.10:49162
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@           WARNING: CERTIFICATE NAME MISMATCH!           @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
The hostname used for this connection (10.10.10.10)
does not match the name given in the certificate:
Common Name (CN):
  DB-20.dom.local
A valid certificate for the wrong name should NOT be trusted!
Certificate details:
  Subject: CN = DB-20.dom.local
  Issuer: CN = DB-20.dom.local
  Thumbprint: 52:da:ec:9e:f6:75:6e:84:12:d2:2f:e9:e3:2c:47:b6:3b:d0:bf:d3
The above X.509 certificate could not be verified, possibly because
you do not have the CA certificate in your certificate store, or the
certificate has expired. Please look at the documentation on how to
create local certificate store for a private CA.
Do you trust the above certificate? (Y/N) Y
{% endhighlight %}

Woo, thanks `netsh`!

### More Info

This is an example of using `netsh interface portproxy` to establish pivoting routes over IPv4, but it's also possible over IPv6, and even IPv4-to-IPv6 (and vice versa). Check out the documentation from Microsoft <a href="https://technet.microsoft.com/en-us/library/cc731068%28v=ws.10%29.aspx" target="_blank">here</a>.

This should work out of the box on Windows 2008 and later. It should also work on Windows 2003, but it may error out if IPv6 isn't supported on the host. More info <a href="https://support.microsoft.com/en-us/kb/555744" target="_blank">here</a>.

#### Share
<ul class="share-buttons">
  <li><a href="https://twitter.com/intent/tweet?source=http%3A%2F%2Fporterhau5.com%2Fblog%2Fnative-port-forwarding-windows%2F&text=Pivoting%20in%20Windows%20Using%20Native%20Port%20Forwarding:%20http%3A%2F%2Fporterhau5.com%2Fblog%2Fnative-port-forwarding-windows%2F&via=porterhau5" target="_blank" title="Tweet"><img alt="Tweet" src="{{ site.urlimg }}flat_web_icon_set/black/Twitter.png"></a></li>
  <li><a href="http://www.reddit.com/submit?url=http%3A%2F%2Fporterhau5.com%2Fblog%2Fnative-port-forwarding-windows%2F&title=Pivoting%20in%20Windows%20Using%20Native%20Port%20Forwarding" target="_blank" title="Submit to Reddit"><img alt="Submit to Reddit" src="{{ site.urlimg }}flat_web_icon_set/black/Reddit.png"></a></li>
  <li><a href="https://getpocket.com/save?url=http%3A%2F%2Fporterhau5.com%2Fblog%2Fnative-port-forwarding-windows%2F&title=Pivoting%20in%20Windows%20Using%20Native%20Port%20Forwarding" target="_blank" title="Add to Pocket"><img alt="Add to Pocket" src="{{ site.urlimg }}flat_web_icon_set/black/Pocket.png"></a></li>
  <li><a href="https://www.facebook.com/sharer/sharer.php?u=http%3A%2F%2Fporterhau5.com%2Fblog%2Fnative-port-forwarding-windows%2F&t=Pivoting%20in%20Windows%20Using%20Native%20Port%20Forwarding" title="Share on Facebook" target="_blank"><img alt="Share on Facebook" src="{{ site.urlimg }}flat_web_icon_set/black/Facebook.png"></a></li>
</ul>
