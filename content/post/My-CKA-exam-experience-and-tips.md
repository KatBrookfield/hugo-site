---
title: "My CKA Exam Experience and Tips"
date: 2021-12-06T09:32:35Z
archives: "2021"
tags: []
author: "Kat Brookfield"
---

Last week I passed the **Certified Kubernetes Administrator (CKA)** exam and in today's post, I'd like to share my experience and some tips for preparing and taking for the exam.

I have taken a fair share of exams in my life but I have to say this was by far the best exam I've ever taken. I'd even go as far as saying it was... **fun**.

The one thing I did not enjoy was the booking and check-in process. I planned to take the exam from home and the dates available were very limited (it may have been just my timezone, but there were days with one slot only at 1AM??) and the system was veeeeery slow.

On the day of the exam, I logged in 15 mins before the scheduled start (when the button becomes available). I am used to taking exams at home so my desk was clean and I had all documents on hand. My exam pre-checks all passed and I was using Chrome as my browser, with enabled extensions, however, I was told by the proctor that he could not see my screen (even though I could see it shared and his view). After few restarts of the browser I was asked to download Vivaldi, so had to get that and configure it before we could proceed. The responses on chat were really slow too, I assume they were busy with other people checking in. My exam was started 20 mins past the scheduled start, so the whole process took 35 mins. Let me just say I am a *very* nervous exam taker and this delay did not help.

From that point on, things got much better. The exam environment was **great**, very responsive and clear. I am using 13" screen and could see everything I needed without having to resize anything. You get your instructions on the left side and your terminal on the right side. You also get access to things like Notepad but I have only used that once.

CKA is a purely practical exam. I personally find this format much better than theoretical exams. You are also given access to the official Documentation, as you would in real life. In my opinion, this truly tests your *understanding* of concepts, rather than your *ability to memorise* stuff.

You are given 2 hours to complete 17 tasks, which I found was fair amount to complete the tasks and save enough time for a review. The time remaining is shown as a shrinking, colour-changing bar on the top left corner of your screen. This is fine, but I would presonally prefer to see a number.

There are multiple clusters available in the environment. For every task you are told which context to use and also given a *switch context* command which you can copy and paste into your terminal. It is very important you do your tasks in the correct context, so make sure to use the command. (ProTip: Just make sure you read what you are copying first. I have accidentally copied an *exit* command and kicked myself out of my terminal, oops. Luckily it automatically reconnected after few seconds.)

That pretty much sums up my experience, so here are few exam tips:
## 1. Familiarise yourself with terminal and vi
This one is mainly aimed at people coming from Windows background. You have to be comfortable doing basic file operations, like reading and updating files. You can have the best Kubernetes knowledge in the world, but if you won't be able to write your answer to a file, you will not complete a task. Also, get used to creating and updating files using *vi*. Again, you don't have to know everything about it, just know how to do basic operations, such as:
1. vi file.yaml (to create a file if not existing, or update a file if it exists)
2. Press I to enter Insert mode
3. Update file
4. Press ESC to exit Insert mode
5. Exit the file using:
a) :wq! to save changes and exit
b) :q! to discard changes and exit
c) :q to exit if you haven't made any changes

## 2. Documentation is your friend
As mentioned before, you can open an extra tab and access documentation on https://kubernetes.io/. I've read some people like to bookmark pages in preparation for the exam but I did not feel like I needed to do that, the search function is really good. Anything you learn, go to the documentation and find it. This will help you to find what you need during the exam quickly because you will know exactly what you are looking for. Just be sure to only open docs, you **cannot** access other sublinks, such as *Discussions*. You will be told which sites you are allowed to access before the exam starts and it will be up to you to make sure you don't click on anything that would redirect you to a different location.

While I was studying, I would define a keyword for everything I was doing, look for that keyword in the Documentation and go through available pages. The best information is not always the first hit (looking at you, Persistent Volumes).

## 3. Don't try to memorise YAML manifests
This one is related to the previous point but is important to stress out. There is no point in trying to memorise a YAML definition. You know by now how sensitive it is and it is very likely you'll make a mistake and won't be able to create your object. Also, copying a manifest out of documentation and updating it as needed is much faster than trying to write it from scratch. 

You can also use imperative commands to create simple definitions and pipe them out to a file. For example, if you need a simple pod defintion, you can run *kubectl run nginx --image=nginx --dry-run -o yaml > nginx-pod.yaml*

## 4. Read the instructions carefully
The instructions are very clear, they tell you what objects to create, what names to use, which namespace to place them in, follow them carefully. I'd recommend to copy the names just to make sure you don't lose points just because you misspelled something.

Also, read the instructions until the end, and pay attention. For example, they may ask you to *record the action*. It's very easy to miss details like this but it will cost you points.

## 5. Don't go crazy with aliases
Setting up aliases can save you some time but I genuinelly don't think you need to have many of them. Personally, I have only set an alias for kubectl because I tend to misspell it.

You can do so by running *alias k="kubectl"*

There are some other useful things like autocomplete mentioned in the **kubectl Cheat Sheet**, you may find those helpful. While we are on the subject, if you are bookmarking any page, bookmark this https://kubernetes.io/docs/reference/kubectl/cheatsheet/. It's the best place to look when you are trying to find kubectl command syntax quickly.

## 6. Verify your answers
Try to leave yourself enough time at the end of the exam to go through your answers. There is a possibily to flag a question, or mark it as *satisfied with my answer*, but I tend to review all questions at the end. Make sure you used the correct name, placed objects in correct namespace, used correct context, or that you have actually written your answer to a correct file. This will help you prevent silly mistakes that would cost you points.

## 7. Practice, practice, practice!
This one is pretty obvious given it's a *practical* exam but it's worth repeating. All tasks in the exam are doable but you need to know what you are doing, where to look for information and how to put it together, in order to be able to finish the exam in time.

I have spent a lot of time preparing for the exam, coming up with scenarios, and repeating them over and over again.

In terms of training, I have purchased the official  **Kubernetes Fundamentals (LFS258)** training in a bundle with the exam. I have personally struggled with the format of this training a bit, as it was just reading pages of text, which I find very hard to concentrate on. The labs however were great, especially for things like setting up your own clusters on GCP. This is handy for tasks like upgrading your clusters. (ProTip: I recomend to start with an older version and do full upgrades (masters and workers) by one minor version at a time. This will enable you to do the task over and over to train your muscle memory).

Another training course that I would recommend is the **ACloudGuru's Certified Kubernetes Administrator (CKA) by William Boyd**. This course was absolutely fantastic and Practice Exams at the end were so well prepared and gave you the ability to practice tasks in a pre-set environment. 

That's it. I hope you found this information useful and I wish you good luck with your exam!

