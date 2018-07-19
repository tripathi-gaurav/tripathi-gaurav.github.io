---
layout: post
title:  "Securing gRPC communications with TLS"
date:   2018-06-20 23:27:07 -0400
categories: technology security
---

`gRPC` is an Open Source piece of technology that comes out from the Google factory. It enables a user to create a `.proto` specification file, and generate basic client and server stubs in as many as 9 languages.

On the security posturing of this technology, `gRPC` comes with inbuilt TLS capabilities and does not transfer credentials without encryption over the wire.

I decided to take a stab at establishing a secure RPC communication line between a `gRPC-Java` server and an `Android` client. While, creating a normal Java Server/Client can be done by following the Guides/Tutorials on [grpc.io](https://grpc.io), it gets quite tricky while generating an `Android` as one has to deal with the quirks of mobile development. The `gRPC` team has done good job at laying down the necessary libraries and best practices on some GitHub modules for development on the `Android` front. However, some of the documents were out of date as is expected with the high paces mobile development landscape.

# Step-0: Creating an unencrypted gRPC server
1. If you are using Maven and Eclipse, you will need to download the jar `maven-os-plugin` as mentioned [here](https://github.com/trustin/os-maven-plugin#issues-with-eclipse-m2e-or-other-ides) and place it in `<ECLIPSE_HOME>/plugins` folder.
NetBeans also needs a similar exercise, whereas IntelliJ does not have any such constraint.

2. Create a new Maven Project in your IDE and copy add the following in your `POM.xml` file as mentioned [here](https://github.com/grpc/grpc-java).
These compile rules helps us generate code to be used for RPC.

```xml
<build>
  <extensions>
    <extension>
      <groupId>kr.motd.maven</groupId>
      <artifactId>os-maven-plugin</artifactId>
      <version>1.5.0.Final</version>
    </extension>
  </extensions>
  <plugins>
    <plugin>
      <groupId>org.xolstice.maven.plugins</groupId>
      <artifactId>protobuf-maven-plugin</artifactId>
      <version>0.5.1</version>
      <configuration>
        <protocArtifact>com.google.protobuf:protoc:3.5.1-1:exe:${os.detected.classifier}</protocArtifact>
        <pluginId>grpc-java</pluginId>
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.13.1:exe:${os.detected.classifier}</pluginArtifact>
      </configuration>
      <executions>
        <execution>
          <goals>
            <goal>compile</goal>
            <goal>compile-custom</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```
3. Apart from the above snippet, you will have to add `dependencies` in your code for `gRPC` and Google `protobuf`. The final XML file can be found in my GitHub project [here](https://github.com/tripathi-gaurav/gRPC-demo-server/blob/master/pom.xml).

## Creating/Generating the Server side code

1. Create a `.proto` file in the root directory, with package option set to map against the one we want
`option java_package = "org.tripathi.grpc";`

2. Once the stubs ares created in the `target` directory, we can start using them to write our server end. Server end will require:

  A. Implement the `GreeterImplBase` class to override the `SayHello` and `SayHelloAgain` methods.

  static class GreeterImpl extends GreeterGrpc.GreeterImplBase {

    ```java
    @Override
    public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
      HelloReply reply = HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
      responseObserver.onNext(reply);
      responseObserver.onCompleted();
    }

    @Override
    public void sayHelloAgain(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
      HelloReply reply = HelloReply.newBuilder().setMessage("Hello again bro, " + req.getName()).build();
      responseObserver.onNext(reply);
      responseObserver.onCompleted();
    }
  }```

  B. Create a server stub to listen and respond to RPC calls
    1. Specify the address and port we want to use to listen for client requests using the builder’s `forPort()` method.
    2. Create an instance of our service implementation class RouteGuideService and pass it to the builder’s `addService()` method.
    3. Call `build()` and `start()` on the builder to create and start an RPC server for our service.

# Step-1: Creating the Android client

I started out by editing the `Android` [helloworld](https://github.com/grpc/grpc-java/tree/master/examples/android/helloworld) example as the server stub was mostly modelled around it.

The first thing is to replace the `.proto` file with our `proto` file and sync the project. This will update the `GreeterImpl` and other stubs that will need to be correctly imported in the `HelloWorldActivity`.

The second step is to add the `Google Services Dynamic Security Provider` as mentioned on [here](https://github.com/grpc/grpc-java/blob/master/SECURITY.md). This consumed the most amount of time as the required `Play Services` created cross dependencies with `appCompat-v7:27.0.1` that was already in use in the project. Eventually, the fix was to downgrade `appCompat`, `compileSdkVersion` and `targetSdkVerion` to 26 equivalent.


# Step-2: Adding TLS capability to Server

```XML
<dependencies>
  <dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-tcnative-boringssl-static</artifactId>
    <version>2.0.7.Final</version>
  </dependency>
</dependencies>
```
