---
layout: post
tags: 
   - Spring Cloud
title: "Spring Cloud Config Series Part 2: Git Backend"
author: Enrique Recarte
---

After the [first part of the series](http://www.enriquerecarte.com/2017-07-21/spring-cloud-config-series-introduction), where we looked at
a Spring Cloud Config introduction, in this post we'll look at a concrete implementation with a [Git](https://git-scm.com/) backend and the 
different options available using this backend.

# Introduction
As we saw previously, one of the factors in the [Twelve Factor App Manifesto](https://12factor.net/) was to move the configuration out of our
application. There are quite a few different ways of doing this, but here are some other concerns you need to think about when choosing a tool
to manage your configuration:

- **Auditing changes**: Should be easy to see the history of what changes were done by whom and when they were applied.
- **Approval of changes**: A lot of production issues are caused by wrong configuration. Before making a configuration change, ideally someone 
should take another look at it.
- **Resiliency**: What happens if the configuration tool is not available?
- **Synchronization**: How do the applications know that there is a change in the configuration?

Looking at the bullet points above, most of them describe a [Version Control System](https://en.wikipedia.org/wiki/Version_control), which is
what we use to store the source code of our applications. The same way you sometimes want to know who changed some code in a Java class 
and why, you should be able to find out who changed some timeout configuration and why, so why not use a VCS for your configuration too?
 
That is exactly what using Git as a Spring Cloud Config backend gives you. First class control over your configuration changes. If you mix Git's
capabilities with something like Pull Requests, you have got a great workflow to have better visibility over the configuration changes applied
to your environment, as well as better tools to figure out if something in the environment has changed, who did the change and a description
of the reason for that change.

# Using Git as the backend
When using Git as your Spring Cloud Config backend, there are three main libraries we need to talk about: 

- `spring-cloud-context`: As we saw in the previous post, Spring Cloud introduced the concept of a `bootstrap` application context, which uses
a `PropertySourceLocator` to load remote configuration. This interface is the entry point to the different backend implementations and it lives
in this library, which is included in any Spring Cloud application. It also contains the `RefreshEndpoint` which will allow us to trigger
 a refresh of the application context by invoking `POST /refresh`.
- `spring-cloud-config-server`: This is the default backend implementation for Spring Cloud Config. It adds a `PropertySourceLocator` which will
use one or more `EnvironmentRepository` that talk to different systems. These repositories are the ones responsible for resolving the `Environment`
that applies for a specific `application`, `profile` and `label`. There are a few different implementations of `EnvironmentRepository` 
that can connect to Git, SVN or [Vault](https://www.vaultproject.io/).  
- `spring-cloud-config-client`: This library is used when you deploy the Spring Cloud Config Server into its own application. When the server
is independent, you need a client which will connect to it through HTTP. That is exactly what this library does, it adds a `PropertySourceLocator` that
will call the server to load up the applicable configuration.

## Structure of the Git Configuration Repository
If you have used Spring Boot before, you are probably familiar with [how it loads configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html).
The main way you usually manage your configuration is by adding some `application-{profile}.properties` or `application-{profile}.yml`, having the 
environment specific properties in different profile files. For example, if you have a UAT environment and a PROD environment, you could have something like 
the tree below, where the profile specific files contain configuration for that specific environment, and the `application.properties` file contains configuration 
properties that apply to all environments. 

```
$ tree
.
├── application-PROD.properties
├── application-UAT.properties
└── application.properties

```


With Spring Cloud Config, this property structures varies slightly. Since you will most likely use a Git configuration repository to store configuration for several applications, the naming structure needs to support it. The
way this is supported is by naming the files after the application names, so if in the previous example, your application was named (using 
`spring.application.name`) as `StandaloneGitFirstClient` and you also had another application named `StandaloneGitSecondClient`, then your property files 
in your configuration repository would look something like this:

```bash
$ tree
.
├── StandaloneGitFirstClient-PROD.properties
├── StandaloneGitFirstClient-UAT.properties
├── StandaloneGitFirstClient.properties
├── StandaloneGitSecondClient-PROD.properties
├── StandaloneGitSecondClient-UAT.properties
├── StandaloneGitSecondClient.properties
└── application.properties
```

Looking at the tree, we now group the `.properties` very similarly, but instead of just using a generic `application-` prefix, we use the application
name.

You have probably noticed that there is still an `application.properties` file in there. If this file exists, its properties will apply to all applications
in all environments. You can also have an `application-UAT.properties` which would apply to all applications in the UAT environment, etc.

Spring Cloud Config also gives you options to have different directories for each application, or even different repositories. If you want more information
on the different options available, check out the [Reference Documentation](http://cloud.spring.io/spring-cloud-static/spring-cloud-config/1.3.1.RELEASE/#_spring_cloud_config_server).

## Architectural Options
As we described earlier, the Spring Cloud Config Server is the library that communicates with Git, and once the repository is cloned, it figures out
based on the different parameters which configuration properties apply.

This library can be used in two different ways, having a central application which will use this library and getting all client applications to communicate
with this server application, or including the library into all your client applications. Let's look at these options in more detail. 

### Standalone Config Server Setup
The first option we are going to look at is the more traditional one where we deploy a separate application which manages the configuration for us. In the diagram
below you can see what it looks like:

![Standalone Config Setup]({{ site.baseurl }}/img/scc-standalone-setup.png)

As you can see, the Spring Cloud Config Server lives in its own application. All this application does, is clone Git repositories, listen for requests
from client applications asking for configuration, and return the applicable configuration for those requests.

The client applications themselves just include the `RefreshEndpoint` we described earlier, we well as the Spring Cloud Config Client, which will run in the bootstrap phase, calling the server for the applicable configuration
passing the name, profile and label. This last parameter can be a specific Git branch or commit.

#### Implementation Details
In order to use this setup, assuming that your applications are already Spring Boot apps, you should do at least the following things:

1. Your client applications will need to import the Spring Cloud Config Client maven dependency.
```xml
   <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-config</artifactId>
   </dependency>
```

2. Define a file named `bootstrap.properties` in the root of your classpath, which will need to include the following properties:
```properties
   #The name of the application
   spring.application.name: StandaloneGitFirstClient
   #The URI of your Spring Cloud Config Server
   spring.cloud.config.uri: http://localhost:8888 
```
This `bootstrap.properties` file defines the configuration that is used in the bootstrap phase of your application startup. Since the remote
configuration is loaded before the main application context starts up, we need a different file to define the configuration that applies
for this specific part of the process. Usually this file just contains properties for Spring Cloud Config concerns.

3. In your server application, import the Spring Cloud Config Server maven dependency:
```xml
   <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-config-server</artifactId>
   </dependency>
```

4. Configure your server application with the appropriate Git repositories by editing the `application.properties`. 
```properties
   # The URI of the Git Repository where the configuration is stored
   spring.cloud.config.server.git.uri: https://github.com/erecarte/blog-spring-cloud-config-configuration.git
   # The port on which this application runs
   server.port: 8888
```
The reason this application
updates the `application.properties` and not the `bootstrap.properties` is because the server does not need to use the Git configuration for itself, 
it just uses it to return it to other applications, so no bootstrap configuration is needed.

5. Annotate your main application class in the server with `@EnableConfigServer`:
```java
   @SpringBootApplication
   @EnableConfigServer
   public class StandaloneGitServerApplication {
      public static void main(String[] args) {
         SpringApplication.run(StandaloneGitServerApplication.class);
      }
   }
```


After you have followed those steps, you should first start your server application, and then start the clients. When the clients are starting
up, the first log entries you'll see look like: 

```
2017-08-03 14:07:57.708  INFO 74429 --- [           main] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@4f6ee6e4: startup date [Thu Aug 03 14:07:57 BST 2017]; root of context hierarchy
2017-08-03 14:07:57.904  INFO 74429 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'configurationPropertiesRebinderAutoConfiguration' of type [org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration$$EnhancerBySpringCGLIB$$ed33cade] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.5.2.RELEASE)

2017-08-03 14:07:58.142  INFO 74429 --- [           main] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at: http://localhost:8888
2017-08-03 14:08:01.631  INFO 74429 --- [           main] c.c.c.ConfigServicePropertySourceLocator : Located environment: name=StandaloneGitFirstClient, profiles=[default], label=null, version=null, state=null
```
If you look at the last two lines of the snippet, right before doing anything, it first fetches the remote configuration from the Spring Cloud Config Server, and only then it starts loading
up the application context.

If you want to look at an standalone setup example in a bit more detail, the code is available [in Github](https://github.com/erecarte/blog-spring-cloud-config/tree/master/standalone-git).
This sample repository contains two client applications and the server. They configuration for them is in [this other repository](https://github.com/erecarte/blog-spring-cloud-config-configuration.git).
 

#### Advantages

- Only one application communicates with Git, which depending on your Architecture, might be a security advantage.
- Your applications just need to include a thin client which talks over HTTP.

#### Disadvantages

- Not as resilient. If your server application goes down, the client applications will not be able to start up, or will start up without configuration, having to rely
on default values. You can configure whether your application should start up or not after this failure with the property `spring.cloud.config.fail-fast` which
defaults to `false`.
- The server becomes another application to be deployed and maintained.

### Embedded Config Server Setup
Now let's look at the other setup. Instead of having a different application running the server, we could just embed the server inside our client applications. The 
following diagram illustrates what this would look like:

![Standalone Config Setup]({{ site.baseurl }}/img/scc-embedded-setup.png)

You can see that the diagram is slightly simpler. The client applications don't use the Spring Cloud Config Client anymore because they don't need to talk through
HTTP to the server, they have the server inside the application instead.

#### Implementation Details
These are the setps you'll need to follow to embed the server:

1. Your client applications will need to import the Spring Cloud Config Server maven dependency:
```xml
   <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-config-server</artifactId>
   </dependency>
```

2. Configure your client applications with the appropriate Git repositories in the `bootstrap.properties`:
```properties
   #The name of the application
   spring.application.name: StandaloneGitFirstClient
   # The URI of the Git Repository where the configuration is stored
   spring.cloud.config.server.git.uri: https://github.com/erecarte/blog-spring-cloud-config-configuration.git
   # Whether the Spring Cloud Config Server should configure iteslf with the loaded configuration.
   spring.cloud.config.server.bootstrap: true
```
Again, notice how now this configuration goes into the `bootstrap.properties` and not the `application.properties`. Notice
also how we had to define the property `spring.cloud.config.server.bootstrap=true` to force the library to bootstrap itself. This is because now we need the 
embedded Spring Cloud Config Server to configure itself from the loaded configuration in the bootstrap phase.

3. Annotate your main application classes with `@EnableConfigServer`:
 ```java
   @SpringBootApplication
   @EnableConfigServer
   public class EmbeddedGitFirstClientApplication {
      public static void main(String[] args) {
         SpringApplication.run(EmbeddedGitFirstClientApplication.class);
      }
   }
 ```

#### Advantages

- Simplified infrastructure.
- The application is more resilient. You don't depend on the intermediate server being up, and also the fact that the server library clones
the repository on to the disk means it can use the local version if the pull fails on startup. If the Git server is down, your application will
start up with the last pulled configuration, reducing the chances of the configuration being wrong and having to use defaults.

#### Disadvantages

- The first application startup will be slower since it needs to checkout the full Git repository. This could be significant if you have one configuration
repository for your full company and you have hundreds or thousands of services. For this reason, I'd probably recommend have configuration repositories
per team or area in your organisation.
- All your applications will communicate with the Git server. This is specially problematic if you use the same Git server for your application code
and your configuration code, since it widens the security threat.

# Detecting Configuration Changes
We already saw a quick overview of how you can refresh your application's context in the previous post, but in this section we'll take a look at an
example with a real implementation of how this could be done.

Spring Cloud provides a solution to detecting configuration changes in your applications based on a couple of libraries:

- `spring-cloud-config-monitor`: This library adds an endpoint to your application which understands webhooks from a few different Git server providers
such as Github or Bitbucket. If you include this dependency, it will add a `/monitor` endpoint. You then just need to configure the webhook
in the Git repository to point wherever this endpoint is deployed. Once this endpoint is called it will parse the payload to understand what has changed, 
and it will then use the next library to tell the client applications that there is a change in configuration, so they should refresh the application
context. You can find out more information about this library [in this blog post](https://spencergibb.netlify.com/blog/2015/09/24/spring-cloud-config-push-notifications/).
- `spring-cloud-bus`: This is a simple library that Spring provides to communicate global events to other applications through some message brokers. 
It's mostly used for this use case, where you need to send an `RefreshRemoteApplicationEvent` event to all/some applications telling them that they should refresh 
their application context. It integrates with either [RabbitMQ](https://www.rabbitmq.com/) or [Kafka](https://kafka.apache.org/) as the message brokers.

Let's take a look at an architectural overview of the solution:

![Standalone Config Setup With Refresh]({{ site.baseurl }}/img/scc-standalone-setup-with-refresh.png)

There are a few changes from the previous diagrams we were looking at:

- The Server Application now has the Spring Cloud Config Monitor component which will listen for requests to `POST /monitor`.
- There is a RabbitMQ broker now.
- The Client Application also integrates with Spring Cloud Bus, which adds a `RefreshListener` which will listen for `RefreshRemoteApplicationEvent`. If this
event happens, it will trigger a context refresh the same way the `RefreshEndpoint` does.

In this example, I've only considered the standalone setup discussed earlier. For the embedded setup, you would have to have a custom solution since
there is no centralized server that will manage all the configuration to which you can add the `/monitor` endpoint.

# Conclusion
In this post we have taken an in-depth look at how you can use Git to store the configuration for your applications and use Spring Cloud Config
to connect to Git and manage the configuration for you.

We've also seen some different options when designing your solution from an infrastructure point of view, as well as one implementation for automatically
updating the application when there is a change in the configuration.

Having gone through Git, the next post will look at a different backend where we can manage our configuration: [Zookeeper](https://zookeeper.apache.org/).
