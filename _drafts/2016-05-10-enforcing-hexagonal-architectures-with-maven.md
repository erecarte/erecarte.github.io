---
layout: post
category: Architecture
title: "Enforcing Hexagonal Architectures with Maven"
"header-photo": paris.jpg
published: true
author: Enrique Recarte
---

How to organize your code is one of the most difficult things in Software Development. I'm not just talking about what package structure and names you should be using, but how to organize your code in a way that is easier to change, easier to understand and easier to test.

In the first few years of my career, when writing new code, I'd spend most of the time thinking about what technology would I use to do something. I'd be asking myself questions like *Should I store this configuration in the database or in an xml file?* or *Should I use Spring to implement this functionality?* The fact that these were the main questions I asked myself meant that my code was mostly about those things. You would see Spring code everywhere, or maybe SAX related classes all over the place.

After a while, things started to feel a bit wrong. Sometimes upgrading some of the frameworks we were using in our applications caused quite a bit of pain. Sometimes someone decided to move some logic out of a stored procedure into its own REST service and it seemed to affect a big part of the codebase. If you don't have the right level of isolation, things can become very messy very quickly.

![Alt text](https://g.gravizo.com/svg?
  digraph G {
    aize ="4,4";
    main [shape=box];
    main -> parse [weight=8];
    parse -> execute;
    main -> init [style=dotted];
    main -> cleanup;
    execute -> { make_string; printf}
    init -> make_string;
    edge [color=red];
    main -> printf [style=bold,label="100 times"];
    make_string [label="make a string"];
    node [shape=box,style=filled,color=".7 .3 1.0"];
    execute -> compare;
  }
)

## Hexagonal Architecture
A few years ago I started reading a bit on how to organize the code in my applications. After
reading some posts like Uncle Bob's [Clean Architecture](https://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html) and Alistair Cockburn's [Hexagonal Architecture](http://alistair.cockburn.us/Hexagonal+architecture), I started to realize that I had never really thought about how the applications would evolve over time. The main thing I'd take away from those posts is: ***keep the things that vary at different rates in different parts of your application***. It's important to remember that changing code costs money. You are adding risk to something that was working. You need to test it again.

Given that we want to keep things that vary at the same rate together, think about this: Frameworks change very often. Technologies change quite often. The actual business logic you are going to write won't probably change very often.

Frameworks and Technologies usually go hand in hand, because that's mostly where frameworks shine. They help a lot interacting with some technology.

Domain logic should be the most important thing in your application. It's the reason the application exists. It's what you are really paid for. Think about it, no application would ever created to handle JMS messages with the minimum amount of code. An application could be created to handle a payment as quickly as possible. The actual implementation might include listening for a JMS message and acting upon receiving it, but it's not its main purpose.

After these two last points, I like to make a big distinction in my code:

- Domain logic
- Integration logic

Thinking of my applications like this, helps me answer these questions:

- Where does this component go?
- What kind of testing should I apply to test this?
- How hard would it be to replace this technology?
- How hard would it be to replace the source of this data?

After applying this pattern, I found that moving some logic from a stored procedure to a REST service only affected 1 or 2 classes and it took a much shorter time that it would have before.
