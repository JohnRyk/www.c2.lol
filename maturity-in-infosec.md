---
description: >-
  I haven't been in information security for a very long time, I actually just
  recently passed the three year mark. But as I've talked with people and grown
  my very long list of people I look up to, I'v
---

# Maturity in Infosec

_Originally Published:  August 11th, 2017_

### Shells are sexy <a id="shells-are-sexy"></a>

I think anyone who starts Red Teaming loves the "badness" of it. You get to be a bad guy, break into companies, and outsmart some very smart people. There's nothing better than getting a shell. Heck, even the shells in a lab environment are fun but man, those live shells. Those are something else.

When I was making the transition from Sysadmin to Information Security, I decided to pursue my OSCP. My first day in the labs, I got my first shell. MS08-067 FTW. It was easy mode:

* Point Metasploit at a Windows 2003 box
* `use exploit/windows/smb/ms08_067_netapi`
* pull trigger
* get shell.

It didn't matter that it was easy, seeing `Meterpreter session 1 opened..` was the coolest thing. I'd hacked something! I got a shell! I was a hacker!

### I break things, I don't fix them <a id="i-break-things-i-don-t-fix-them"></a>

Sometime later I got a job as a Penetration Tester. I was one of the cool kids! I was being paid to hack things!

And man did I love getting shells.

Around this time I also submitted a CFP to BSides Charleston and got accepted! I was going to talk about PowerShell, how to use it, and some of the cool hacking tools that are out there for it.

The talk went well, I got laughs where I wanted them and people were engaged. During the couple of minutes of Q and A after the talk I was asked a question that has since become a strangely defining moment in my career:

> How do you protect against this PowerShell stuff?

I didn't know how to respond. I knew how I shouldn't respond, which is, of course, _"I break things, I don't fix them"_. Which is exactly what I said. I meant to say something like "Defense isn't really a focus of mine right now" and as my mouth opened and the words came out I genuinely thought "You shouldn't be saying this". Ultimately though, I didn't worry about it too much at the time. I knew I misspoke and didn't represent myself the way I wanted to, but it wasn't the end of the world.

Later the video for the talk was posted to YouTube and then Jeffrey Freaking Snover tweeted about the talk. He quoted something I'd said. I was on cloud nine. That dude created PowerShell! He's like, a big technical badass at Microsoft! He watched my video!

And then Matt Graeber tweeted:

> Words that should never be uttered from your mouth - "I break things. I don't fix things."— Matt Graeber \(@mattifestation\) [February 6, 2016](https://twitter.com/mattifestation/status/696053262751911936)

I was crushed. Matt was one of my heroes at the time \(still is for that matter\). He was very kind not to attribute that to me directly, but I knew it had to be about me. I replied to the tweet, owning up to my mistake and he ended up DMing me and we chatted about it a bit. I managed to save some face and make a new friend at the same time.

It was a lesson learned. I should take defense more seriously.

### Good attackers know defense <a id="good-attackers-know-defense"></a>

I regularly think about whats next for me. Where do I want to go in my career, what new skill do I want to learn, whats going to keep me entertained and happy. A lot of that is inspired by the people around me and the people I look up to.

When I think about the security people that inspire me, there's a common thread: No matter what their job is, they know how to defense. [Carlos Perez](https://twitter.com/carlos_perez) is a master at this. For any given technique he talks about, he can tell you what files are dropped and what registry keys are touched. [Dave Kennedy](https://twitter.com/hackingdave) has an entire company dedicated to defense. [Matt Graeber](https://twitter.com/mattifestation) has published the [go to guide for implementing Device Guard](http://www.exploit-monday.com/2016/09/introduction-to-windows-device-guard.html). [Devon Kerr](https://twitter.com/_devonkerr_) and [Daniel Bohannon](https://twitter.com/danielhbohannon/) are both masterful data forensics/incident response guys turned Researchers. They'd also be two of the most formidable red team members around if they decided to go that route.

There are several reasons that mature and competent attackers are mindful of defense. Knowing what protective measures are out there can inform our attacks and lead us to new and more advanced techniques. But I think the main reason that a lot of respected members of the security community are so focused on defense is that they know that all of information security is about defense.

No matter our position, our job in infosec is to make people safer. Whether we're doing this by demonstrating risk, monitoring logs, hunting threats, reviewing source code or any one of the hundreds of other things we can be doing, we are the trusted experts on stopping bad guys from hacking stuff.

We can only be those trusted experts though if we can help fix things. Popping shells is great. I will never get tired of seeing `Meterpreter session opened` or `Agent Connected`, or of being somewhere I'm not supposed to be, but if we can't help fix whats broken, we're of no use and we're wasting a very valuable and powerful position.

To me, pursuing an understanding of both offense and defense is a sign of a mature infosec professional. With that maturity and knowledge comes the opportunity for real impact and that impact can be huge. In my short career as a red teamer, I've helped startups, giant financial companies, hardware manufacturers, and others all become more secure. And I'm not unique here, no matter what you're doing in infosec, your work is making the world a safer place for people. That's absolutely something we should be making the most of.

As attackers it should be a natural part of our process when we look at a new technique, we should be running procmon \(or your tool of choice\) and documenting what happens. Before we run an attack against a customer we should be able to speak in depth as to what our Indicators of Compromise are and we should always keep an eye out for whats new on the horizon for both attacks _and_ defense.

Dave Hull had a [great Twitter thread](https://twitter.com/davehull/status/895300527348748289) that inspired this article. I encourage you to read it. I would love to be at a con and hear overwhelming applause at a detection or mitigation. That's exactly what we _should_ be applauding! Detections and mitigations are awesome, they fix whats broken!

And they let us focus on new and exciting things to break.. and help fix. :\)

