---
title: "Java Buildpack"
weight: 303
menu:
  main:
    parent: "buildpacks"
---

# Java Buildpack
The [Java Buildpack](https://github.com/) allows users to create an image from a precompiled artifact or directly from source.

To build a sample app locally with this buildpack using the `pack` CLI, run

{{< code/copyable >}}
git clone https://github.com/paketo-buildpacks/samples
cd samples/jar
pack build example/app --buildpack gcr.io/paketo-buildpacks/java
{{< /code/copyable >}}

## Table of Contents
<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Java Buildpack](#java-buildpack)
  - [Table of Contents](#table-of-contents)
  - [Build](#build)
    - [From Source](#from-source)
    - [From a Precompiled Artifact](#from-a-precompiled-artifact)
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
      - [JAR](#jar)
  - [Reference](#reference)
    - [Source Code](#source-code)
    - [Process Types](#process-types)
    - [Providing Additional Arguments](#providing-additional-arguments-1)
    - [JRE](#jre)
  - [Build Configuration](#build-configuration)
    - [JRE/JDK](#jrejdk)
    - [Precompiled Artifacts](#precompiled-artifacts)
      - [JAR](#jar-1)
      - [WAR](#war)
      - [Distribution ZIP](#distribution-zip)
    - [Build Tools](#build-tools)
      - [Gradle](#gradle)
        - [Environment Variables](#environment-variables-2)
      - [Maven](#maven)
        - [Environment Variables](#environment-variables-3)
        - [Maven Settings](#maven-settings)
      - [Leinigen](#leinigen)
      - [sbt](#sbt)
    - [APM Integrations](#apm-integrations)
      - [Azure Application Insights](#azure-application-insights)
      - [Google Stackdriver](#google-stackdriver)
    - [Debug Configuration](#debug-configuration)
    - [Remapping Dependency URIs](#remapping-dependency-uris)
  - [Run Configuration](#run-configuration)
  - [Metadata](#metadata)

## Building an Image
The java buildpack can build an image from source or from a precompiled artifact.

### From Source
The java buildpack can also build from Source using any of the following build tools:
* Gradle
* Leiningen
* Maven
* sbt

The build should produce one the of supported artifact formats in the section below. After building the buildpack will replace provided application source code with the exploded archive.

The following example uses the `pack` CLI, to build an image from source code with `maven`. This build will produce an app image identical to produce by the previous example.

{{< code/copyable >}}
cd samples/java/maven
pack build example/app
{{< /code/copyable >}}

### From a Precompiled Artifact
An application developer may build an image from following archive formats:
* JAR
* WAR
* Distribution Zip
 
The java buildpack expects the application directory to contain the extracted contents of the archive (e.g. an exploded JAR). Most platforms will automatically extract provided archives.

The following example uses the `pack` CLI, to build from an application JAR file provided with the `--path` flag.

{{< code/copyable >}}
cd samples/java/maven
./mvnw package
pack build example/app --path ./target/demo-0.0.1-SNAPSHOT.jar --buildpack gcr.io/paketo-buildpacks/java
{{< /code/copyable >}}


### Configuration
Users may configure a build with the java buildpack via either environment or a mounted directory called a binding. Bindings are used to provide credentials or other configuration that should be kept secret.

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

### Building Behind a Firewall

#### Proxies
The Java Buildpack can be configured to route traffic through a proxy using the `http_proxy`, `https_proxy`, and `no_proxy` environment variables. `pack` will handle setting those environment variables in the build container if they are set in the host environment.

#### Dependency Mappings
The Java Buildpack may download dependencies from the internet. For example, the BellSoft Liberica JRE will be downloaded from the corresponding [github](https://github.com/bell-sw/Liberica/releases) by default.

If a dependency URI is inaccessible from the build environment, a binding can be used to map a new URI to a given dependency. This allows organizations to upload a copies of vetted dependencies to an accessible location and provide developers or build systems with configuration that points the buildpack at the accessible dependencies.

The URI mappings should be provided to the buildpack as a binding with type `dependency-mapping`. Each key value pair in the binding should map the `sha256` of a dependency to a URI. Information about the dependencies a buildpack may download can be found in the `buildpack.toml` of the component buildpack.

For example, if an organization wanted to make the Bellsoft Liberica JRE dependency accessible to builds in an environment that was not able to connect to github an operator should:
1. Find the `sha256` and default `uri` for the desired dependency in [buildpack.toml](https://github.com/paketo-buildpacks/bellsoft-liberica/blob/main/buildpack.toml) of the `paketo-buildpacks/bellsoft-liberica` buildpack.
2. Download the dependency from the `uri` (e.g. https://github.com/bell-sw/Liberica/releases/download/11.0.8+10/bellsoft-jre11.0.8+10-linux-amd64.tar.gz) and upload it to a location on the internal network (e.g. `https://internal.example.com/dependencies/bellsoft-jre11.0.8+10-linux-amd64.tar.gz`) that is accessible during the build.
3. Provide a binding with:
   * `type` equal to `dependency-mapping`
   * A key/value pair where the key is equal to the `sha256` of the dependency (e.g `b4cb31162ff6d7926dd09e21551fa745fa3ae1758c25148b48dadcf78ab0c24c`) and the value is equal to the new URI.
 3. Provide this binding to all builds.

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

## Modular Components
The Java Buildpack is composed multiple component buildpacks.

* [`paketo-buildpacks/apache-tomcat`](https://github.com/paketo-buildpacks/apache-tomcat)
* [`paketo-buildpacks/azure-application-insights`](https://github.com/paketo-buildpacks/azure-application-insights)
* [`paketo-buildpacks/bellsoft-liberica`](https://github.com/paketo-buildpacks/bellsoft-liberica)
* [`paketo-buildpacks/debug`](https://github.com/paketo-buildpacks/debug)
* [`paketo-buildpacks/dist-zip`](https://github.com/paketo-buildpacks/dist-zip)
* [`paketo-buildpacks/encrypt-at-rest`](https://github.com/paketo-buildpacks/encrypt-at-rest)
* [`paketo-buildpacks/environment-variables`](https://github.com/paketo-buildpacks/environment-variables)
* [`paketo-buildpacks/executable-jar`](https://github.com/paketo-buildpacks/executable-jar)
* [`paketo-buildpacks/google-stackdriver`](https://github.com/paketo-buildpacks/google-stackdriver)
* [`paketo-buildpacks/gradle`](https://github.com/paketo-buildpacks/gradle)
* [`paketo-buildpacks/image-labels`](https://github.com/paketo-buildpacks/image-labels)
* [`paketo-buildpacks/jmx`](https://github.com/paketo-buildpacks/jmx)
* [`paketo-buildpacks/maven`](https://github.com/paketo-buildpacks/maven)
* [`paketo-buildpacks/procfile`](https://github.com/paketo-buildpacks/procfile)
* [`paketo-buildpacks/sbt`](https://github.com/paketo-buildpacks/sbt)
* [`paketo-buildpacks/spring-boot`](https://github.com/paketo-buildpacks/spring-boot)

#### JAR
**Detection Criteria**: A `META-INF/MANIFEST.MF` containing a Main-Class entry

If an exploded JAR is detected, the buildpack will:
* Add the application directory to the Class Path
* Add any Class-Path entries from  `META-INF/MANIFEST.MF` to the Class Path
* Add `executable-jar`, `web` and `task` process types that execute the following command

``

## Reference
###

### Source Code

### Process Types


### Providing Additional Arguments

### JRE

## Build Configuration

The Java Buildpack accepts most configuration options as environment variables.

### JRE/JDK
### Precompiled Artifacts

#### JAR

#### WAR

#### Distribution ZIP

### Build Tools

#### Gradle

##### Environment Variables
* `BP_MAVEN_BUILD_ARGUMENTS`
    * **Default**: `-Dmaven.test.skip=true package`
    * **Description**: Arguments to pass to `maven`.
    * **Example**: To package using the `dev` profile, set `BP_MAVEN_BUILD_ARGUMENTS="-Dmaven.test.skip=true package -Pdev`
    
* `BP_MAVEN_BUILT_MODULE`
    * **Default**: none (the root/parent module is selected by default)
    * **Description**: The subdirectory in which the buildpack will look for the application artifact, typically a submodule in a multi-module project. Superceded by `BP_MAVEN_BUILT_ARTIFACT`.
    * **Example**: If `BP_MAVEN_BUILT_MODULE=my-service`, the buildpack will look for an artifact matching `my-service/target/*.[jw]ar` after building.
    
* `BP_MAVEN_BUILT_ARTIFACT`
    * **Default**: `target/*.[jw]ar`
    * **Description**: Path to the built application artifact, relative to the application root. Matching artifacts are selected using [shell pattern matching](https://www.gnu.org/software/bash/manual/html_node/Pattern-Matching.html#Pattern-Matching). If the pattern matches more than one valid artifact the buildpack will return an error. Supercedes `BP_MAVEN_BUILT_MODULE`.
    * **Example**: Given `BP_MAVEN_BUILT_ARTIFACT=my-service/target/artifact-*.zip`, the buildpack would select the artifact at path `my-service/target/artifact-1.2.3.zip`.
    
#### Maven
If a file with the name `pom.xml` exists at the root of the application the buildpack will build the application with [Maven][TODO: fill me in].

##### Environment Variables
* `BP_MAVEN_BUILD_ARGUMENTS`
    * **Default**: `-Dmaven.test.skip=true package`
    * **Description**: Arguments to pass to `maven`.
    * **Example**: To package using the `dev` profile, set `BP_MAVEN_BUILD_ARGUMENTS="-Dmaven.test.skip=true package -Pdev`
    
* `BP_MAVEN_BUILT_MODULE`
    * **Default**: none (the root/parent module is selected by default)
    * **Description**: The subdirectory in which the buildpack will look for the application artifact, typically a submodule in a multi-module project. Superceded by `BP_MAVEN_BUILT_ARTIFACT`.
    * **Example**: If `BP_MAVEN_BUILT_MODULE=my-service`, the buildpack will look for an artifact matching `my-service/target/*.[jw]ar` after building.
    
* `BP_MAVEN_BUILT_ARTIFACT`
    * **Default**: `target/*.[jw]ar`
    * **Description**: Path to the built application artifact, relative to the application root. Matching artifacts are selected using [shell pattern matching](https://www.gnu.org/software/bash/manual/html_node/Pattern-Matching.html#Pattern-Matching). If the pattern matches more than one valid artifact the buildpack will return an error. Supercedes `BP_MAVEN_BUILT_MODULE`.
    * **Example**: Given `BP_MAVEN_BUILT_ARTIFACT=my-service/target/artifact-*.zip`, the buildpack would select the artifact at path `my-service/target/artifact-1.2.3.zip`.
    
##### Maven Settings

To provide a custom maven `settings.xml` file


#### Leinigen
If a file with the name `project.clj` exists at the root of the application the Java Buildpack will build the application with [Leiningen][l].

* `BP_LEIN_BUILD_ARGUMENTS`
    * **Default**: `uberjar`
    * **Description**: Arguments to pass to the `lein` CLI.
    * **Example**: To run unit test first and then build a war file with the [lein-uberwar][lu] plugin, set `BP_LEIN_BUILD_ARGUMENTS="do test, uberwar"`

* `BP_LEIN_BUILT_ARTIFACT`
    * **Default**: `uberjar`
    * **Description**: Arguments to pass to the `lein` CLI.
    * **Example**: To run unit test first and then build a war file with the [lein-uberwar][lu] plugin, set `BP_LEIN_BUILD_ARGUMENTS="do test, uberwar"`

* `BP_LEIN_BUILT_MODULE`
    * **Default**: N/A
    * **Description**: Desired Module within a multimodule project. Super
    * **Example**:
    
 

#### sbt

### APM Integrations

#### Azure Application Insights

#### Google Stackdriver

### Debug Configuration

### Remapping Dependency URIs

## Run Configuration

## Metadata


[l]:http://www.apache.org/licenses/LICENSE-2.0
[lu]:https://github.com/luminus-framework/lein-uberwar