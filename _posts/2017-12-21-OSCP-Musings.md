---
layout: post
title: OSCP Musings
---

Originally I had a long post about my trials and tribulations pursuing my OSCP. Rather than bore those who are reading with the details I'm just going to give you some tips and things that I think would have been helpful when I started. Because let's face it, OSCP is pretty tough unless you already have a ton of experience or are just a straight up wizard. 


#### Pre-Lab Time:

- Get familiar with Kali and some of the tools. 

- Do some vulnhubs, read through walkthroughs to understand methodologies. 

- Check out [HackTheBox](https://www.hackthebox.eu/), start with easy machines and work up. Retired machines have writeups.

#### During Lab Time:

- Do the course materials all the way through first. Also document all of your exercises during this time so you don't have to do it later if you want the extra 5 points for the lab report. Coming back to do this later will be a burden. 

 - Do not get root happy and start trying to own boxes as fast as possible without understanding what you are doing.

 - Take really good notes on everything you do when working on a box, even the things that don't work. You will learn a lot from your failures. If your notes are sloppy then you'll come to find yourself confused on what you were doing.

- Copy and paste the actual commands you use, as well as any screenshots. This makes it easier to reuse commands later on. 

- This is a hard one, but try to avoid the forums as much as possible. The admins do a decent job at cleaning up spoilers on boxes, but it is still easy to extract information out of posts and I got spoiled a few times. This robs you of a good learning experience! If you get really stuck, message an admin via their help. 

- Make sure you do proper post enumeration after rooting a box. Do not just grab the flag and go. Look for files, search logs, network connections, etc. Some boxes have dependencies!

#### Exam:

Besides the usual get rest and plenty of sleep the night before here's what I recommend:

- Work on one box only for an hour or two at a time unless you are making progress. If you get stuck, you need to move on. Time will absolutely fly by!

- Do the buffer overflow machine early while your mind is fresh. You can run scans and whatever else in the background while you're working on it. If your methodology is solid you should be able to knock this out quickly. I made the mistake of waiting until late in my first exam attempt to work on it and I spent way too long making silly mistakes because I was already tired.

- The exam will probably psyche you out a bit. You're probably going to overcomplicate things and make things harder than they really are. When this happens, take a breather and come back. Many times starting enumeration over will provide the path forward. 

- Make sure you take good notes, especially annotating what you have already tried so you don't waste time doing the same things over and over hoping to find something. 

- Have a guide to follow steps on what you should be doing so you have some structure and aren't just doing random things. I made a few cheatsheet type guides. Check them out [here](https://github.com/absolomb/Pentesting). After reviewing notes from my exam failures there were many things I simply did not do that most likely would have led to my success. 

- Take breaks, go for a run, or go to the gym if/when you get stuck. You'd be amazed at how much it can help. 

Lastly, don't give up. Many people fail this exam (I did), keep at it. If you start feeling burned out, take some time away, play some video games, take a vacation, etc. 
In the end after all the hard work, it is so rewarding to finally get the passing email from OffSec.

![OSCP](/img/oscp.png)

OSCP was certainly the hardest thing I had done thus far in my IT career and the most satisfying. However, I've realized that OSCP is only the door into this vast field and there is still much to learn. 