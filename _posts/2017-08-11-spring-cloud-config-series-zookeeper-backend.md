---
layout: post
tags: 
   - Spring Cloud
title: "Spring Cloud Config Series Part 3: Zookeeper Backend"
author: Enrique Recarte
---

In the next part of the series, we take a look at how we can use Spring Cloud Config with [Zookeeper](https://zookeeper.apache.org/) to manage
our configuration. 

# Introduction
I've always found it hard to define what Zookeeper is. This is the definition given in its website

> ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.

In short, the way I think about Zookeeper, is basically a system that stores information and makes sure it's consistent and synchronized
across distributed systems. What kind of information you store in it, is up to you. You could store some information which will
help with [Leadership Election](https://zookeeper.apache.org/doc/trunk/recipes.html#sc_leaderElection) or implement [Distributed
Transactions](https://zookeeper.apache.org/doc/trunk/recipes.html#sc_recipes_twoPhasedCommit).

In our case, we are going to use it to store our application configuration.

# Using Zookeeper as the backend
Zookeepers stores its data in something called [ZNodes](https://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html#ch_zkDataModel). You can think
of these ZNodes as a filesystem. The way a ZNode is identified is very similar to files as well, so for example, you can have a ZNode
identified by `/config/ZookeeperSampleApplication/server.port` which can contain a value like `8081`. 

The `/` separates ZNodes, and these ZNodes can have values and children ZNodes at the same time, so `/config` can also have a value as well as be the
parent of the `ZookeeperSampleApplication` ZNode.

In order to set the data into Zookeeper, you'll need to use the Zookeeper client, or try finding a User Interface available. In our example, 
we will use [Exhibitor](https://github.com/soabase/exhibitor/wiki), which was created by Netflix and then released as Open Source.

## Structure of the ZNodes in Spring Cloud Config
The same way that your runtime properties were structured in property files in the Git backend example, with Zookeeper,
this configuration will be structured in ZNodes. Let's take a look at how they are divided. 

Assuming you have an application named `ZookeeperSampleApplication`, and you have UAT and PROD environments, 
then your ZNodes containing configuration (like folders) would look something like this:

```
/config/ZookeeperSampleApplication,PROD/
/config/ZookeeperSampleApplication,UAT/
/config/ZookeeperSampleApplication/
/config/application/
```

Let's take a closer look:

- By default all your configuration lives under the `/config` ZNode. You can configure this root node by setting the `spring.cloud.zookeeper.config.root`
property.
- In `/config/{application-name}` is the configuration that is specific for your application.
- In `/config/{application-name},{profile}` you'll find the configuration for your application when running with
that specific profile. This is equivalent to the `{application-name}-{profile}.properties` file in the Git solution. By default, 
you separate the profile with a comma, though you can change this by setting `spring.cloud.zookeeper.config.profile-separator`. For
example, if you set it to `-`, then your ZNode would be `/config/ZookeeperSampleApplication-UAT`.
- Finally, `/config/application` contains the configuration that applies to all applications. This ZNode is also configurable by setting
the property `spring.cloud.zookeeper.config.defaultContext`.
- The one thing that is not possible compared to the Git solution is that you can't have configuration that applies
to all applications for one profile, so for example if you wanted to declare the global URI for some service and you always want it to be the
same for all applications in the UAT environment, then you couldn't just declare it in one ZNode, while you could
declare it in a `application-UAT.properties` file in Git.

To be more specific, let us look at a more concrete example. If you wanted to define the logging level (confgured in Spring
Boot by setting the property `logging.level.ROOT`) to be different per environment for your application, as well
as defining defaults for both your application (when the profile is not matched) and any other application, your ZNode tree would look like this:

![Exhibitor ZNode Tree]({{ site.url }}/img/exhibitor-example.jpg)

You can see that each of the ZNodes containing properties has a property `logging.level.ROOT` defined, and for the selected ZNode,
which configures the UAT profile, the value is `DEBUG` (look at the "Data as String" field)

## Architecture Overview
The Architecture for this setup is a bit simplified compared to the one we saw in the [previous post]({% post_url 2017-08-04-spring-cloud-config-series-git-backend %}) for the Git setup. 

![Zookeeper Setup]({{ site.baseurl }}/img/scc-zookeeper-setup.png)

Spring Cloud Config will add a `ZokeeperPropertySource` to your application which will use [Curator](http://curator.apache.org/) to
communicate with Zookeeper. Then you just need to setup the Zookeeper cluster and tell your application where it lives.

## Implementation Details
In order to use this setup, assuming that your applications are already Spring Boot apps, you should do at least the following things:

1. Your client applications will need to import the Spring Cloud Config Zookeeper Client maven dependency.
```xml
   <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-zookeeper-config</artifactId>
   </dependency>
```

2. Define a file named `bootstrap.properties` in the root of your classpath, which will need to include the following properties:
```properties
   # The name of the application
   spring.application.name: ZookeeperSampleApplication
   # The connection string to for your Zookeeper instance.
   spring.cloud.zookeeper.connectString: localhost:2181
````
This file will tell Spring Cloud to load up the remote Zookeeper configuration on the bootstrap phase of your application
startup, before anything else happens. Once the Zookeper configuration is loaded, the library will identify which ZNodes apply
to the current scenario (application name and profiles), and will add a `PropertySource` to the environment with the highest
priority to make sure that Zookeeper properties are looked up before any other sources such as property files or system properties.

You can find the example code in [this Github repository](https://github.com/erecarte/blog-spring-cloud-config/tree/master/zookeeper) and
play around with it.

# Detecting Configuration Changes
Zookeeper comes with built in support for receiving events when ZNodes change. Spring Cloud Config will hook into this events
and will add listeners to the ZNodes it cares about. When these ZNodes are added, removed or updated, it will refresh the context.

Remember that, even though the context will be refreshed automatically, you still need to make sure that the way you read your configuration
will support the refresh automatically, otherwise you may need to annotate the beans that use the configuration with `@RefreshScope`. For more information
on how Spring Cloud manages the configuration and the refresh events, take a look at the [first post of the series]({% post_url 2017-07-21-spring-cloud-config-series-introduction %}).

This automatic refresh mechanism can be enabled or disabled by setting the property `spring.cloud.zookeeper.config.watcher.enabled` which
is `true` by default. As with any Spring Cloud Config application, as long as you have the [Spring Boot Actuators](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html)
enabled, you can send a `POST /refresh` request to your application, which will refresh the context.

# Conclusion
We do have Zookeeper instances in my current project, and in the past I've seen how some teams in my current company would write
quite complex code to integrate with it to manage their configuration, so I was always a bit hesitant about integrating with it.

Then, after Spring Cloud Config Zookeeper came out, and took a look at how easy it was to configure it, we immediately started using it. Having
a library that will manage the ZNode structure for you as well as listen to events and act accordingly is a huge addition to any project that
already uses Zookeeper.  

Having said that, I think Zookeeper has a few drawbacks as a configuration management system:

- It's quite a complex system. I've read quite a bit of its documentation, and I still don't have a very clear idea of what it is. If you haven't used
it yet, I think you should make sure someone in your team or company knows it quite well.
- It does not have a User Interface by default. It does have some command line utilities, but managing all your configuration in Zookeeper without a
proper User Interface can be quite painful. There are some UIs around, like [Exhibitor](https://github.com/soabase/exhibitor/wiki) shown in the
previous example, [zkui](https://github.com/echoma/zkui) or [zk-web](https://github.com/qiuxiafei/zk-web).
- Authentication, Authorisation and Auditability are not straightforward or non-existing. We had an issue in Production once because someone
had changed a ZNode with the wrong value, and there was no way to know that this had happened, or who did it and when. Maybe there
are some additional options to add some of these features, but as I said, it looks like there is a lot of complexity in setting up some
of the desired features you would want in your Configuration Management System, like approving changes, auditing them, etc.

Overall, I think that from the infrastructure point of view, Zookeeper is quite a complex solution, but on the other hand, from an
application point of view, with a library like Spring Cloud Config Zookeeper, it becomes a very interesting option. 
