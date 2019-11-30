---
layout: post
title: "Public response to Everspin's warning letter"
description: I worked on Android app research with my team at Theori Korea this year from October 7 - October 25. 
modified: 2019-11-29
category:
  - Legal
  - Research
  - English
featured: true
imagefeature: false
---

Hi, I'm Brian Pak, CEO and Co-founder at Theori.

I worked on Android app research with my team at Theori Korea this year from October 7 - October 25. The goal of the research was to analyze and evaluate the security of various mobile app obfuscation solutions that are globally used by many companies, including our customers. I gave a talk at Power of Community (POC), an international cybersecurity conference, on November 7th, 2019, to present our work. As I emphasized multiple times in the talk, the purpose of this research was to technically analyze various existing mobile app obfuscation solutions, and to compare features and performance in a more objective manner. We hoped to help companies come up with more accurate and realistic threat models.

As we only considered evaluating based on technology and not on marketing materials, not everyone received good grades. After the talk, a few Korean companies have contacted me directly or indirectly to tell us the result of the research was a helpful critique for them and promised to improve their solution.

However, not every company was satisfied with the result. A company even sent a warning letter to prosecute criminal charges by hiring a fairly large law firm and claiming that our findings and presentation were seriously defamatory with false information. The company requested that we: 1) delete the slides that are publicly available on GitHub, 2) submit an apology letter for dissemination of false facts, and 3) submit a pledge to not reveal any more information about the company ever again.

It is possible for us to mask or delete specific company and/or solution names on request, but it is not acceptable for us to adhere to the request for an apology letter because we do not believe we released any false information. The research was for the common good and it is very disappointing that a larger corporation is now legally pressuring a small tech startup to shoot down a negative result even when it's factual.

The threatening company, Everspin, has claimed that:
- The false information about Everspin's mobile app obfuscation solution, Eversafe, is severely undermining the company's reputation.
- Eversafe is marked as "easy" (to analyze and bypass), and this is ill-intended / malicious.

In particular, Everspin claims the research was malicious for the following reasons.

#### Release of false information 1

##### Everspin's claim

The screenshot that shows server response error (503) is from a 2017 blog post (bpak.org). By using the same image, the researcher claimed that the dynamic module loading feature is broken from 2017 and that the issue persists until this date.

##### My response

As mentioned in your warning letter, I know of the incident in 2017 where the dynamic module was not loaded due to the server component failure. However, in our recent and separate experiment in 2019, the server indeed responded with a 503 error, which resulted in the dynamic module not being loaded. At the time of the experiment, we used a rooted device (with Magisk) but did not use any rooting detection bypass scripts.

Furthermore, Google regulates and disallows any app that downloads and loads code dynamically from a non-Google server, such as Everspin's 3rd party server. It is the violation of Play Store policy because it can introduce a huge security risk for Android users. During my talk, I even mentioned this as a potential and reasonable explanation for the server misbehavior.

The screenshot that was used in the slide deck is from my 2017 blog post. The researcher who prepared the slide referenced the same image because our latest experiment showed the same behavior as 2 years ago. I never claimed that the screenshot was from the 2019 experiment, but merely commented on the fact that not much has improved since 2017.

Everspin's claim that our research stated false results about their server status, by using 2017 data as if it had happened in 2019, is untrue. The dynamic module did not run when we tested it in October 2019 as the server did not respond successfully. After I received Everspin's warning notice, our research team simulated the situation where the server was returning a 503 error code, by replacing the response packet on the fly using MITM (Man-in-the-middle) proxy technique. We confirmed that the dynamic module did not load and a rooted device was not detected.

Below demo shows the simulation process and its result.

<p markdown="0">
<iframe width="560" height="315" src="https://www.youtube.com/embed/7GmUjARNcY0" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</p>

As we can see in the demo, if the server returns an error code 503, the dynamic module does not work correctly, whereas if the server returns the success code 200, the dynamic module gets loaded and is able to detect the rooted phone. Note that we did not change anything on the phone during the test -- only the response code on the network. This further shows that the behavior we observed back in October indicates the server was returning an error code as we described in the research.

Note: As Everspin explained, the static module is used when the dynamic module doesn't load, but the static module did not even detect our obviously rooted device.

#### Release of false information 2

##### <Everspin's claim>

Eversafe does not get bypassed as easily as described in the talk by simply preventing the initialization API invocation. The research falsely claimed that the protection was easily bypassed by just showing a very simple example.

##### <My response>

Everspin arbitrarily defines "completely hacked" and claims that our research is false because our definitions do not match with theirs. The two points I made during the talk were the following:
1) It was trivial to locate the Eversafe initialization code since the invocation was in the app's main activity constructor, and if the invocation is prevented, the security module stopped working.
2) Because of this, the dynamic module did not get loaded and the rooting detection + integrity check did not work properly either. This was shown with an example of us injecting arbitrary code (a toast message), repackaging the app, and successfully executing it without being blocked.

Everspin defines whether or not you can "log in" as a measure of "completely hacked". However, not only do the above facts already indicate that the app's security and integrity have been compromised, but the ability to log in, when arbitrary code can be run within the repackaged app, seemed obvious. As such, I didn't cover it any deeper.

However, the bypass that meets the conditions of "completely hacked" as defined by Everspin was already successful in the research. In fact, we were able to demonstrate that a banking app that is protected by Everspin can be trivially modified to disable the Eversafe security module, log in as a user, and transfer funds to another bank. We did not disclose this in detail in our presentation because it wouldn't be fair to do so given that we did not cover the topic as deeply for examples from other vendors.

The demo below shows the process of modifying the latest version of a banking app that is protected by Eversafe. You can see that it is possible to disable the security module (rooting + integrity not validated), log in, and transfer funds to another account. For demonstration purposes, we simply showed a toast message containing our company's name. But it is possible to modify the app to change the receiving account number arbitrarily and transfer the funds to an unintended account.

<p markdown="0">
<iframe width="560" height="315" src="https://www.youtube.com/embed/g0pPnW_gBhg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</p>

Additionally, Everspin has cited a memorandum of understanding (MOU) signed by Theori US and Raonsecure back in January 2017. Everspin suspected that the research was indirectly promoting third-party security solutions that are in a special relationship with Theori by classifying them as a "hard" target, while degrading Everspin's evaluation.

This is a fundamental misunderstanding of the relationship between the two companies. Theori signed an MOU with Raonsecure in early 2017 hoping to actively cooperate with Raon White Hat Center, which focuses on their security consulting business only. Since then, the companies have maintained an ostensible partnership without a single exchange and ended it a year later in 2018. At the time of this research, our company had no interest in Raonsecure despite what Everspin may believe. There was no reason for us to promote any specific company in our research.

We originally were not planning to include Everspin in the presentation as one of the listed vendors, due to the lack of any significant improvements/changes compared to 2 years ago. However, their product was being used by Korean and non-Korean financial institute apps so we decided it would be unfair to exclude them. The roster was filled mainly based on social impact in the public domain, and Everspin's belief that we pursued this complex and cumbersome research only to degrade their reputation is simply incorrect.

As you may know, POC is a conference led by hackers and security professionals that promotes the latest security technologies by finding and sharing vulnerabilities in existing solutions to help improve them. It is understandable that you felt uncomfortable about the result which states your product was relatively insecure. However, I deeply regret your response, which is not to use the research work as an opportunity to make a better security solution, but to contract a large law firm to distort facts, threaten to file criminal charges, and scare away security researchers to cover up negative assessments of your company.

In a warning letter, Everspin said 'we will meet at the investigation agency' unless I submit a sincere apology letter by November 29, 2019. Now, I am posting a public response to Everspin's warning letter, and I wish to hear from many people whether or not they think the research is stating false information and whether or not the fact that we classified Eversafe as an "easy" [to bypass] product is intentionally malicious.

As a security researcher, I hope this research, the presentations, and the public discussion will improve existing technologies and enable users to use mobile apps in a more secure environment.

