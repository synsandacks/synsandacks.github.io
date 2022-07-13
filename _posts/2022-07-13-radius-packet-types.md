---
layout: post
title: Radius Packet Types
subtitle: Sometimes we need to lab it again..
tags: [ise, radius, cisco, pcap]
---

Have you ever been asked a question that you know the answer to, but your mind doesn’t let you recall that information at that moment in time? This recently happened to me while discussing Cisco ISE and in the conversation, I was asked if I know the common RADIUS packet types. Like a deer in the headlights, my thoughts were frozen, and my brain would not let me recall that information. Luckily the other end of the conversation didn’t make me feel bad for this. Instead, they rattled them off and as they did, I could see the baby blue lines of RADIUS packets in Wireshark flash in my mind. Maybe it was the setting of the conversation, or I haven’t had to troubleshoot a problem recent enough that my mind decided it was safe to tuck this information away. When I find myself in situations like this or when I’m trying to learn something new, I like to lab it up and take notes. This post will be my notes on this topic. I hope you enjoy!

# Overview

What is RADIUS? Well the acronym stands for Remote Authentication Dial In User Service. The protocol allows you to authenticate users or machines accessing a service or network through a central management server. In this post our RADIUS server will be a standalone ISE Node running 2.6. 

Some key things to know about RADIUS:

* UDP Based Protocol
    * Port 1812 - Authentication
    * Port 1813 - Accounting
* Open Standard 
* Authentication and Authorization are combined into a single packet.
* Only the password is encrypted in the Access-Request packet. 
* [RFC 2865](https://datatracker.ietf.org/doc/html/rfc2865)

# Radius Packet Types

For RADIUS transactions there are four packet types.

* Access-Request - The NAD sends this request to the authentication server starting the RADIUS session. 
* Access-Accept - The authentication server sends this packet to the NAD to permit access. 
* Access-Reject - The authentication server sends this packet back to the NAD to deny access.
* Access-Challenge - The authentication server needs more information from the client or NAD to process the request. (Not included in these notes.)

Within the RADIUS packets there are quite a few attributes that are passed between the client and the server. The full list can be found [here](https://datatracker.ietf.org/doc/html/rfc2865#section-5.44). For these notes I will highlight three of them.

* Type 1 - User-Name
* Type 2 - User-Password (Encrypted in packet)
* Type 26 - Vendor Specific

To leverage vendor specific attributes (VSA) within ISE you need to have them defined. ISE 2.6 includes the VSA's of 14 vendors by default. Katherine McNamara curated a list for most vendors and made them available for download on her blog [network-node.com](http://www.network-node.com/blog/2018/11/10/vendor-specific-dictionaries-for-ise).

# Lab Setup

So just a high level overview of this lab environment. It's a fresh install of ISE 2.6 running and EVAL VM. I also have eve-ng community edition running a single IOSv router. That router has a single connection to a cloud network allowing interaction with the ISE node hosted outside of eve. 

Since I have limited resources (desktop computer) I opted to leverage local users and groups within ISE for my policy. Typically you would expect to see Active Directory leveraged for this.

* Created Two Identity Groups
    * Admins
    * Lab-NetOps

* Created Two Users
    * labadmin - added to Admins identity group. 
    * labuser - added to Lab-NetOps identity group.

* Created One Device Group
    * Cisco Routers

* Created One Network Access Device (NAD)
    * Added labrouter to Cisco Routers device group.

* Created Two Authorization Profiles
    * Cisco-IOS-Priv-15 - Send a RADIUS access-accept and send down the Cisco VSA *cisco-av-pair* with a value of shell:priv-lvl=15
    * Cisco-IOS-Priv-2 - Send a RADIUS access-accept and send down the Cisco VSA *cisco-av-pair* with a value of shell:priv-lvl=2

* Created One RADIUS Policy - Cisco Router Management
    * Top level condition matches on Device Type = Cisco Routers
        * Authentication Policy - Leveraging "Internal Users" Identity Store
        * Authorization Policy - Three authorization policies
            * Cisco-IOS-Admin - If user is a member of Admins group permit access and set privilege level 15.
            * Cisco-IOS-NetOPs - If user is a member of Lab-NetOps group permit access set privilege level 2.
            * Default - Deny Access

On the IOSv Router I have the following AAA config: 

```
aaa authentication login default group LABRADIUS local
aaa authorization exec default group LABRADIUS local
username admin privilege 15 secret 5 $1$2.Qh$Hbwd8olTH8TZ5e1xhupYG.
!
!
radius server LABISE
 address ipv4 192.168.228.30 auth-port 1812 acct-port 1813
 key radkey
!
!
aaa group server radius LABRADIUS
 server name LABISE
!
!
aaa new-model
aaa session-id common
!
!
```

## Inspecting Packets

So lets take a look at the packets for the the following scenarios. 

* Attempt to login to the labrouter with a bad username / password. 
* Login to labrouter with the labadmin account.
* Login to labrouter with the labuser account. 

### Access-Reject 

When we attempt to login to the labrouter via SSH using the username `baduser` we see two RADIUS packets. One radius access-request sourced from the router to ISE and one access-reject from ISE back to the router.

![baduser-packets](/assets/img/07122022/baduser-packets.png)

Looking deeper into the access-request packet we can see the following:

* RADIUS Packet Code: 1 (Access-Request)
* Attributes Value Pairs
    * Type 1 - User-Name
    * Type 2 - Password

![baduser-access-request](/assets/img/07122022/baduser-access-request.png)

Since ISE doesn't know this user it denied the request with a RADIUS access-reject. If we look deeper into that packet we don't see much beyond RADIUS packet code 3. This tells the NAD to deny this request.

![baduser-access-request](/assets/img/07122022/baduser-access-reject.png)

## Access-Accept - labadmin

When we attempt to login to the labrouter via SSH using the username `labadmin` we see two RADIUS packets. One radius access-request sourced from the router to ISE and one access-accept from ISE back to the router.

![labadmin-packets](/assets/img/07122022/labadmin-packets.png)

The access-request packet looks exactly the same as the previous example with the exception of the user-name and password being different. For brevity we will only examine the access-accept packet sent back from ISE.

Looking at the RADIUS section of the access-accept packet we see the following:

* RADIUS Packet Code: 2 (Access-Accept)
* Attributes Value Pairs
    * Type 1 - User-Name
    * Type 26 - Vendor-Specific
        * Cisco-AVPair and we are passing the following value: SHELL:priv-lvl=15

![labadmin-access-accept](/assets/img/07122022/labadmin-access-accept.png)

By sending down the VSA Cisco-AVPair we are able to tell the router to set the users privilege level to 15. 

![labadmin-priv15](/assets/img/07122022/priv15.png)

## Access-Accept - labuser

When we attempt to login to the labrouter via SSH using the username `labuser` we see two RADIUS packets. One radius access-request sourced from the router to ISE and one access-accept from ISE back to the router.

![labuser-packets](/assets/img/07122022/labuser-packets.png)

The access-request packet looks exactly the same as the previous example with the exception of the user-name and password being different. For brevity we will only examine the access-accept packet sent back from ISE.

Looking at the RADIUS section of the access-accept packet we see the following:

* RADIUS Packet Code: 2 (Access-Accept)
* Attributes Value Pairs
    * Type 1 - User-Name
    * Type 26 - Vendor-Specific
        * Cisco-AVPair and we are passing the following value: SHELL:priv-lvl=2

![labuser-access-accept](/assets/img/07122022/labuser-access-accept.png)

By sending down the VSA Cisco-AVPair we are able to tell the router to set the users privilege level to 15. 

![labuser-priv2](/assets/img/07122022/priv2.png)

## Closing Thoughts

Although we can lock down a users session to specific privilege levels RADIUS isn't the preferred AAA protocol for device administration. Maybe I will follow up with another post about TACACS.

[Download associated PCAP.](/assets/img/07122022/radius-packets.pcap)