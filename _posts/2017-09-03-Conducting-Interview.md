---
layout: post
title: My way of Conducting an Interview
excerpt_separator: <!--more-->
---

Interviewing people is not an easy job to do. You want to find the person which is going to get things done, enjoy working with given project, fit into the team and be happy about the money you can offer.

As an interviewer, you are also being judged by the candidate. You very often create the first impression of the company. So you also need to make a good impression. Nobody wants to work with mean or incompetent people!

In this blog post, I am describing my way of conducting the interview. In my career, I have interviewed a hundred developers and hired over a dozen of them. So my experience is not very reach, it's limited to "my sample". I recently changed job and I don't interview people anymore but I hope that my experience can help somebody to improve the interviewing process.

<!--more-->

## Evolution

My interviewing style has evolved over the years. Initially, I was focused on asking very strict technical questions. Some examples:

* what are the differences between Value and Reference Types?
* how GC works?
* what is the difference between `DROP TABLE` and `TRUNCATE TABLE`?

**But I very soon realized that the fact that somebody can answer similar questions does not mean that she or he can get things done.**

**Also, the fact that somebody does not know the answers does not mean that he or she can't search for them and learn fast when needed.**

## Homework

Before I start interviewing I do the homework: **I read the candidate CV**, mark the things that I want to talk about. If I don't read the CV before the interview, and during the interview, I am surprised about things that were stated in the CV it is just a **disrespect**.

> Interviewer: I did not know that you are not graduated in Computer Science.  
> Candidate: But I have described my education in the resume.  
> Next: An awkward moment of silence.

**Find out as much as possible about the project that you are interviewing for.**
Is it some kind of a rocket science? Or maintenance? Or simple CRUD?
What technology?
Does it require a lot of travel?

You need to find a person that is going to fit the project and the team. If you know too little about the project it's going to be hard or impossible. **Be prepared!**

## Relax!

Candidates are typically very nervous at the beginning of the interview. If you start asking hard questions to a person who barely breaths and just wants to run away from the room you won't get good answers.

So as an interviewer I always focus first on chilling out the candidate. I start with some Chit Chat about some positive things. An example:

> The weather outside sucks. I need to go for a holiday to recharge. What was your recent holiday destination?

After that, we might talk for a few minutes about holidays. The candidate just needs to start talking!

If the candidate has not been on holidays for years you can say what the company has to offer. An example:

> We offer 30 days of fully paid holidays. We develop products for our internal purpose, so there are no super-strict deadlines and you can take a week off anytime you want to.

I keep the conversation positive and informal. I continue to the next stage when I feel that the candidate is not nervous anymore.

*And yes my dear US readers, 30 or even 35 days of fully paid holidays is totally possible in Europe. The same goes for unlimited sick days.*

## Warmup

In the beginning, I ask some simple, but very important questions. I also say it loud and clear that I am searching for a good fit for the project and I am expecting honest answers.

1. What's your favorite thing about programming?
2. What are the things about programming that you don't like?
3. Could you describe your dream job?

Some candidates say that they can do anything. In such case, I ask if they would be happy to debug some old Java scripts in Oracle Bus or migrate a relational database with no documentation to NoSQL cloud database in two weeks. I need to make sure that they understand that I am asking these questions to avoid putting them to a project they are not going to enjoy.

The answers help me to understand if given person can be a good fit for the project and the position that I am recruiting for.

If the candidate is a good software engineer but not a good fit for this particular project I offer a different project or just stay in touch. Otherwise, if I hire that person then she or he will not be satisfied and probably just leave. Everybody is going to lose a lot of time, and the company a lot of money. **Don't be selfish, think future-wise.**

The funniest answer I ever got:

> Me: Could you describe your dream job?  
> Candidate: I would like to be a Team Leader.  
> Me: Why?  
> Candidate: Because I would like to make important decisions.  
> Another interviewer: Would you also like to keep the team motivated, help with planning and estimations?  
> Candidate: No. Just making the important decisions.

## Learning

It's very hard to find a perfect candidate that is familiar with specific product and technology. Moreover, the requirements change over the time so in my opinion the most important thing is the possibility to learn. So I ask a LOT about learning.

1. What is your favorite way of learning new things?
2. When was the last time you learned something new? What was it? How did you apply it at work?
3. Do you have some gurus? Who are they and why do you value them?
4. When was the last time when you did not agree with some blog post/book/video? What was that and why you did not agree?
5. When was the last time you have changed one of your programming habits? What was that and why?
6. When was the last time when you shared the knowledge with somebody else? Did you like it? Why did you do that?

The answers help me to understand if given person enjoys learning new things and is self-motivated or needs a manager to tell him/her to read a book. I also want to understand the learning process, validation of the information and using it in practice.

This is also the moment when good software engineers are fully relaxed and enjoy the conversation. So I can move on to the next part.

## Problem solving and decision making

If I got here it means that the candidate can be a good fit and enjoys learning new things. So what I need to check is problem-solving and decision making. Of course, it's impossible to fully test that during the interview, but I can at least try.

Here, I start with something like: *Could you tell me about your last big assignment that you were working on? What were you supposed to implement, what were the steps that you took and which technology did you choose? Keep the company secrets for yourself, I just want to understand the way you approach and solve problems.*

From here I let the candidate talk and later I ask a lot of questions:

1. Why did you have to do that?
2. How did you start the task? Did you do some research? Did you talk with the customer?
3. Which technology and/or components did you choose and why?
4. When did you write tests?
5. Was there anything that you could do better?

**I ask as many questions as needed. Here I very often go deep into the technical details.** I ask deep technical questions related to the tools and compontents used by the candidate to understand if candidate knows how they work. If the candidate does not answer some questions I say something like: *Don't be afraid, it's perfectly fine that you don't know something. I am just testing your knowledge. I don't expect you to know everything.* Of course, sometimes it's just a lie when given person fails to answer a simple question. It's important to not freak out the candidate!! 

I want to hire people who want to understand problems before solving them. Who choose the best tool for given problem. Who write tests first to make sure everything works fine and the problems don't come back. Those, who understand how the technology they use works.

Very often the initial answer is very promising but the problem-solving skills are bad. One of such of my interviews:

> Me: Could you tell me about your last big assignment that you were working on?  
> Candidate: I have been working on improving the performance of our critical system. I have improved it by 20%. (sounds cool!)  
> Me: how did you start? Did you do some research?  
> Candidate: I instrumented every method call in the code with stopwatch and logged the output to the logs.  
> Me: How did you do that?  
> Candidate: I manually added stopwatch to every public method.  
> Me: Why you did not use a profiler? Did you run out of licenses?  
> Candidate: What is a profiler?  
> Me: Nothing important, I was just curious. (lie)  
> Me: How did you found out which methods were taking most of the time?  
> Candidate: I just read all of the log files.  
> Me: In which environment were you testing the performance?  
> Candidate: I copy pasted the instrumented dlls to a live system of one of our customers who was complaining about perf.  
> Me: Did you create a backup of the app first?  
> Candidate: Yes (I guess it was a lie too).

So given candidate somehow managed to solve the problem, but did not use the right tool and moreover did not perform the right research. It took a lot of time and risked issues at production.

When a smart developer faces a new problem then she or he starts the web browser and performs a search. Chooses the right tool and gets the job done. If the problem is more complex it might require reading a book, discussing it with other teammates, architect or tech lead.

## Fuck up

This is simple. I just ask about the most recent fuck up. What was that? Why did it happen? What did you learn from it? What did you do to make sure the problem does not occur again?

I want to hire people who can acknowledge that they did something wrong and learn from their own mistakes. Nobody wants to work with narcissists.

People are typically afraid to answer this question so I encourage them by talking about my recent fuck up. This helps a lot and opens most of the candidates.

## My recent assignment

I never ask the candidates to write code during the interview, instead of that I describe them my recent assignment and ask how they would solve it. I ask about my real task because I want to check how they would approach a real problem. I also know a lot about possible solutions because I always do a good research. It also helps me to make every interview unique.

When the candidate answers: *I would just google for it* I give my laptop to the candidate and let her/him search and read for a few minutes. I check what they put to the search engine. Most of the candidates are very surprised when I do that. But I am searching for people who can solve new problems, not artifical tasks from *Cracking the Coding Interview* book. And solving new problems typically requires to perform some web search.

## Money

When it comes to the money, I never offer a lowball salary or overpay when compared to the other team members. People talk to each other, do the math or web search and once they find out that they earn much less than others on the same level they get angry and leave. The company is losing a lot of money, more than lowball offer could save. It's just a matter of time!

## Summary

1. Talk about something positive and not related to interviewing to relax the candidate.
2. Ask about what the candidate really wants to do and is not willing to do. Is given position and project a good fit?
3. Ask about learning because learning is the fundamental part of being a software engineer.
4. Ask many questions about recent assignment to find out how given candidate approaches problems.
5. Ask about a failure to make sure you filter out the narcists.
6. Ask about your recent assignment to find out how given person would solve a real problem related to the job that you are interviewing for.

Good luck with your interviews!!

