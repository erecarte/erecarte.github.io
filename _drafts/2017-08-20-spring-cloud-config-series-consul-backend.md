---
layout: post
tags: 
   - Spring Cloud
title: "Spring Cloud Config Series Part 4: Consul Backend"
author: Enrique Recarte
---

Time in our Spring Cloud Config Series for the Consul backend. Let's see how it works and how it compares against the other ones.

# Introduction
If you have read [the previous post about Zookeeper]({% post_url 2017-08-11-spring-cloud-config-series-zookeeper-backend %}), you'll find that this one is quite similar. That is because [Consul](https://www.consul.io/) and
[Zookeeper](https://zookeeper.apache.org/) are really similar, making it sometimes a bit difficult to understand the difference between them. After having gone through
their documentation, I would say that the difference is that Consul is a service discovery tool which can also be used for consistent data management, 
while Zookeeper is a consistent data management tool which can be used for service discovery. Consul is much more geared towards service discovery, providing
some tools around it like complex health checks and DNS resolution. Zookeeper on the other hand is more a *build the service discovery yourself* kind of tool.

Consul is distributed and agent based, which means you deploy a single agent process on each node or server and applications strictly communicate with
the local agent. The local agents will then use the [Gossip protocol](https://www.consul.io/docs/internals/gossip.html) to communicate
any changes to other agents in the cluster. This is a very simplified description of how consul works, but it should be enough for our purpose. If you
are interested in finding out a bit more about Consul's architecture, [the documentation on their website](https://www.consul.io/docs/internals/architecture.html)
is quite good.

# Using Consul as the backend
Consul provides a Key/Value store which is pretty straightforward. However, Spring Cloud Config provides a few ways to structure the data in these Key/Values.

## Standard Key/Value format
The default when using the Spring Cloud Consul library is to use the Key/Value store as it is, which means you have the name of property
being the **Key**, and the value for it being the actual **Value**. 

When using this option, the Key/Values are structured in a very similar way as a filesystem, where a key is identified
with an identifier like `/config/folder/server.port`, and that key would have a value like `8080`.

The way you structure your Key/Values are as follows:

```
/config/ConsulSampleApplication,PROD/
/config/ConsulSampleApplication,UAT/
/config/ConsulSampleApplication/
/config/application,PROD/
/config/application,UAT/
/config/application/
```

- By default all your configuration lives under the `/config` folder. You can configure this root folder by setting the `spring.cloud.consul.config.prefix` property.
- In `/config/{application-name}` is the configuration that is specific for your application.
- In `/config/{application-name},{profile}` you’ll find the configuration for your application when running with that specific profile. This is equivalent to the `{application-name}-{profile}.properties` file in the Git solution. By default, you separate the profile with a comma, though you can change this by setting `spring.cloud.consul.config.profile-separator`. For example, if you set it to -, then your folder would be `/config/ConsulSampleApplication-UAT`.
- The `/config/application` folder contains the configuration that applies to all applications. This folder is also configurable by setting the property `spring.cloud.consul.config.defaultContext`.
- Finally, if you want to configure something for all applications for a given profile, for example, the logging level for all applications in the UAT environment, then you should define the properties under the `/config/application,{profile}` folder.

 
## Properties or YAML format

## git2consul format 


## Architecture Overview
The Architecture of a Consul is similar to the other Spring Cloud Config setups, except that Consul requires all nodes that are part of the
Consul cluster to have a Client Agent running. This agent is responsible for 

![Consul Setup]({{ site.baseurl }}/img/scc-consul-setup.png)

## Implementation Details
In order to use this setup, assuming that your applications are already Spring Boot apps, you should do at least the following things:

1. Your client applications will need to import the Spring Cloud Config Consul Client maven dependency.

```xml
   <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-consul-config</artifactId>
   </dependency>
```

2. Define a file named `bootstrap.properties` in the root of your classpath, which will need to include the following properties:

```properties
   # The name of the application
   spring.application.name: ZookeeperSampleApplication
   # The host for your Consul agent
   spring.cloud.consul.host: localhost
   # The port for your Consul agent
   spring.cloud.consul.port: 8500

````
As you can see, these properties default to `localhost`. Given that every node in the Consul cluster needs to run a client agent, usually
the default properties will work out of the box because that is where the agent will be running.


# Detecting Configuration Changes
The Spring Cloud Config Consul library comes with a `ConfigWatch` which will run a scheduled job to refresh the configuration, so it basically
does polling on the configuration. You can set up the refresh rate with the property `spring.cloud.consul.config.watch.delay`, which is defined
in milliseconds and defaults to `1000`.

# Conclusion

You can find the example code in [this Github repository](https://github.com/erecarte/blog-spring-cloud-config/tree/master/consul) and
play around with it.
