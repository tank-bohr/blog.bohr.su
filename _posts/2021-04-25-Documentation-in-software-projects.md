---
layout: post
title:  "Documentation in software projects"
date:   2021-04-25 02:30:00 +0300
---

Today I want to talk about documentation:

- What kinds of documentation could be.
- How it should look.
- How we should write it.

## Classification

I see two main kinds of documentation:

- API description
- motivation and rationale description

API description is a pretty straightforward thing. You have a library or a service, and it's handy to have something that can tell you how to interact with your software. In my opinion, such tools as Javadoc (Java), rdoc (Ruby), edoc (Erlang), and similar are the best choice for it because the documentation lives as close to the source code as possible. For HTTP APIs, one can use swagger specification inside the controllers (see [phoenix_swagger](https://github.com/xerions/phoenix_swagger), for example). And all these tools can generate pretty HTML sites from the source code for better representation. The second kind is much more many-sided and has several levels.

## Level 0: No documentation

All you have is a source code. The source code tells you very accurately **how** your system works. In most cases, it should be enough to figure out everything. But the source code means you nothing about **why** the system works this way and what problem it should solve. To answer this question, you need context.

## Level 1: Commit messages

You can start from the commit messages. Most often, you can see there a short description of what was implemented with this commit. But the true professionals put the ticket number beside. I believe every commit must have a link to the ticket in the task tracker because every line of code should have a reason to be. In a good ticket, you can find requirements. It's exactly what you need to realize the rationale for this particular code and the problem this code solves. Task tracker is a foundation for your documentation. The link between the commit and ticket should be bidirectional. You often want to find a commit or a pull request that meets the ticket's requirements. If you don't have a ticket for some reason, you should describe your intentions in the commit message.


## Level 2: A ticket from the task tracker

As I said before, any commit should be linked to the ticket. But the ticket could be good or bad. A bad ticket is empty or tells you only **what** to do and not **why**. A good ticket contains a business problem which you need to solve and optionally the way it can be solved.

Compare two tickets:

* Add the `birth_date` column to the `users` table in the database.

* We want to congratulate our users and wish them a happy birthday by email. That's why we need to store the birth date in the database.

The difference is crucial. When you have the second ticket, and someone asks why we need this column, the answer can be found extremely easy and fast. With the first one, it's a dead end. The first ticket can also be legit if it belongs to some epic where the original business feature is described.
Tickets can be hierarchical, but they should be concise and clear. The original task can be formulated very generally, for example, "Implement billing subsystem." One cannot write any code by this ticket, but he can write a dozen linked tickets based on product owner interviews and decomposition.

## Level 3: Wiki pages

Sometimes, it's impossible to express all the context in a ticket—for example, some research, brainstorming results, or specifications. But all those things can cause source code to be written in some particular way. But the ticket must be concise and clear and should not contain all the history but should contain the link to the wiki page. It allows anyone to get broader context if needed.

## How to write documentation

I have one simple rule that allows the documentation to be comprehensive, relevant, and not very time-consuming. The rule is **"Any activity must have an artifact as a result."** It's very usual for developers. Our main craft, software development, leaves artifacts in the source control systems. The same should be with all other activities. Attend a meeting - write a summary. Do some research - write a wiki page with the result. Interview a product owner - write tickets with requirements. If you do some dev support from time to time, you should have some journal for it. It's not necessary to write a comprehensive report on every complaint. Just write down a couple of sentences about what happened and how you solved it. If you fix some outage, write a post mortem (and yeah, now it should be a comprehensive report).
It's a very simple rule. It does not require much effort. Moreover, documentation should be as concise as possible. It should contain the very essence.
As a side effect, it reduces all possible meetings, calls, slack-chats dramatically. It allows making all the communications truly asynchronous. You don't need a human to get information anymore because all the information is externalized, structured, linked, and searchable.

## Conclusion

I believe we should treat documentation as history. Documentation cannot be outdated. If it happened, it happened. When we dig some legacy code, we like genealogists who try to figure out when your great-great-grandfather was born by the church records. And good engineer leaves a lot of clues for next generations. The generational change in software projects runs very fast and painful. The lack of documentation leads to the loss of the experience of the earlier generations. We have to rewrite everything from scratch, making the same mistakes, wasting time on useless meetings. Civilization is based on the passing of knowledge from one generation to another and accumulating it. It works the same in software projects.
