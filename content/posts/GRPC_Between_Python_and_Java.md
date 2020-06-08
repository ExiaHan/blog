---
title: "GRPC Between Python and Java"
date: 2020-06-04T03:38:36+08:00
modified: 2020/06/04
tags: ['GRPC', 'Python', 'Java']
keywords: ['GRPC', 'Python', 'Java']
category: ['Technology']
---

Recently I am trying to integrate [flowdroid](URL) into my static android application vulnerabilities scanner which is written in python with [androguard](URL) used to enhance its ability of vul detection.

At first I tried [JPype](URL), it works well in my own machine and docker image, but it can not be used with k8s now because the pip package celery use **fork** to create child process which will hang JPype, for details can check out [here](https://github.com/jpype-project/jpype/issues/358). And after seek for how to change celery from using **fork** to **spawn**. I got that currently celery does not support that option from [here](https://github.com/celery/celery/issues/6036) due to the version of celery my teammate used.

So finally I decide to let flowdroid runs in a single process as a server, and exposes some interface for remote to communicate with. We all know that there are many methods, but here I will try to use [gRPC](https://github.com/grpc/grpc) cause I've never tried it so I think I need to have a try. Also finally it works well. Here I will share you some details about how I do it.

Another reason that I wrote this is that to docs in gRPC's official site is a little confusing for me, I think there will be other guys have the same feel as me, so I wrote my experience here as notes and hope can help others who are in panic.

## Design Your Data Format

For my project I need two interfaces, one for init flowdroid with specified APK file, one for run my wrappers with extra data that I gather from androguard. So my protobuf format like below:

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "com.exiahan.flowdroidrpc";
option java_outer_classname = "FlowdroidRPCProto";
option objc_class_prefix = "FLD";

package flowdroidrpc;

// The service definition.
service FlowdroidRPC {
  // Init
  rpc InitEnv (FlowdroidInitRequest) returns (FlowdroidInitReply) {}
  // Real Process with extra data
  rpc Scan (FlowdroidScanRequest) returns (FlowdroidScanReply) {}

}

// The request message containing args to initialize flowdroid object.
message FlowdroidInitRequest {
  string field_1 = 1;
  string field_2 = 2;
  string field_3 = 3;
}

// The response message containing the result state
message FlowdroidInitReply {
    int32 result = 1;
}

// The request message containing args to indicate which analysis to be performed
message FlowdroidScanRequest {
    string field_1 = 1;
    repeated string field_2 = 2;
    string field_3 = 3;
    string field_4 = 4;
}

// The response message containing the result state
message FlowdroidScanReply {
    int32 result = 1;
}
```

## Java Part with Protobuf

With a pom which contains below contents:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.exiahan.modified</groupId>
  <artifactId>flowdroid-rpc</artifactId>
  <version>1.0</version>

  <name>flowdroid-rpc</name>
  <!-- FIXME change it to the project's website -->
  <url>http://blog.exiahan.com</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <grpc.version>1.29.0</grpc.version>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-bom</artifactId>
        <version>${grpc.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>io.grpc</groupId>
      <artifactId>grpc-netty-shaded</artifactId>
      <version>1.29.0</version>
    </dependency>
    <dependency>
      <groupId>io.grpc</groupId>
      <artifactId>grpc-protobuf</artifactId>
      <version>1.29.0</version>
    </dependency>
    <dependency>
      <groupId>io.grpc</groupId>
      <artifactId>grpc-stub</artifactId>
      <version>1.29.0</version>
    </dependency>
    <dependency> <!-- necessary for Java 9+ -->
      <groupId>org.apache.tomcat</groupId>
      <artifactId>annotations-api</artifactId>
      <version>6.0.53</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>com.exiahan.modified.flowdroid</groupId>
      <artifactId>flowdroidrpc</artifactId>
      <version>1.0</version>
    </dependency>
  </dependencies>

  <build>
    <extensions>
      <extension>
        <groupId>kr.motd.maven</groupId>
        <artifactId>os-maven-plugin</artifactId>
        <version>1.6.2</version>
      </extension>
    </extensions>
    <plugins>
      <plugin>
        <groupId>org.xolstice.maven.plugins</groupId>
        <artifactId>protobuf-maven-plugin</artifactId>
        <version>0.6.1</version>
        <configuration>
          <protocArtifact>com.google.protobuf:protoc:3.11.0:exe:${os.detected.classifier}</protocArtifact>
          <pluginId>grpc-java</pluginId>
          <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.29.0:exe:${os.detected.classifier}</pluginArtifact>
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
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-assembly-plugin</artifactId>
        <configuration>
          <finalName>flowdroid_rpc</finalName>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
          <archive>
            <manifest>
              <addClasspath>true</addClasspath>
              <mainClass>com.exiahan.flowdroid.RPCServer</mainClass>
            </manifest>
          </archive>
        </configuration>
        <executions>
          <execution>
            <id>simple-command</id>
            <phase>package</phase>
            <goals>
              <goal>attached</goal>
            </goals>
          </execution>
        </executions>
        </plugin>
      <plugin>
        <artifactId>maven-clean-plugin</artifactId>
        <version>3.1.0</version>
      </plugin>
      <plugin>
        <artifactId>maven-resources-plugin</artifactId>
        <version>3.0.2</version>
      </plugin>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.8.0</version>
      </plugin>
      <plugin>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>2.22.1</version>
      </plugin>
      <plugin>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.0.2</version>
      </plugin>
      <plugin>
        <artifactId>maven-install-plugin</artifactId>
        <version>2.5.2</version>
      </plugin>
      <plugin>
        <artifactId>maven-deploy-plugin</artifactId>
        <version>2.8.2</version>
      </plugin>
      <plugin>
        <artifactId>maven-site-plugin</artifactId>
        <version>3.7.1</version>
      </plugin>
      <plugin>
        <artifactId>maven-project-info-reports-plugin</artifactId>
        <version>3.0.0</version>
      </plugin>
    </plugins>
  </build>
</project>
```

Then build it with jars your project will need. It will generate the RPC_Related code.

```bash
mvn install
```

The generated code like below:

![generated_protobuf_java](/images/GRPCs/generated_protobuf_java.jpg)

Then implement the **ImplBase** class as the official tutorial says.

```java
import com.exiahan.flowdroidrpc.flowdroidrpc.FlowdroidRPCGrpc;
import com.exiahan.flowdroidrpc.flowdroidrpc.FlowdroidInitRequest;
import com.exiahan.flowdroidrpc.flowdroidrpc.FlowdroidInitReply;
import com.exiahan.flowdroidrpc.flowdroidrpc.FlowdroidScanRequest;
import com.exiahan.flowdroidrpc.flowdroidrpc.FlowdroidScanReply;
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.stub.StreamObserver;
import java.io.IOException;
import java.net.URL;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.TimeUnit;
import java.util.logging.Level;
import java.util.logging.Logger;
import com.exiahan.modified.flowdroid.Wrapper

public class FlowdroidServer {
    private static final Logger logger = Logger.getLogger(FlowdroidService.class.getName());
    private final int port;
    private final Server server;

    public FlowdroidServer(int port) {
        this(port, "FlowdroidServer");
    }

    public FlowdroidServer(int port, String serverName)  {
        this(ServerBuilder.forPort(port), port, serverName);
    }

    public FlowdroidServer(ServerBuilder<?> serverBuilder, int port, String serverName) {
        this.port = port;
        server = serverBuilder.addService(new FlowdroidService(serverName)).build();
    }

    /** Start serving requests. */
    public void start() throws IOException {
        server.start();
        logger.info("Server started, listening on " + port);
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                // Use stderr here since the logger may have been reset by its JVM shutdown
                // hook.
                System.err.println("*** shutting down gRPC server since JVM is shutting down");
                try {
                    FlowdroidServer.this.stop();
                } catch (InterruptedException e) {
                    e.printStackTrace(System.err);
                }
                System.err.println("*** server shut down");
            }
        });
    }

    /** Stop serving requests and shutdown resources. */
    public void stop() throws InterruptedException {
        if (server != null) {
            server.shutdown().awaitTermination(30, TimeUnit.SECONDS);
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

    /**
     * Main method. This comment makes the linter happy.
     */
    public static void main(String[] args) throws Exception {
        FlowdroidServer server = new FlowdroidServer(10085);
        server.start();
        server.blockUntilShutdown();
    }

    private static class FlowdroidService extends FlowdroidRPCGrpc.FlowdroidRPCImplBase {
        private final String serverName;

        private Wrapper Wrapper = null;

        public FlowdroidService(String serverName) {
            logger.info("FlowdroidService Init");
            if (serverName == null) {
                serverName = "FlowdroidService";
            }
            this.serverName = serverName;
        }

        public String getServerName() {
            return this.serverName;
        }

        @Override
        public void initEnv(FlowdroidInitRequest request, StreamObserver<FlowdroidInitReply> responseObserver) {
            logger.info("Init Environment Begin");
            this.Wrapper = new Wrapper(request.getField_1(), request.getField_2(),
                    request.getField_3());
            logger.info("Init Environment Done");
            FlowdroidInitReply flowdroidInitReply = FlowdroidInitReply.newBuilder().setResult(1).build();
            responseObserver.onNext(flowdroidInitReply);
            responseObserver.onCompleted();
        }

        @Override
        public void Scan(FlowdroidScanRequest request, StreamObserver<FlowdroidScanReply> responseObserver) {
            logger.info("Scan Begin");
            FlowdroidScanReply flowdroidScanReply = null;
            try {
                this.Wrapper.runVariableTaint(request.getField_1(),
                        (List<String>) request.getField_2(), request.getTaintValue(),
                        request.getField_3(), request.getField_4(),);
                logger.info("Scan Complete");
                flowdroidScanReply = FlowdroidScanReply.newBuilder().setResult(1).build();
            } catch (Exception e) {
                flowdroidScanReply = FlowdroidScanReply.newBuilder().setResult(0).build();
            }
            responseObserver.onNext(flowdroidScanReply);
            responseObserver.onCompleted();
        }
    }
```

Then use maven to build it again, we will get an jar that can be run via:

```bash
java -jar [your jar file]
```

## Python Part

Python is a little like java. Here we use below shell command to generate codes.

```bash
pip install grpcio
python -m grpc_tools.protoc -I[PATH TO YOUR .PROTO FOLDER] --python_out=. --grpc_python_out=. [PATH TO YOUR TARGET .PROTO File]
```

The generated file like below:

![generated_protobuf_python](/images/GRPCs/generated_protobuf_python.jpg)

The **FlowdroidRPC.py** is the wrapper written by me and the other two py file that end with **pb2\_grpc.py** is generated file.

The wrapper is very easy. As below shows:

```python
from __future__ import print_function

import grpc

import flowdroid_pb2
import flowdroid_pb2_grpc


def rpc_init(logger, field_1, field_2, field_3):
    logger.info("Init flowdroid rpc server")
    channel = grpc.insecure_channel('localhost:10086')
    stub = flowdroid_pb2_grpc.FlowdroidRPCStub(channel)
    init_request = flowdroid_pb2.FlowdroidInitRequest(
        field_1=field_1, field_2=field_2, field_3=field_3)
    response = stub.InitEnv(init_request)
    if response.result == 1:
        logger.info("InitEnv Done")
    else:
        logger.info("InitEnv Error")
    channel.close()

def rpc_scan(logger, field_1, field_2, field_3, field_4):
    logger.info("Begin to scan")
    channel = grpc.insecure_channel('localhost:10085')
    stub = flowdroid_pb2_grpc.FlowdroidRPCStub(channel)
    scan_request = flowdroid_pb2.FlowdroidScanRequest(field_1=field_1, field_2=field_2,
                                                      filed_3=field_3, field_4=field_4)
    response = stub.Scan(scan_request)
    if response.result == 1:
        logger.info("Scan Done")
    else:
        logger.info("Scan Error")
    channel.close()
```

Now all the python and java part is done.
Enjoy it.
