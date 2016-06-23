---
layout: page
title:  "Introducing Sleat: Security Logon Event Analysis Tools"
teaser: "Sleat is a toolkit for weaponizing Windows logs, providing a suite of scripts for collecting, parsing, and analyzing Logon Events to perform scope validation, identify exploitation targets for pivoting attacks, visualizing network logons, and more."
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
### Quick Overview of Sleat
Sleat is a collection of scripts for collecting, parsing, and analyzing logon events from Windows Security logs.

These scripts can be used for:

- Identifying workstations belonging to privileged users
- Identifying workstations/accounts connecting from the CDE
- Identifying workstations/accounts connecting from an IP address that wasn't included (or conveniently forgotten!) in the scoping documents
- Visualizing the relationship of logon events across the environment using Graphviz

## Background
Before I get into Sleat, I'd like to provide context on why I saw a need to create the toolkit. Doing so requires me to explain a little about PCI penetration testing and some common vocabulary used by those in the space.

In the PCI world, there is this notion of a "Cardholder Data Environment", or a **CDE** -- essentially a high-security environment where credit cards are transmitted, processed, or stored. The success criteria of this kind of test is to determine if an attacker, originating from outside the CDE, can find their way in.

The latest iteration of the PCI-DSS, a standard that governs network and configuration baselines for these types of networks, mandates that the CDE be completely segmented from the rest of the corporate network. This means that using a single Active Directory domain to preside over both the corporate network and the CDE *is prohibited*. If remote access by **privileged users** is needed, then it must be done in a controlled, secure manner with multi-factor authentication and non-corporate domain credentials.

This is where Sleat comes in -- a toolkit for finding privileged users, validating scope, and visualizing a network of users, all via Windows logs.

Read more about it on the [Sleat project page](/projects/sleat/).
