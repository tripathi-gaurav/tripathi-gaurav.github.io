---
layout: post
title:  "Securing interoperable gRPC communications with TLS between Python Client and Java server"
date:   2018-06-20 23:27:07 -0400
categories: technology security
---

`gRPC` is an Open Source piece of technology that comes out from the Google factory. It enables a user to use a `.proto` specification file to generate basic client and server stubs in as many as nine languages. `gRPC` works strictly with HTTP2 alone and also comes with inbuilt TLS capabilities. `gRPC` also says that it does not transfer credentials without encryption over the wire.

I decided to take a stab at establishing a secure RPC communication line between a [`gRPC-Java` server](https://github.com/tripathi-gaurav/gRPC-demo-server) and an [`Python` client](https://github.com/tripathi-gaurav/gRPC-demo-client-py).

In this post and related GitHub repositories, you should be able to run the client and server, without having to go and study the `Protobuf` and `gRPC` documentation. The post requires understand of `Maven` package management tool to clean and build dependencies.

Also, I was able to develop and `Android` client to the `gRPC-java` server and communicate between them without `TLS` encryption. So, you can choose to skip Step-1 and Step-2 after completing Step-0.

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

1. Create a `.proto` file in the root directory, with package option set to map against the one we want to use, in my case `org.tripathi.grpc.hellodroidtls`. This document defines two methods, `SayHello` and `SayHelloAgain`. We only use `SayHello` method in our calls. Note that I have used the package name `hellodroidtls`, as I initially intended to write an `Android` client instead of `Python` client.

```java
syntax = "proto3";

option java_package = "org.tripathi.grpc.hellodroidtls";
option java_multiple_files = true;
option java_outer_classname = "HelloDroidTLSProto";

package hellodroidtls;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
  rpc SayHelloAgain (HelloRequest) returns (HelloReply){}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

2. Once the stubs are created in the `target` directory, we can start using them to write our server end. Server end will require:

  A. Implement the `GreeterImplBase` class to override the `SayHello` and `SayHelloAgain` methods.


```java
static class GreeterImpl extends GreeterGrpc.GreeterImplBase {
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
  }
```
Final file can [referenced here](https://github.com/tripathi-gaurav/gRPC-demo-server/blob/master/src/main/java/org/tripathi/grpc/hellodroidtls/HelloDroidTLSServer.java).

  B. Create a server stub to listen and respond to RPC calls

  From [gRPC Basic - Java docs](https://grpc.io/docs/tutorials/basic/java.html):

  1. Specify the address and port we want to use to listen for client requests using the builder’s `forPort()` method.
  2. Create an instance of our service implementation class RouteGuideService and pass it to the builder’s `addService()` method.
  3. Call `build()` and `start()` on the builder to create and start an RPC server for our service.

  The final RPC code will be as follows:

  ```java
  /**
	 * == Code to drive the server ==
	 */
	private void start() throws IOException {
		/* The port on which the server should run */
		int port = 40443;
		server = ServerBuilder.forPort(port).addService(new GreeterImpl()).build().start();
		logger.info("Server started, listening on " + port);
		Runtime.getRuntime().addShutdownHook(new Thread() {
			@Override
			public void run() {
				// Use stderr here since the logger may have been reset by its JVM shutdown
				// hook.
				System.err.println("*** shutting down gRPC server since JVM is shutting down");
				HelloDroidTLSServer.this.stop();
				System.err.println("*** server shut down");
			}
		});
	}

	private void stop() {
		if (server != null) {
			server.shutdown();
		}
	}

	/**
	 * Await termination on the main thread since the grpc library uses daemon
	 * threads.
	 */
	private void blockUntilShutdown() throws InterruptedException {
		if (server != null) {
			server.awaitTermination();
		}
	}
/** == End of Code to drive the server == */
  ```

Finally, you just need to add the `main` method to start your server:
```java
public static void main(String[] args) throws IOException, InterruptedException {
  final HelloDroidTLSServer server = new HelloDroidTLSServer();
  server.start();
  server.blockUntilShutdown();
}
```


# Step-1: Adding TLS capability to Server

Since, we need CA certificates for TLS/SSL based encryption, we will generate self-signed certificates and add it to CA certs for the server. A simple script to do so can be found [here](https://github.com/grpc/grpc-java/tree/master/examples#generating-self-signed-certificates-for-use-with-grpc). The recommended approach for TLS with `gRPC` is to use TLS with `OpenSSL` using `netty-tcnative-boringssl-static`. Therefore, we need to add the following dependency to our POM.xml file:

  ```XML
  <dependencies>
    <dependency>
      <groupId>io.netty</groupId>
      <artifactId>netty-tcnative-boringssl-static</artifactId>
      <version>2.0.7.Final</version>
    </dependency>
  </dependencies>
  ```

We now need to modify the server startup code slightly to enable TLS.
Final file can [referenced here](https://github.com/tripathi-gaurav/gRPC-demo-server/blob/master/src/main/java/org/tripathi/grpc/hellodroidtls/HelloDroidTLSServer.java).

```java
String certChainFile = "/path/to/server.crt";
String privateKeyFile = "path/to/server.pem";
int port = 8443;
server = ServerBuilder.forPort(port)
      // Enable TLS
      .useTransportSecurity(new File(certChainFile), new File(privateKeyFile))
      .addService(new GreeterImpl())
      .build();
server.start();

```

# Step-2: Creating the Python client

Using the `.proto` file, we will create Python stubs with `gRPC`.

```shell
$ pip install grpcio-tools
$ pip install googleapis-common-protos
$ python -m grpc_tools.protoc -I../path/to/proto_folder/ --python_out=. --grpc_python_out=. /path/to/proto_folder/proto_file.proto
```

With our stubs in place we can go on to create our client:

```Python
import grpc
import HelloDroidTLS_pb2
import HelloDroidTLS_pb2_grpc

with open('/path/to/server.crt', 'rb') as f:
    trusted_certs = f.read()


creds = grpc.ssl_channel_credentials(root_certificates=trusted_certs)
channel = grpc.secure_channel('localhost:8443', creds)
stub = HelloDroidTLS_pb2_grpc.GreeterStub(channel)
response = stub.SayHello(HelloDroidTLS_pb2.HelloRequest(name='your_name'))
print("Greeter client received: " + response.message)
```

Output:
```shell
$ python grpc_client_py.py
Greeter client received: Hello yourname
```

# Future work

While, creating a normal Java Server/Client can be done by following the Guides/Tutorials on [grpc.io](https://grpc.io), it got quite tricky when I attempted to make an interoperable Java server with an `Android` client, as one has to deal with the quirks of mobile development. The `gRPC` team has done good a job at laying down the necessary libraries and best practices on GitHub regarding the modules for development on the `Android` front. However, some of the documents were out of date, as is expected with the high pace mobile development landscape.

I started out by editing the `Android` [helloworld](https://github.com/grpc/grpc-java/tree/master/examples/android/helloworld) example as the server stub was mostly modelled around it.

The first thing is to replace the `.proto` file with our `proto` file and sync the project. This will update the `GreeterImpl` and other stubs that will need to be correctly imported in the `HelloWorldActivity`.

The second step is to add the `Google Services Dynamic Security Provider` as mentioned on [here](https://github.com/grpc/grpc-java/blob/master/SECURITY.md). This consumed the most amount of time as the required `Play Services` created cross dependencies with `appCompat-v7:27.0.1` that was already in use in the project. Eventually, the fix was to downgrade `appCompat`, `compileSdkVersion` and `targetSdkVerion` to 26 equivalent.
