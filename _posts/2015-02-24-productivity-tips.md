---
layout: post
author: Enrique Recarte
category: Productivity
title: "Please be productive!"
---

These last few months I've been working with an overseas development team in the company as a consultant, trying to raise their awareness
of code quality and provide some feedback on how I think they could improve individually and collectively as a team of software developers.

One of the main things I've been talking to them about is the need for [Refactoring](http://en.wikipedia.org/wiki/Code_refactoring). I think that most of the times in the past,
this team has been told what to do and how to do it, which means they haven't put that much thought into the design of what
they are actually developing. Not caring about the design in turn means they didn't really care about whether the solution could be
improved or cleaned up at all.

After a few months of talking to them about what refactoring is and trying encouraging them to use some techniques like the ones
described in [Michael Feathers's excellent book](http://www.amazon.co.uk/Working-Effectively-Legacy-Robert-Martin/dp/0131177052)
I felt like I wasn't getting anywhere.

I had the opportunity to come on-site to help them out for couple of weeks. During that time, I wanted to do as many pair programming sessions
as possible and try getting to the main reason why things are not improving as quickly as they could.

After a few sessions, I started seeing a pattern throughout the team which could explain why they were struggling. I set up a session for the whole
team in a meeting room with a screen. I asked one of the developers to take control and do the following:

- Checkout a specific git project from scratch.
- Add a class which just says "Hello <name>" where <name> is whatever you passed to the method.
- Add a test class with 2 tests with 2 different names. I asked the developer to add the test to a module which didn't have
any tests yet to make sure he had to add the `junit` dependency in Maven and the testing folder structure.

The developer started working on it and did something like this:

1. Find in our GIT projects web page (we use [Atlassian Stash](https://www.atlassian.com/software/stash)) the URL to clone the project.
2. Use the GIT GUI tool to checkout the project.
3. Open IntelliJ and click on `Import Project` finding with the mouse the appropriate directory where the project had been checked out.
4. Find the `pom.xml` on which to add the `junit` dependency.
5. Find some other project they had installed locally that had the `junit` dependency so he can copy and paste it.
6. With the mouse, add the `src/test` directory by `Right Click -> New -> Directory`. And then do the same for `src/test/java`.
7. Find the package to put the class in and add it with `Right Click -> New -> Class`. At this point he added the code.
8. Add the same package in the test directory and again add the test class by `Right Click -> New -> Class`
9. Add the 2 tests manually by typing the full method including annotations.

I tried this process with another developer afterwards and they both spent about 20 minutes, one of them not even being able to finish
in that time.

Then I took control, and I showed them how I would have done it. Some of the differences where:

1. I use [Cygwin](https://www.cygwin.com/) and I like making my life easier. This means after a few times of manually doing the first 3 steps on the list, I added
a custom command like `.clone name-of-the-project` which will resolve the URL and check it out, as well as open the pom.xml file in
IntelliJ, which recognises it's a Maven POM file, and imports it automatically. This saves me quite a lot of time.
This is an example of what that bash script would look for cloning one of my Github repositories clone function:

<script src="https://gist.github.com/erecarte/2d449464fab3e65dc88404097aaf40bf.js"></script>

2. Once I find the pom.xml on which to add the junit dependency, I type `dep` and press `Tab`, which causes IntelliJ to add
a new dependency, asking you which one you want to add. If you start typing junit, it will resolve it for you.
3. I used keyboard shortcuts to add the main class. After having added the method, I create the test by pressing `Ctrl+Shift+T`.
4. For every test I want to add, I press `Alt+Insert` so I don't have to type the method signature or any annotations.

Overall it took me 4 minutes to finish the exercise. I pointed out that it had taken me 4 minutes what they couldn't finish in
20, and I asked them one important question: *"What was the time difference between us for the actual coding?"* We agreed that it took me maybe
3 minutes, while it had taken them maybe 5 minutes. That is not a huge difference! The main amount of time for them was spent in
other things that are not coding, like checking out a project, or doing lots of things with the mouse in the IDE instead of using
keyboard shortcuts.

Refactoring is something that takes practice and time to do right. It's very easy to refactor something carelessly and find out you broke
something. The fact that it takes quite a bit of time to master, means you need to make sure that this time you spend refactoring
is as much as possible about refactoring, and not anything else like finding out the exact test path where the test should go.

Developer productivity for me is probably the most important thing a developer needs. Even before learning good practices like
TDD or Refactoring, you need to make sure you are productive.

Imagine doing TDD without keyboard shortcuts:

- You add a test
- You write something like `myObject.myNewMethod();`.
- At this point the new method does not exist, so you need to use the mouse
to go to the production code and add manually `public void myNewMethod() {}`.
- Then go back to the test class and add something to the test which doesn't exist in the production code.
- ...

If I tried to do TDD like this, I'd probably give up quite quickly because you spend too much time. Same thing goes for Refactoring. If
I need to use the mouse and do some copy/paste for extracting methods and variables, I wouldn't do it very often.

If you are a developer, you should be interested in making your life as easy as possible, an example being the GIT command
I described above. Try finding things you spend time one besides the actual development, and think of ways you can automate it,
or at least find a tool to help you!

### Useful links
Here are some interesting posts and tools if you want to learn how you can be more productive:

- [Productivity tips for IntelliJ](http://blog.jetbrains.com/idea/2012/10/intellij-idea-productivity-tips-part-1/)
- [IntelliJ Refactoring basics](http://blog.jetbrains.com/idea/2014/01/30-days-with-intellij-idea-refactoring-basics/)
- [Uncle Bob live refactoring session](https://www.youtube.com/watch?v=y4_SJzNJnXU)
- [Ditto](http://ditto-cp.sourceforge.net/): Clipboard manager tool which avoids having to copy and paste in intermediate documents for history.
- [Introduction to Cygwin](http://lifehacker.com/179514/geek-to-live--introduction-to-cygwin-part-i)
- [Key Promoter IntelliJ plugin](https://plugins.jetbrains.com/plugin/1003): Very good plugin which will show a very annoying pop-up on the screen every time you don't use a keyboard shortcut.
