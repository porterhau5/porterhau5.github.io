---
layout: page
title:  "Introducing Sleat: Security Logon Event Analysis Tools"
teaser: "Sleat is a toolkit for weaponizing Windows logs, providing a suite of scripts for collecting, parsing, and analyzing Logon Events. Sleat can perform scope validation, identify exploitation targets for pivoting attacks, visualize network logons, and more."
tags:
    - windows
    - pen-testing
    - sleat
header: no
image:
    title: photo-sleet.jpeg
    thumb: photo-sleet-thumb.jpeg
author: porterhau5
---
## Quick Overview of Sleat
Sleat is a collection of scripts for collecting, parsing, and analyzing logon events from Windows Security logs.

These scripts can be used for:

- Identifying workstations belonging to privileged users
- Identifying workstations/accounts connecting from the CDE
- Identifying workstations/accounts connecting from an IP address that wasn't included (or conveniently forgotten!) in the scoping documents
- Visualizing the relationship of logon events across the environment using Graphviz

## Background
Before I get into Sleat, I'd like to provide context on why I saw a need to create the toolkit. Doing so requires me to explain a little about PCI penetration testing and some common vocabulary used by those in the space.

In the PCI world, there is this notion of a "Cardholder Data Environment", or a **CDE** -- essentially a high-security environment where credit cards are transmitted, processed, or stored. Typical success criteria for this kind of test is to determine if an attacker originating from outside the CDE can find their way in.

The latest iteration of the PCI-DSS, a standard that governs network and configuration baselines for these types of networks, mandates that the CDE be completely segmented from the rest of the corporate network. This means that using a single Active Directory domain to preside over both the corporate network and the CDE *is prohibited*. If remote access by **privileged users** is needed, then it must be done in a controlled, secure manner with multi-factor authentication and non-corporate domain credentials.

However, I often find that a properly configured, PCI-compliant environment is rare. The majority of my engagements show some degree of connectivity between the two scopes, providing a vector for an attacker to get into the CDE. The key to exploiting this vector is being able to systematically identify it.

This is where Sleat comes in -- a toolkit for finding privileged users, validating scope, and visualizing a network of users, all via Windows logs.

By default, Windows will store events pertaining to Logons in the Security log (Security.evtx). These events include different types of logons, such as interactive logons when a user logs in directly at the console, or RemoteInteractive logons when a user logs in via Remote Desktop or Terminal Services. On Windows systems later than 2003/XP, these events get stored under Event ID 4624.

## Sleat's Scripts
Sleat's collection scripts query for these types of events (Event ID 4624) from the target host, parsing for relevant fields about the authenticating user, such as: IP address, Domain name, Username, and Workstation name. The results are stored in a text file.

Sleat's analysis scripts correlate these parsed events with user-provided data points, such as a list of networks known to be in the CDE, or a list of privileged users who are known to access the CDE.

The end result is a list of attack vectors in the form of usernames or IP addresses. These are the targets a penetration tester could use for abusing privileged access and pivoting to the CDE.

Sleat's parse script is for parsing raw EVTX files directly. For example, if a Security.evtx file has been exported from the target host, then the parse script can be used to extract the relevant fields before passing it to the analysis scripts.

## More Info
To get into the weeds and see how Sleat works, along with example usage and syntax, check out the [Sleat project page](/projects/sleat/). The source code for Sleat is hosted on GitHub <a href="https://github.com/porterhau5/sleat" target="_blank">here</a>.
