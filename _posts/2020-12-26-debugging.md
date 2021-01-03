---
layout: post
title:  "[Letters to my 20-yr old self] Debugging"
date:   2020-12-15 09:22:24 -0800
categories: letters_to_myself
---

Debugging are one of those fundamental skills that can make your day to day life very pleasant. I think it can really tip the scales from a frustrating day to a day filled with learning. You will most likely pick up debugging skills over the course of your technical career. But what if you could compress some of these learnings and be ready on day 1? This post assumes that you will be debugging technical problems, user related bugs as well as general software bugs.

There are three dimensions to how you debug a problem. First, there is the 1000ft view of the problem. Second, the actual skills and processes that you use to debug the issue and third is communication.

# The 1000ft view

Any technical task that you attempt will most likely never be a smooth a->b path. You'll likely encounter something that did not work as expected, or be asked to fix something that is broken from the get go. Either way, at its very essence it is unexpected behavior. The very first thing that you want to think about is the problem that you are trying to solve. Some questions you should be asking yourself are:

* Why are you trying to solve this issue? Is this expected? 
* Has this been solved already? 

Something that I have seen more than once is an attempt to solve the wrong problem. 

Sometimes given the behavior of the system, it is technically impossible to solve this issue. You might end up in this situation because you are new to the technical stack and probably don't have sufficient context to understand this when you start working on the problem. It is generally useful to also figure out if someone has attempted to solve this problem. 9/10 times the answer is yes. Someone has attempted to solve a similar problem which helps you get an idea of what the potential solution would look like.

Are you the right person to solve this issue? The previous question segues into this nicely. When you start understanding the context of the problem you are trying to solve, you will start to realize how familiar with the general technical knowledge required for the solution. For example, if you are a model builder but are seeing a CUDA error, chances are that you will require an inordinate amount of time to solve this problem given the lack of technical knowledge. At this point you need to do two things - figure out the priority of the task and then decide if you have sufficient time to learn and solve the problem yourself given some external guidance. I am a big believer of learning things end to end and not offloading tasks to other people. These are the opportunities to really grow technically which means sacrificing short term gains where you probably don't have a fancy weekly update but have a long term gain of understanding and fixing an issue by being largely independent. A caveat to this is of course if the issue is in a part of the stack that is very removed from what you to on a daily basis. In that case going deep wouldn't really add a ton of value to you. You need to analyze the takeaways when deciding on a course of action.

# The 1ft view or the skills needed to actually figure out the issue

If you have somehow made it to this point it means that you are solving a problem on your own. The goal at this point is first have an understanding of what the expected behavior should be. What is the code expected to do in the ideal case? What is it doing now? Identify the delta. The delta will then ideally lead you to come up with a set of hypotheses about what could be going wrong. In a large system, often times you might just have to start with the most likely hypothesis and eliminate possibilities.

One of the fundamental mistakes I've seen people make is not understanding the call flow. And this call flow can mean different things depending on what you are debugging. Essentially think about it like information flow and functions executing on that information. It is critical that you read code and understand this flow as much as possible. Without this, you will not be able to do the following - identify what you know and more importantly what you don't.

It is often the case that the issue lies in area of things that you don't know. If you misidentify this area then debugging becomes an issue because you expect execution and information flow to be working correctly. You've developed a blind spot. Assumptions lead us to spend days on an issue because we haven't analyzed our knowledge areas causing big blind spots to develop.

In terms of actual processes, the first step should be to reproduce the error. When working on lots of different projects, you are often going to be debugging issues in between project work. To help you focus your maximum efforts on identifying the problem, you should have a setup where you can reproduce issues very quickly. For example, a folder with the name bugs, followed by a skeleton template for quickly placing code inside a main function perhaps and finally naming this new file with the bug id. 9/10 times you should ideally ask the user for a repro example or work with them for figuring out the smallest snippet of code that you can run locally to be able to see an issue. The time sinks in this area to be wary of are:

* if the user provides partial information and no repro example
* repro cannot be reproduced locally and needs network connectivity

Finally, you should be maintaining a work log where you are meticulously tracking all the steps you used to fix this issue. This means that you note down the path of the repro example, behavior that you were seeing, commands used to run the repro etc. This will help you to communicate and track your progress when debugging issues over days. Another advantage of this kind of logging is that you can use it as a rubber duck. A rubber duck (if you haven't heard of that term before) is a kind of proxy soundboard for figuring out issues. People generally have an aha moment when they are talking to their teammates about bugs. The idea is that you have this aha moment when the rubber duck acts as a proxy for your teammate. When you explain an issue to someone else, you often realize things you missed or anticipate questions that can help you either explore more debugging paths and resolve bugs.

# Communication

I am calling this out because I can't stress how important this is. This speaks to how effective you are in understanding and getting information from the user, collaborating with teammates and finally responding and reporting the issue.

Often times users don't know what they are looking at. Our code is littered with unintelligible stack traces that make sense to engineers on a single team. This should not be the case in an ideal world. If you ask the right questions you will be able to get the complete picture of the user's setup, commands they used, software versions, job logs etc. Often times, the stack trace is all that the user shares but it is imperative to understand the full picture.

Once you have the information from the user, you will need to employ your communication skills for two possibilities:

* You might need help from your teammates if you get stuck. For this you should try solving the issue and make a note of what you found. The detailed work log will help you in this. When communicating to you teammates, you should tell them the different things you tried and the resulting behavior. This is usually sufficient for teammates to start chiming in. What you don't want is to type a vague message with no details about the bug or what you did to solve it. Chances are you won't see a lot of responses.

* The other communication muscle you may need to exercise is working with multiple people across teams to solve an issue. Complex technology stacks will often mean that you will need folks from multiple teams to chime in on an issue. You will need to find those teams, contact the individuals and work with them to solve the issue. It is good practice to be polite and ask people to take a look but also follow up with them at a cadence that reflects the priority of the issue you are working on. The other team members have their own tasks but this is also an important one. Respecting your coworker's time but also ensuring that you are making progress is a fine balancing act that you learn over time.

The final communication once you have resolved the issue is to update the user and other teammates that you collaborated with. You want to write a quick one line summary of the issue that the user will understand. Next, you want to write a detailed summary for the team so that everyone understands what went wrong. Finally you want to identify if there were any improvements that could be made so that users are not plagued with these issues.

Debugging is a skill but identifying steps and using tools effectively can make you a master bug basher! 
