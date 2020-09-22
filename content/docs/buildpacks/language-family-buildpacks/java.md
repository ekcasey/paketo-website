---
title: "Java Buildpack"
weight: 303
menu:
  main:
    parent: "language-family-buildpacks"
---

# Java Buildpack
The [Paketo Java Buildpack][java] allows users to create an image containing a JVM application or task from a precompiled artifact or directly from source.

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Java Buildpack](#java-buildpack)
  - [About the Examples](#about-the-examples)
  - [Table of Contents](#table-of-contents)
  - [Building an Image](#building-an-image)
    - [From Source](#from-source)
      - [**Example: Building with Maven**](#example-building-with-maven)
    - [From a Precompiled Artifact](#from-a-precompiled-artifact)
      - [Executable JAR](#executable-jar)
      - [WAR](#war)
      - [Distribution ZIP](#distribution-zip)
      - [**Example: Building from an Executable JAR**](#example-building-from-an-executable-jar)
    - [](#)
    - [Configuration](#configuration)
      - [Environment Variables](#environment-variables)
      - [Bindings](#bindings)
      - [Procfiles](#procfiles)
    - [Building Behind a Firewall](#building-behind-a-firewall)
      - [Proxies](#proxies)
      - [Dependency Mappings](#dependency-mappings)
  - [Running the App Image](#running-the-app-image)
    - [Providing Additional Arguments](#providing-additional-arguments)
    - [Executing a Custom Command](#executing-a-custom-command)
    - [Executing a Custom Command in the Buildpack-Provided Environment](#executing-a-custom-command-in-the-buildpack-provided-environment)
    - [Runtime Configuration](#runtime-configuration)
      - [Environment Variables](#environment-variables-1)
      - [Bindings](#bindings-1)
  - [Inspecting the App Image](#inspecting-the-app-image)
    - [Bill of Materials](#bill-of-materials)
    - [Buildpack-Provided Labels](#buildpack-provided-labels)
    - [User-Provided Labels](#user-provided-labels)
  - [Modular Components](#modular-components)

## Building From Source
The Java buildpack can build from Source using any of the following build tools:
* [Gradle][gradle]
* [Leiningen][leiningen]
* [Maven][maven]
* [sbt][sbt]

The correct build tool to use will be detected based on the contents of the applicaiton directory.
 
The build should produce one the of supported artifact formats in the section below. After building, the buildpack will replace provided application source code with the exploded archive. The build will proceed as described [Building from a Precompiled Artifact](#building-from-a-precompiled-artifact).

**Example**: Building with Maven

The following command uses the `pack` CLI, to build with `maven`.

{{< code/copyable >}}
cd samples/java/maven
pack build example/app
{{< /code/copyable >}}

### Build from Source Configuration
**Important**: The following set of configuration options are not comprehensive, see the reference docs for the relavent [component buildpacks](#components) for a full-set of configuration options.

### Selecting a Module or Artifact

For a given build `<TOOL>`, where `<TOOL>` is one of `MAVEN`, `GRADLE`, `LEIN` or `SBT`, the Java Buildpack accepts the following environment variables:
* `BP_<TOOL>_BUILT_MODULE`
    * *Defaults* to the root module.
    * Configures the module in a multi-module build from which the buildpack will select the application artifact.
    * *Example*: Given `BP_MAVEN_BUILT_MODULE=api`, Paketo Maven Buildpack will look for the application artifact with the file pattern `target/api/*.[jw]ar`.
* `BP_<TOOL>_BUILT_ARTIFACT`
    * Defaults to a tool-specific pattern (e.g. `target/*.[jw]ar` for Maven, `build/libs/*.[jw]ar` for gradle). See component buildpack docs for details.
    * Configures the built application artifact path, using [Bash Pattern Matching](https://www.gnu.org/software/bash/manual/html_node/Pattern-Matching.html).
    * Supercedes `BP_<TOOL>_BUILT_MODULE` if set to a non-default value.
    * *Example*: Given`BP_MAVEN_BUILT_ARTIFACT=out/api-*.jar`, the Paketo Maven Buildpack will select a file with name `out/api-1.0.0.jar`.

### Specifying the Build Command

For a given build `<TOOL>`, where `<TOOL>` is one of `MAVEN`, `GRADLE`, `LEIN` or `SBT`, the Java Buildpack accepts the following environment variables:
   * `BP_<TOOL>_BUILD_ARGUMENTS`
   * *Defaults* to a tool-specific value (e.g. `-Dmaven.test.skip=true package` for Maven, `--no-daemon -x test build` for Gradle). See component buildpack docs for details.
   * Configures the arguments to pass to the build tool.
   * *Example*: Given `BP_GRADLE_BUILD_ARGUMENTS=war`, the Paketo Gradle Buildpack will execute `./gradlew war` or `gradle war` (depending on the presence of the gradle wrapper).

### Connecting to a Private Maven Repository

A binding with type `maven` can be used to provide a Maven `settings.xml` file to a build. This `settings.xml` file may contain the credentials needed to connect to a private Maven repostiory.

The Java Buildpack will use the custom Maven Settings if it finds a directory matching the following in `/platform/bindings`: 
```
<binding-name>
├── settings.xml
└── type
```
**Example**: Providing Maven Settings

The following steps demonstrate how to use the `settings.xml` file from your workstation with `pack`.

1. Create a directory to contain the binding.
```
cd samples/maven
mkdir binding
```
2. Indicate that the binding is of type `maven` with a file called `type` inside the binding, containing the value `maven`.
```
echo -n "maven" > ./binding/type
```
3. Copy the `settings.xml` file from our workstation to the binding.
```
cp ~/.m2/settings.xml ./binding/settings.xml
```
4. Provide the binding to `pack build`.
```
cd java/maven
pack build example/app --volume $(pwd)/binding:/platform/bindings/my-maven-settings
```

## Building from a Precompiled Artifact
An application developer may build an image from following archive formats:
* [Executable JAR][executable jar]
* [WAR][war]
* [Distribution ZIP][dist-zip]
 
The Java Buildpack expects the application directory to contain the extracted contents of the archive (e.g. an exploded JAR). Most platforms will automatically extract provided archives.

#### Executable JAR

#### WAR
#### Distribution ZIP

#### **Example: Building from an Executable JAR**

The following command uses `maven` directly to compile an executable JAR and then uses the `pack` CLI to build an image from the JAR.

{{< code/copyable >}}
cd samples/java/maven
./mvnw package
pack build example/app --path ./target/demo-0.0.1-SNAPSHOT.jar --buildpack gcr.io/paketo-buildpacks/java
{{< /code/copyable >}}

The resulting application image will be identical to that built in [Example 1](#example-1-building-from-source-with-maven) with the excluding 

### 


## Configuring the JVM Version
The following are some commonly use

## Debug Images

#### Environment Variables

Users may configure the build by setting variables in the buildpack environment. By convention the names of all variables accepted by the Java Buildpack at build-time are prefixed with `BP_`.

The following example uses an environment variable to configure the JVM version.

```
cd samples/maven
pack build --env BP_JVM_VERSION=8 example/app --buildpack gcr.io/paketo-buildpacks/java
```

During the build process the Java Buildpack may invoke other tools that accept configuration via the environment. Users may continue to use these variables idiomatically.

The following example configures the JVM memory settings for the JVM running Maven.

```
cd samples/maven
pack build --env "MAVEN_OPTS=-Xms256m -Xmx512m" example/app --buildpack gcr.io/paketo-buildpacks/java
```

#### Bindings

The Java Buildpack accepts credentials and other secrets using bindings. Bindings may be used to provide the location and credentials needed to connect to an external service. Common examples of external services used at build time include private artifact repositories and SaaS security scanning tools.

From the perspective of the Java Buildpack a binding is a directory in a discoverable location (`/platform/bindings`), containing the type and, optionally, the provider of the binding along with a set of key value pairs. The java buildpack accepts two different binding specification: the [Service Binding Specification for Kubernetes](https://github.com/buildpacks/spec/blob/main/extensions/bindings.md) or the [Cloud Native Buildpacks Bindings Specification](https://github.com/k8s-service-bindings/spec). The former is prefered as it will eventually replace the latter but some platforms may not yet support the newer specification.

The workflow for creating a binding and providing it to a build will depend on the chosen platform. For example, `pack` users should use the `--volume` flag to mount a binding directory into `/platform/bindings`. Users of the `kpack` platform should store key value pairs in a Kubernetes Secret and provide that secret and associated metadata to an Image as described in the [kpack documentation](https://github.com/pivotal/kpack/blob/master/docs/servicebindings.md#service-bindings).


Example 1: Providing Custom Maven settings.xml

The following example illustrates how to provide a Maven `settings.xml` file to a build. This `settings.xml` file may contain the credentials needed to connect to a private Maven repostiory. At build time we want the Java Buildpack to find the following directory in `/platform/bindings`. 
```
my-maven-settings
├── settings.xml
└── type
```

To do this we can:
1. Create a directory to contain our binding.
```
cd samples/maven
mkdir binding
```
2. Indicate that the binding is of type `maven` with a file called `type` inside the binding, containing the value `maven`.
```
echo -n "maven" > ./binding/type
```
3. Copy the `settings.xml` file from our workstation to the binding.
```
cp ~/.m2/settings.xml ./binding/settings.xml
```
4. Provide the binding to `pack build`.
```
pack build example/app --volume ./binding:/platform/bindings/my-maven-settings
```

The above example used the Service Binding Specification for Kubernetes; an equivalent binding using the CNB Binding Specification would produce identical results:
```
my-maven-settings
└── metadata
|   └── provider
└── secret
    └── settings.xml
```

#### Procfiles

When building [from source](#from-source) users may specify a custom process types by placing a `Procfile` at the root of the repo. A Procfile should have the following schema:

```
<process_name1>: <command1>
<process_name2>: <command2>
```

## Running the App Image

Images created by the Java buildpack can be run just like any OCI image.

Execute the following commands to start a container using the `example/app` image (built with any of the example commands above).
{{< code/copyable >}}
docker run  --rm --publish 8080:8080 example/app
curl -s http://localhost:8080/actuator/health
{{< /code/copyable >}}

### Providing Additional Arguments

Additional arguments can be provided to the application using the container `CMD`.

Execute the following command passes an additional argument to application start command, setting the port to `8081`.
{{< code/copyable >}}
docker run --rm --publish 8081:8081 example/app --server.port=8081
curl -s http://localhost:8081/actuator/health
{{< /code/copyable >}}

In Kubernetes set `CMD` using the `args` field on the [container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#container-v1-core) resource.

### Executing a Custom Command

To override the buildpack-provided start command with a custom command, set the container `ENTRYPOINT`

The following command runs Bash interactively:
{{< code/copyable >}}
docker run --rm --entrypoint bash example/app
{{< /code/copyable >}}

### Executing a Custom Command in the Buildpack-Provided Environment

Every CNB image contains an executable called the `launcher` which can be used to execute a custom command in an environment containing buildpack-provided environment variables. The `launcher` will execute any buildpack provided profile scripts before running to provided command, in order to set environment variables with values that should be calculated dynamically at runtime.
 
To run a custom start command in the buildpack-provided environment set the `ENTRYPOINT` to `launcher` and provide the command using the container `CMD`.

The following command will print value of `$JAVA_TOOL_OPTIONS` set by the buildpack:
{{< code/copyable >}}
docker run --rm --entrypoint launcher example/app echo 'JAVA_TOOL_OPTIONS: $JAVA_TOOL_OPTIONS'
{{< /code/copyable >}}

Each argument provided to the launcher will be evaluated by the shell prior to execution and the original tokenization will be preserved. Note that, in the example above `'JAVA_TOOL_OPTIONS: $JAVA_TOOL_OPTIONS'` is single quoted so that `$JAVA_TOOL_OPTIONS` is evaluated in the container, rather than by the host shell.

### Runtime Configuration

#### Environment Variables

Users may configure runtime features of the app image by setting envinoment variables in the app container environment. By convention, the names of most variables accepted by runtime components installed by the Java Buildpack (e.g. profiles scripts, processes types) are prefixed with `BPL_`. However, environment variables that are well-known conventions in the Java ecosystem such as `JAVA_OPTS` should be provided without a prefix.

The following example uses `JAVA_OPTS` to set the server port of the sample application:
{{< code/copyable >}}
docker run --rm --publish 8082:8082 --env "JAVA_OPTS=-Dserver.port=8082" example/app
curl -s http://localhost:8082/actuator/health
{{< /code/copyable >}}

#### Bindings

Some runtime components installed by the Java Buildpack use bindings to find the location and credentials needed to connect to external sevices at runtime.

For example, given a Spring Boot application, the Java Buildpack will contribute (Spring Cloud Bindings)[https://github.com/spring-cloud/spring-cloud-bindings] to the Class Path. Spring Cloud Bindings can autoconfigure the application to connect to a variety of external services when provided with a binding of a supported `type` at runtime.

Buildpack enabled APM integrations, including Azure Application Insights, and Google Stackdriver also leverage bindings to connect at runtime.

## Inspecting the App Image

### Bill of Materials
The Java Buildpack generates a bill-of-materials (BOM) describing the application and any application dependencies it contributed to the image. The BOM is stored as a label on the image.

It is recommended to use a tool like `jq` to query the BOM.

The following are just a few examples of the type of information that can be pulled from the BOM with `pack` and `jq`.

1. Print name of every BOM entry
{{< code/copyable >}}
pack inspect-image example/app --bom | jq ".local[].name"
{{< /code/copyable >}}

2. Print the version of the JRE
{{< code/copyable >}}
pack inspect-image example/app --bom | jq '.local[] | select(.name=="jre") | .metadata.version'
{{< /code/copyable >}}

3. List every dependency of a Spring Boot application
{{< code/copyable >}}
pack inspect-image example/app --bom | jq '.local[] | select(.name=="dependencies") | .metadata.dependencies[].name'
{{< /code/copyable >}}

4. Print the license type associated with each BOM entry
{{< code/copyable >}}
pack inspect-image example/app --bom | jq -r '.local[] | select(.metadata | has("licenses")) | .name + ": " + .metadata.licenses[].type'
{{< /code/copyable >}}

### Buildpack-Provided Labels

The Java Buildpack applies additional informational labels to the app image.

For example

### User-Provided Labels

## Components
The following component buildpacks compose the Paketo Java Buidpack.

| Buildpack | Required/Optional | Responsibility
|-----------|----------|---------------
|[Paketo BellSoft Liberica Buildpack](https://github.com/paketo-buildpacks/bellsoft-liberica) | *Required*| Provides the JDK and/or JRE.
|[Paketo Gradle Buildpack](https://github.com/paketo-buildpacks/gradle) | *Optional* | Builds Gradle-based applications from source.
|[Paketo Leiningen Buildpack](https://github.com/paketo-buildpacks/leiningen) | *Optional* | Builds Leiningen-based applications from source.
|[Paketo Maven Buildpack](https://github.com/paketo-buildpacks/maven) | *Optional* | Builds Maven-based applications from source.
|[Paketo SBT Buildpack](https://github.com/paketo-buildpacks/sbt) | *Optional* | Builds SBT-based applications from source.
|[Paketo Executable JAR Buildpack](https://github.com/paketo-buildpacks/executable-jar) | *Optional* | Contributes a process Type that launches an executable JAR.
|[Paketo Apache Tomcat Buildpack](https://github.com/paketo-buildpacks/apache-tomcat)| *Optional*| Contributes Apache Tomcat and a process type that launches a WAR with Tomcat.
|[Paketo DistZip Buildpack](https://github.com/paketo-buildpacks/dist-zip)| *Optional*. |  Contributes a process type that launches a DistZip-style application.
|[Paketo Spring Boot Buildpack](https://github.com/paketo-buildpacks/spring-boot)| *Optional* | Contributes configuration and metadata to Spring Boot applications.
|[Paketo Procfile Buildpack](https://github.com/paketo-buildpacks/procfile)| *Optional* | Allows the application to define or redefine process types with a [Procfile]({{< ref "/docs/buildpacks/configuration#procfiles" >}})
|[Paketo Azure Application Insights Buildpack](https://github.com/paketo-buildpacks/procfile)|*Optional* | Contributes the Application Insights Agent and configures it to connect to the service.
|[Paketo Debug Buildpack](https://github.com/paketo-buildpacks/debug)|*Optional* | Configures debugging for JVM applications.
|[Paketo Google Stackdriver Buildpack](https://github.com/paketo-buildpacks/procfile)|*Optional* | Contributes Stackdriver agents and configures them to connect to the service.
|[Paketo JMX Buildpack](https://github.com/paketo-buildpacks/jmx)| *Optional* | Configures JMX for JVM applications.
|[Paketo Encrypt At Rest Buildpack](https://github.com/paketo-buildpacks/encrypt-at-rest)| *Optional* | Encrypts an application layer and contributes a profile script that decrypts it at launch time.
|[Paketo Environment Variables Buildpack](https://github.com/paketo-buildpacks/environment-variables)| *Optional* | Contributes arbitrary user-provided environment variables to the image.
|[Paketo Image Labels Buildpack](https://github.com/paketo-buildpacks/image-labels)| *Optional* | Contributes OCI-specific and arbitrary user-provided labels to the image.



[dist-zip]:https://docs.gradle.org/current/userguide/distribution_plugin.html
[gradle]:https://gradle.org/
[maven]:https://maven.apache.org/
[leiningen]:https://leiningen.org/
[sbt]:https://www.scala-sbt.org/index.html
[executable jar]:https://en.wikipedia.org/wiki/JAR_(file_format)#Executable_JAR_files
[war]:https://en.wikipedia.org/wiki/WAR_(file_format)
[java]:https://github.com/paketo-buildpacks/java
