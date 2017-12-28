---
layout: post
title: OSCP Journey
---

My journey for Offensive Security's coveted OSCP started back in May of 2016. At this point in time I was starting to get interested in information security after doing years of system and network administration. I was wanting to push myself to the next level and not only become a better sysadmin but get into a new area of IT. After looking into different security certifications, the OSCP caught my eye. I read the recommended requirements and figured I had enough to satisfy them and I jumped right in purchasing the course and 60 days of lab time. 

I found the course to be very exciting and interesting as most of it was all new to me. A few concepts seemed slightly difficult to grasp at first such as ssh tunneling and 
pivoting. But in reality it was simple to understand after my mind decided to stop trying to overcomplicate things. The buffer overflow section of the courseware is probably one of the strongest and most interesting areas of the courseware, in my opinion. 

After around two weeks I finished the course materials and found myself wondering, "Well now what do I do?". I felt unready to start attacking lab machines and that the courseware had not prepared me sufficiently. OffSec does not 
do much by hand holding whatsoever, they provide some basic fundamentals and it is up to you to learn. The real learning occurs in the labs, by practicing and owning machines. If you find yourself in this same situation after starting OSCP, just know that it is normal. Start with the low hanging fruit and work from there. Also check the writeup for Alpha on the forums, it is an excellent walkthrough by g0tm1lk.

After my 60 days of lab time was up, I decided to renew for another 30 days. At the time I think I had only owned about 25-30 machines and felt nowhere ready to attempt the exam. After another 30 days I was up close to the 40s and decided to schedule my first exam attempt about a month after my lab time ended to give myself time to review notes, anything in the courseware, and a small break. 

First exam attempt I managed 2 roots shells and 2 low privileged shells. I got severly stuck on privilege escalation for hours and came up with nothing finally giving up around 6am after starting at 7am the previous day. I decided to not even submit my report. After the failure I experienced extreme burnout after four months of straight work towards the certification. I winded up taking 3 months off and didn't even touch Kali again until the beginning of 2017.

I finally got motivated again and instead of purchasing more lab time I decided to start working on Vulnhub boxes. I managed to do around ten of them and during this time I also reviewed notes, joined a slack group, and read a couple books and a ton of articles. I scheduled my second attempt at the end of March. 

Second attempt of the exam, I got 3 root shells. Once again I failed. I went on a vacation and came back with a fresh mind. I bought 30 days of lab time and winded up taking down the rest of the boxes in the OSCP lab including Sufferance and Humble. After finishing up, I scheduled my exam for July giving me a month and a half or so of time inbetween. Something new popped up in the slack that I'm apart of called hackthebox. At the time it was still in beta and very limited. It has grown quite a bit now and is an excellent resource. I winded up taking a look at that as they were releasing new boxes weekly. I spent the time before my exam doing lots of hackthebox and it really paid off. I was able to really get my methodology down even more and work on my weak areas. Privilege escalation became somewhat of a strong point. 

I finally felt fully prepared and started the exam at 8am on a Friday. Within the first four hours I was certain I was going to fail again after only getting one shell. After taking a short break to go to the gym and coming back I settled back in and was able to find my way into the other boxes. I got stuck on Windows privilege escalation for hours on one box, thinking it was so locked down and impossible. I decided to start my enumeration over and found I missed something so simple the first time around. 

At 9pm I had four out of five roots. I knew I passed at that point and debated on just calling it a night since the last box I seemed to have absolutely no idea on. I took a short break and decided to look at the last box again. I winded up finding something within the first ten minutes and had a shell in no time. Privilege escalation was also found in no time and I finished with five out of five roots! With the exam report submitted the following day, a couple days later I got the email that took over a year to get. 

![OSCP](/img/oscp.png)

This was certainly the hardest thing I had done thus far in my IT career and the most satisfying. At this point I've realized that OSCP is only the door into this vast field and there is still much to learn. 

### Lessons Learned:

Before starting the OSCP labs I would recommend making sure you're able to satisfy the requirements OffSec posts as well as:

- Get familiar with Kali and some of the tools. 

- Do some vulnhubs, read through walkthroughs to understand methodologies. 

- Check out [HackTheBox](https://www.hackthebox.eu/), start with easy machines and work up. Retired machines have writeups.

When starting the labs:

- Do the course materials all the way through first. Depending on experience you can probably skip some of it. Also document all of your exercises during this time so you don't have to do it later if you want the extra 5 points for the lab report. Coming back to do this later will be a burden. 

 - Do not get root happy and start trying to own boxes as fast as possible without understanding what you are doing.

 - Take really good notes on everything you do when working on a box, even the things that don't work. You will learn a lot from your failures.

- Copy and paste the actual commands you use, as well as any screenshots. This makes it easier to reuse commands later on. 

- This is a hard one, but try to avoid the forums as much as possible. The admins do a decent job at cleaning up spoilers on boxes, but it is still easy to extract information out of posts and I got spoiled a few times. This robs you of a good learning experience! If you get really stuck, message an admin via their help. 

- Make sure you do proper post enumeration after rooting a box. Do not just grab the flag and go. Look for files, search logs, network connections, etc. Some boxes have dependencies!

Exam Tips:

Besides the usual get rest and plenty of sleep the night before here's what I recommend:

- Work on one box only for an hour or two at a time unless you are making progress. If you get stuck, you need to move on.

- Do the buffer overflow machine early while your mind is fresh. You can run scans and whatever else in the background while you're working on it. I made the mistake of waiting until late in my first exam attempt to work on it and I spent way too long making silly mistakes because I was already tired.

- The exam will probably psyche you out a bit. You're probably going to overcomplicate things and make things harder than they really are. When this happens, take a breather and come back. Many times starting enumeration over will provide the path forward. 

- Make sure you take good notes, especially annotating what you have already tried so you don't waste time doing the same things over and over hoping to find something. 

- Have a guide to follow steps on what you should be doing so you have some structure and aren't just doing random things. I made a few cheatsheet type guides. Check them out [here](https://github.com/absolomb/Pentesting). After reviewing notes from my exam failures there were many things I simply did not do that most likely would have led to my success. 

- Take breaks, go for a run, or go to the gym if/when you get stuck. You'd be amazed at how much it can help. 

Lastly, don't give up. Many people fail this exam, keep at it. If you start feeling burned out, take some time away, play some video games, take a vacation, etc. 