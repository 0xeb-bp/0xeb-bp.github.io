---
layout: post
title:  "Practical Guide to Passing Kerberos Tickets From Linux"
date:   2019-11-21 09:00:00 -0500
categories: [blog]
tags: [security, linux, windows, kerberos]
---

This goal of this post is to be a practical guide to passing Kerberos tickets from a Linux host. In general, penetration testers are very familiar with using Mimikatz to obtain cleartext passwords or NT hashes and utilize them for lateral movement. At times we may find ourselves in a situation where we have local admin access to a host, but are unable to obtain either a cleartext password or NT hash of a target user. Fear not, in many cases we can simply pass a Kerberos ticket in place of passing a hash.

This post is meant to be a practical guide. For a deeper understanding of the technical details and theory see the resources at the end of the post.

## Tools

To get started we will first need to setup some tools. All have information on how to setup on their GitHub page.

**Impacket**

    https://github.com/SecureAuthCorp/impacket

**pypykatz**

    https://github.com/skelsec/pypykatz

**Kerberos Client**

    RPM based: yum install krb5-workstation
    Debian based: apt install krb5-user

**procdump**

    https://docs.microsoft.com/en-us/sysinternals/downloads/procdump

**autoProc.py (not required, but useful)**

    wget https://gist.githubusercontent.com/knavesec/0bf192d600ee15f214560ad6280df556/raw/36ff756346ebfc7f9721af8c18dff7d2aaf005ce/autoProc.py

## Lab Environment

This guide will use a simple Windows lab with two hosts:

    dc01.winlab.com (domain controller)
    client01.winlab.com (generic server

And two domain accounts:

    Administrator (domain admin)
    User1 (local admin to client01)

## Passing the Ticket

By some prior means we have compromised the account user1, which has local admin access to client01.winlab.com.

![image 1](/assets/images/practical-guide-pass-the-ticket/1.png)

A standard technique from this position would be to dump passwords and NT hashes with Mimikatz. Instead, we will use a slightly different technique of dumping the memory of the `lsass.exe` process with `procdump64.exe` from Sysinternals. This has the advantage of avoiding antivirus without needing a modified version of Mimikatz. 

This can be done by uploading `procdump64.exe` to the target host: 

![image 2](/assets/images/practical-guide-pass-the-ticket/2.png)

And then run:

    procdump64.exe -accepteula -ma lsass.exe output-file

![image 3](/assets/images/practical-guide-pass-the-ticket/3.png)

Alternatively we can use `autoProc.py` which automates all of this as well as cleans up the evidence (if using this method make sure you have placed `procdump64.exe` in `/opt/procdump/`. I also prefer to comment out line 107):

    python3 autoProc.py domain/user@target

![image 4](/assets/images/practical-guide-pass-the-ticket/4.png)

We now have the `lsass.dmp` on our attacking host. Next we dump the Kerberos tickets: 

    pypykatz lsa -k /kerberos/output/dir minidump lsass.dmp

![image 5](/assets/images/practical-guide-pass-the-ticket/5.png)

And view the available tickets:

![image 6](/assets/images/practical-guide-pass-the-ticket/6.png)

Ideally, we want a krbtgt ticket. A krbtgt ticket allows us to access any service that the account has privileges to. Otherwise we are limited to the specific service of the TGS ticket. In this case we have a krbtgt ticket for the Administrator account! 

The next step is to convert the ticket from `.kirbi` to `.ccache` so that we can use it on our Linux host:

    kirbi2ccache input.kirbi output.ccache

![image 7](/assets/images/practical-guide-pass-the-ticket/7.png)

Now that the ticket file is in the correct format, we specify the location of the `.ccache` file by setting the `KRB5CCNAME` environment variable and use `klist` to verify everything looks correct:

    export KRB5CCNAME=/path/to/.ccache
    klist

![image 8](/assets/images/practical-guide-pass-the-ticket/8.png)

We must specify the target host by the fully qualified domain name. We can either add the host to our `/etc/hosts` file or point to the DNS server of the Windows environment.
Finally, we are ready to use the ticket to gain access to the domain controller: 

    wmiexec.py -no-pass -k -dc-ip w.x.y.z domain/user@fqdn

![image 9](/assets/images/practical-guide-pass-the-ticket/9.png)

Excellent! We were able to elevate to domain admin by using pass the ticket! Be aware that Kerberos tickets have a set lifetime. Make full use of the ticket before it expires! 

## Conclusion

Passing the ticket can be a very effective technique when you do not have access to an NT hash or password. Blue teams are increasingly aware of passing the hash. In response they are placing high value accounts in the Protected Users group or taking other defensive measures. As such, passing the ticket is becoming more and more relevant. 

## Resources

[https://www.tarlogic.com/en/blog/how-kerberos-works/][r1]

[https://www.harmj0y.net/blog/tag/kerberos/][r2]

**Thanks to the following for providing tools or knowledge:**

<a href="https://github.com/SecureAuthCorp/impacket"><svg class="svg-icon grey"><use xlink:href="{{ '/assets/minima-social-icons.svg#github' | relative_url }}"></use></svg><span class="username">Impacket</span></a>

<a href="https://twitter.com/gentilkiwi"><svg class="svg-icon grey"><use xlink:href="{{ '/assets/minima-social-icons.svg#twitter' | relative_url }}"></use></svg><span class="username">gentilkiwi</span></a>

<a href="https://twitter.com/harmj0y"><svg class="svg-icon grey"><use xlink:href="{{ '/assets/minima-social-icons.svg#twitter' | relative_url }}"></use></svg><span class="username">harmj0y</span></a>

<a href="https://twitter.com/SkelSec"><svg class="svg-icon grey"><use xlink:href="{{ '/assets/minima-social-icons.svg#twitter' | relative_url }}"></use></svg><span class="username">SkelSec</span></a>

<a href="https://twitter.com/knavesec"><svg class="svg-icon grey"><use xlink:href="{{ '/assets/minima-social-icons.svg#twitter' | relative_url }}"></use></svg><span class="username">knavesec</span></a>

[r1]: https://www.tarlogic.com/en/blog/how-kerberos-works/
[r2]: https://www.harmj0y.net/blog/tag/kerberos/
[harmj0y-twitter]: https://twitter.com/harmj0y
[ben-delp-twitter]: https://twitter.com/gentilkiwi 
[skelsec-twitter]: https://twitter.com/SkelSec
[impacket-git]: https://github.com/SecureAuthCorp/impacket
[knavesec-twitter]: https://twitter.com/knavesec
