---
layout: post
category: Spring Cloud
title: "Spring Cloud Config Overview Series Part 1: Standalone GIT Backend"
author: Enrique Recarte
published: false
---

I haven't written a post in a very long time and I thought I'd get back on track by announcing I've just contributed for the
first time to an open source project.

I'm a big fan of the [Spring Cloud](http://projects.spring.io/spring-cloud/) set of tools. I've been using it for a while both in a
professional capacity (though quite limited) and a personal one. If you have not used it before, you should try it out. You'll see how
convenient it is to set up a Micro-service environment where your applications can follow the [Twelve Factor App Manifesto](https://12factor.net/).

At work, we have started using [Spring Cloud Config Zookeeper](http://cloud.spring.io/spring-cloud-static/spring-cloud-zookeeper/1.1.0.RELEASE/#spring-cloud-zookeeper-config), which
provides a way to have distributed configuration for your applications by using [Zookeeper]() as a configuration repository. If you have
never heard about Zookeeper, it's a *"centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services"*.

By using Spring Cloud Config, your application will read some configuration from Zookeeper before starting up, and it will listen for changes whenever there is any updates
 in Zookeeper. The one thing I thought was missing from the library, was the capacity of letting your application start up even if Zookeeper was not available at the time. Ideally this
 should no happen since Zookeeper should be highly available, but we've had some issues at our company where Zookeeper was not running when starting an application,
 but we still wanted the possibility of starting up the application and register for changes in the connection to load up the Zookeeper configuration once
 it was available again.

That was essentially [my contribution](https://github.com/spring-cloud/spring-cloud-zookeeper/pull/106) to the project. I've added a runtime property which you can declare in your `bootstrap.yml` file which will specify whether you want your application to fail
fast or not. Below you can find a sample configuration:

## Introduction to Spring Cloud Config

## Standalone GIT Backend Setup

![Standalone GIT Setup]({{ site.baseurl }}/img/standalone-setup-2.png)


## Refreshing the configuration

## Conclusion
### Advantages

### Disadvantages

