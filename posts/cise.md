---
title: Certified iOS Security Engineer
description: Review of the Certified iOS Security Engineer certification
date: 2023-11-29T19:30:20.363191+01:00
toc: false
tags:
    - ios
    - cert
refs:
draft: false
---

# Background and preparation
A couple of days ago I have obtained [CISE](https://8ksec.io/cise/) (Certified iOS Security Engineer) from [8ksec](https://8ksec.io). As I am a penetration tester and the person who is interested in iOS and macOS security, this was really nice exam to test my knowledge about it. I did not take any specific preparation besides the work I do daily which is pen testing iOS applications and learning about its internals.

The exam will prove that you have the knowledge about the iOS applications, filesystem and different attack vectors. You will have to utilise the static and dynamic analysis using tools such as IDA Pro, Hopper, radare2, Frida, etc. There are no any specific prerequisites for the exam, but you will be in much better position if you spend some time penetration testing or generally doing any kind of iOS based research. This certificate is an intermediate level so you can approximately know what to expect.

# The exam
For the exam, you get 24 hours to finish it. It consists of multiple iOS applications that you have to attack, but this may not always be the case because from the description of the exam you can also get some services to attack, such as XPC services.

For the exam, you get applications/services installed on the Corellium labs and the access to it and the exam objectives are provided to you at the time of the exam start. This was the first time that I have used Corellium labs so it was pretty nice to get my hands on it.

After you are logged in, you will start with the usual reconnaissance that starts from the static analysis, such as utilizing Hopper, examining the app structure in the case if you are dealing with application instead of service, dumping the strings, etc.

Sometimes there may not be anything interesting during the static analysis, so you will have to start with dynamic analysis. For the dynamic analysis, the Frida is the tool you will use mostly. I had to bypass anti-debug and anti-Frida in order to do dynamic analysis properly. After you have bypassed these methods, you will need to find other vulnerabilities and the goal is to find them as much as you can.

After you are done with the exam, you write the report with all your findings and a couple of days later you will be notified about the results.

# Conclusion
Overall, the exam was pretty nice and anyone with some level of experience with iOS should be able to pass the exam. I would suggest this exam to anyone interested in iOS as this will be some kind of confirmation for you knowledge.

# Resources
Here are a couple of resources where you can learn more about the things that will be required in the exam.

* https://frida.re/docs/javascript-api/
* https://8ksec.io/advanced-frida-usage-part-1-ios-encryption-libraries-8ksec-blogs/
* https://8ksec.io/advanced-frida-usage-part-2-analyzing-signal-and-telegram-messages-on-ios/
* https://8ksec.io/advanced-frida-usage-part-3-inspecting-ios-xpc-calls/
* https://8ksec.io/advanced-frida-usage-part-4-sniffing-location-data-from-locationd-in-ios/
* https://8ksec.io/arm64-reversing-and-exploitation-part-1-arm-instruction-set-simple-heap-overflow/
* https://8ksec.io/ios-deeplink-attacks-part-1-introduction-8ksec-blogs/
* https://8ksec.io/ios-deep-link-attacks-part-2-exploitation-8ksec-blogs/
* https://codeshare.frida.re/
* [https://dhiyaneshgeek.github.io/mobile/security/2021/12/25/hopper-disassembler/](https://dhiyaneshgeek.github.io/mobile/security/2021/12/25/hopper-disassembler/)
* https://payatu.com/blog/runtime-manipulation-with-lldb/
* https://philkeeble.com/ios/reverse-engineering/iOS-Anti-Anti-Hooking/
* And of course my own blog that contains a couple of useful iOS related blog posts
