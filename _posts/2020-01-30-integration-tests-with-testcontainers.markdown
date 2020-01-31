---
layout: post
title:  "Integration tests with Testcontainers"
date:   2019-10-09 10:24:10 +0100
categories: java test
---
Recently, I had to write integration tests for one of my project which involved testing the behavior of my API with Elasticsearch and RabbitMQ.

Implementing integration with a database is fairly easy nowadays, with solutions such as H2, providing in-memory databases to work with, as long as you stay in relatively common features of SQL databases.

But when it comes to Elasticsearch or RabbitMQ, things are getting a little bit more complex. And a fairly good candidate to solve this problem is [Testcontainers][testcontainers.org].

The idea is that your code will start a Docker container, run the test, and then stop the container and remove it.

## An example

My project is built with Spring Boot, the integration tests are testing a regular Spring MVC REST API.

First, you will want to add some dependencies to your pom.xml.
```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.12.5</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>1.12.5</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>rabbitmq</artifactId>
    <version>1.12.5</version>
    <scope>test</scope>
</dependency>
```

You may also find useful to add some logging for Testcontainers, just to see what's happening, and remove that later.
```xml
<logger name="org.testcontainers" level="TRACE"/>
```

Now, getting to the test itself, I started with an `AbstractTestClass` that I would extend in my concrete test classes.
The responsibility of this abstract class is to start the Docker container(s) and get properties from those containers and inject them in the application properties of Spring.
```java
package com.anthonyrichir.example.api;

import org.springframework.boot.test.util.TestPropertyValues;
import org.springframework.context.ApplicationContextInitializer;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.test.context.ContextConfiguration;
import org.testcontainers.containers.RabbitMQContainer;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.lifecycle.Startables;

import java.util.stream.Stream;

@ContextConfiguration( // - 1
    initializers = AbstractTestClass.Initializer.class
)
public class AbstractTestClass {

    public static class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> { // - 2

        static ElasticsearchContainer elasticsearch = 
                new ElasticsearchContainer("docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.5"); // - 3

        static RabbitMQContainer rabbit = new RabbitMQContainer("rabbitmq:alpine");

        @Override
        public void initialize(ConfigurableApplicationContext applicationContext) {
            Startables.deepStart(Stream.of(elasticsearch, rabbit)).join(); // - 4

            TestPropertyValues.of(  // - 5
                "spring.data.jest.uri=" + "http://" + elasticsearch.getHttpHostAddress(),
                "spring.rabbitmq.host=" + rabbit.getContainerIpAddress(),
                "spring.rabbitmq.username=" + rabbit.getAdminUsername(),
                "spring.rabbitmq.password=" + rabbit.getAdminPassword(),
                "spring.rabbitmq.port=" + rabbit.getAmqpPort()
            ).applyTo(applicationContext);
        }
    }
}
```

1. I declare a ContextContext that will use a static inner class defined below as initializer.
2. The `Initializer` class implements the `ApplicationContextInitializer` callback interface, used for web applications that require some programmatic initialization of the application context.
3. The Elasticsearch container, defined as a static field of the `Initializer`, instantiated via the constructor, using the Docker image name.
4. I could have simply called the `start()` method of each container individually, but there is this `deepStart(...)` method provided by Testcontainers which allows to start recursively and asynchronously the containers passed.
5. Very interesting part, `TestPropertyValues` provides methods to easily pass application properties to the Spring context, used here with properties from the containers.

This is all you need to run your test with testcontainers, running without having to write mocks.

## Running tests

Here below is an extract from the logs of one of my test classes.
You can see that the first thing that Testcontainers will start is `Ryuk`, refering the fictional character of the manga *Death Note*. It's used to removed the containers created by the framework.

Next step is, of course, starting the containers defined in the test.

```
2020-01-30 17:03:00.863 DEBUG   --- [           main] o.t.utility.TestcontainersConfiguration  : Testcontainers configuration overrides will be loaded from file:/home/arichir/.testcontainers.properties
2020-01-30 17:03:00.911  INFO   --- [           main] o.t.d.DockerClientProviderStrategy       : Loaded org.testcontainers.dockerclient.EnvironmentAndSystemPropertyClientProviderStrategy from ~/.testcontainers.properties, will try it first
2020-01-30 17:03:01.229 DEBUG   --- [     ducttape-0] o.t.d.DockerClientProviderStrategy       : Pinging docker daemon...
2020-01-30 17:03:01.590  INFO   --- [           main] tAndSystemPropertyClientProviderStrategy : Found docker client settings from environment
2020-01-30 17:03:01.590  INFO   --- [           main] o.t.d.DockerClientProviderStrategy       : Found Docker environment with Environment variables, system properties and defaults. Resolved dockerHost=unix:///var/run/docker.sock
2020-01-30 17:03:01.590 DEBUG   --- [           main] o.t.d.DockerClientProviderStrategy       : Checking Docker OS type for Environment variables, system properties and defaults. Resolved dockerHost=unix:///var/run/docker.sock
2020-01-30 17:03:01.735  INFO   --- [           main] org.testcontainers.DockerClientFactory   : Docker host IP address is localhost
2020-01-30 17:03:01.763  INFO   --- [           main] org.testcontainers.DockerClientFactory   : Connected to docker: 
  Server Version: 19.03.3
  API Version: 1.40
  Operating System: Pop!_OS 19.10
  Total Memory: 31739 MB
2020-01-30 17:03:01.763 DEBUG   --- [           main] org.testcontainers.DockerClientFactory   : Ryuk is enabled
2020-01-30 17:03:01.804 DEBUG   --- [           main] o.t.utility.RegistryAuthLocator          : Looking up auth config for image: quay.io/testcontainers/ryuk:0.2.3
2020-01-30 17:03:01.805 DEBUG   --- [           main] o.t.utility.RegistryAuthLocator          : RegistryAuthLocator has configFile: /home/arichir/.docker/config.json (exists) and commandPathPrefix: 
2020-01-30 17:03:01.812 DEBUG   --- [           main] o.t.utility.RegistryAuthLocator          : registryName [quay.io] for dockerImageName [quay.io/testcontainers/ryuk:0.2.3]
2020-01-30 17:03:01.812 DEBUG   --- [           main] o.t.utility.RegistryAuthLocator          : no matching Auth Configs - falling back to defaultAuthConfig [null]
2020-01-30 17:03:01.812 DEBUG   --- [           main] o.t.d.a.AuthDelegatingDockerClientConfig : Effective auth config [null]
2020-01-30 17:03:06.877 TRACE   --- [           main] org.testcontainers.utility.AuditLogger   : START action with image: null, containerId: a4fa05a96f8be42120f6725fafe5794c6c48804f27c6dab916754e9b7eb850c4
2020-01-30 17:03:06.940 DEBUG   --- [containers-ryuk] o.testcontainers.utility.ResourceReaper  : Sending 'label=org.testcontainers%3Dtrue&label=org.testcontainers.sessionId%3D8ab6e3b0-900c-4109-aa06-c75e8f4ff4f5' to Ryuk
2020-01-30 17:03:06.940 DEBUG   --- [containers-ryuk] o.testcontainers.utility.ResourceReaper  : Received 'ACK' from Ryuk
2020-01-30 17:03:06.940  INFO   --- [           main] org.testcontainers.DockerClientFactory   : Ryuk started - will monitor and terminate Testcontainers containers on JVM exit
2020-01-30 17:03:06.941 DEBUG   --- [           main] org.testcontainers.DockerClientFactory   : Checks are disabled
2020-01-30 17:03:06.957 DEBUG   --- [           main] o.t.images.AbstractImagePullPolicy       : Using locally available and not pulling image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.5
2020-01-30 17:03:08.761 DEBUG 14329 --- [ers-lifecycle-0] o.t.utility.RegistryAuthLocator          : Looking up auth config for image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.5
2020-01-30 17:03:08.762 DEBUG 14329 --- [ers-lifecycle-0] o.t.utility.RegistryAuthLocator          : RegistryAuthLocator has configFile: /home/arichir/.docker/config.json (exists) and commandPathPrefix: 
2020-01-30 17:03:08.762 DEBUG 14329 --- [ers-lifecycle-0] o.t.utility.RegistryAuthLocator          : registryName [docker.elastic.co] for dockerImageName [docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.5]
2020-01-30 17:03:08.762 DEBUG 14329 --- [ers-lifecycle-0] o.t.utility.RegistryAuthLocator          : no matching Auth Configs - falling back to defaultAuthConfig [null]
2020-01-30 17:03:08.762 DEBUG 14329 --- [ers-lifecycle-0] o.t.d.a.AuthDelegatingDockerClientConfig : Effective auth config [null]
2020-01-30 17:03:08.776 TRACE 14329 --- [ers-lifecycle-1] o.t.images.LocalImagesCache              : Image rabbitmq:alpine not found

com.github.dockerjava.api.exception.NotFoundException: {"message":"no such image: rabbitmq:alpine: No such image: rabbitmq:alpine"}

	at org.testcontainers.dockerclient.transport.okhttp.OkHttpInvocationBuilder.execute(OkHttpInvocationBuilder.java:281)
	at org.testcontainers.dockerclient.transport.okhttp.OkHttpInvocationBuilder.execute(OkHttpInvocationBuilder.java:265)
	at org.testcontainers.dockerclient.transport.okhttp.OkHttpInvocationBuilder.get(OkHttpInvocationBuilder.java:231)
	at org.testcontainers.dockerclient.transport.okhttp.OkHttpInvocationBuilder.get(OkHttpInvocationBuilder.java:101)
	at com.github.dockerjava.core.exec.InspectImageCmdExec.execute(InspectImageCmdExec.java:28)
	at com.github.dockerjava.core.exec.InspectImageCmdExec.execute(InspectImageCmdExec.java:13)
	at com.github.dockerjava.core.exec.AbstrSyncDockerCmdExec.exec(AbstrSyncDockerCmdExec.java:21)
	at com.github.dockerjava.core.command.AbstrDockerCmd.exec(AbstrDockerCmd.java:35)
	at com.github.dockerjava.core.command.InspectImageCmdImpl.exec(InspectImageCmdImpl.java:40)
	at org.testcontainers.images.LocalImagesCache.refreshCache(LocalImagesCache.java:42)
	at org.testcontainers.images.AbstractImagePullPolicy.shouldPull(AbstractImagePullPolicy.java:24)
	at org.testcontainers.images.RemoteDockerImage.resolve(RemoteDockerImage.java:62)
	at org.testcontainers.images.RemoteDockerImage.resolve(RemoteDockerImage.java:25)
	at org.testcontainers.utility.LazyFuture.getResolvedValue(LazyFuture.java:20)
	at org.testcontainers.utility.LazyFuture.get(LazyFuture.java:27)
	at org.testcontainers.containers.GenericContainer.getDockerImageName(GenericContainer.java:1263)
	at org.testcontainers.containers.GenericContainer.logger(GenericContainer.java:600)
	at org.testcontainers.containers.GenericContainer.doStart(GenericContainer.java:311)
	at org.testcontainers.containers.GenericContainer.start(GenericContainer.java:302)
	at org.testcontainers.lifecycle.Startables$$Lambda$592.00000000B03932A0.run(Unknown Source)
	at java.base/java.util.concurrent.CompletableFuture$UniRun.tryFire(CompletableFuture.java:783)
	at java.base/java.util.concurrent.CompletableFuture$Completion.run(CompletableFuture.java:478)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:831)

2020-01-30 17:03:08.776 DEBUG 14329 --- [ers-lifecycle-1] o.t.images.AbstractImagePullPolicy       : Not available locally, should pull image: rabbitmq:alpine
2020-01-30 17:03:08.778 DEBUG 14329 --- [ers-lifecycle-1] o.t.utility.RegistryAuthLocator          : Looking up auth config for image: rabbitmq:latest
2020-01-30 17:03:08.778 DEBUG 14329 --- [ers-lifecycle-1] o.t.utility.RegistryAuthLocator          : RegistryAuthLocator has configFile: /home/arichir/.docker/config.json (exists) and commandPathPrefix: 
2020-01-30 17:03:08.779 DEBUG 14329 --- [ers-lifecycle-1] o.t.utility.RegistryAuthLocator          : registryName [index.docker.io] for dockerImageName [rabbitmq:latest]
2020-01-30 17:03:08.779 DEBUG 14329 --- [ers-lifecycle-1] o.t.utility.RegistryAuthLocator          : no matching Auth Configs - falling back to defaultAuthConfig [null]
2020-01-30 17:03:08.779 DEBUG 14329 --- [ers-lifecycle-1] o.t.d.a.AuthDelegatingDockerClientConfig : Effective auth config [null]
2020-01-30 17:03:08.898 TRACE 14329 --- [ers-lifecycle-0] org.testcontainers.utility.AuditLogger   : CREATE action with image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.5, containerId: fa13c1cd9ec094928d622908aee61893a9db5def6d870ff6549f5022615aafb5
2020-01-30 17:03:09.389 TRACE 14329 --- [ers-lifecycle-0] org.testcontainers.utility.AuditLogger   : START action with image: null, containerId: fa13c1cd9ec094928d622908aee61893a9db5def6d870ff6549f5022615aafb5
2020-01-30 17:03:09.400  INFO 14329 --- [ers-lifecycle-0] o.t.c.wait.strategy.HttpWaitStrategy     : /distracted_mendel: Waiting for 120 seconds for URL: http://localhost:32770/
2020-01-30 17:03:15.512 TRACE 14329 --- [     ducttape-0] o.t.c.wait.strategy.HttpWaitStrategy     : Get response code 200
2020-01-30 17:03:21.789 DEBUG 14329 --- [ers-lifecycle-1] o.t.utility.RegistryAuthLocator          : Looking up auth config for image: rabbitmq:alpine
2020-01-30 17:03:21.789 DEBUG 14329 --- [ers-lifecycle-1] o.t.utility.RegistryAuthLocator          : RegistryAuthLocator has configFile: /home/arichir/.docker/config.json (exists) and commandPathPrefix: 
2020-01-30 17:03:21.790 DEBUG 14329 --- [ers-lifecycle-1] o.t.utility.RegistryAuthLocator          : registryName [index.docker.io] for dockerImageName [rabbitmq:alpine]
2020-01-30 17:03:21.790 DEBUG 14329 --- [ers-lifecycle-1] o.t.utility.RegistryAuthLocator          : no matching Auth Configs - falling back to defaultAuthConfig [null]
2020-01-30 17:03:21.790 DEBUG 14329 --- [ers-lifecycle-1] o.t.d.a.AuthDelegatingDockerClientConfig : Effective auth config [null]
2020-01-30 17:03:25.966 TRACE 14329 --- [ers-lifecycle-1] org.testcontainers.utility.AuditLogger   : CREATE action with image: rabbitmq:alpine, containerId: 114b1e7bb31d22adc8e44f67d3163373414c56eecc9631b5f1c32cb645fc3634
2020-01-30 17:03:26.546 TRACE 14329 --- [ers-lifecycle-1] org.testcontainers.utility.AuditLogger   : START action with image: null, containerId: 114b1e7bb31d22adc8e44f67d3163373414c56eecc9631b5f1c32cb645fc3634
```

## Additionnal Configuration
Testcontainers checks for a file in your home folder `~/.testcontainers.properties`, note that the framework modifies the file as well.

One of the properties that you may want to consider is to enable to reuse created containers, and to achieve, you will need 2 things.

First, you need one property in this file.
```
testcontainers.reuse.enable=true
```

Second thing you need is, in your code, specifiy that a container can be reused.
```java
static RabbitMQContainer rabbit = new RabbitMQContainer("rabbitmq:alpine")
        .withReuse(true);
```

You need to have both of these enabled for the reuse to work.
This is to allow to only reuse one container or another, because keeping one of the containers up between test runs might not have an impact on the functionality, and save you a bit of execution time.
But also to prevent the reuse of containers on some environments, on a CI for instance, and allow it only on your dev machine.

[testcontainers.org]: https://www.testcontainers.org/