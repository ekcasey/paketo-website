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

## Building from a Precompiled Artifact
An application developer may build an image from following archive formats:
* JAR
* WAR
* Distribution Zip
 
The java buildpack expects the application directory to contain the extracted contents of the archive (e.g. an exploded JAR). Most platforms will automatically extract provided archives.

The following example uses the `pack` CLI, to build from an application JAR file provided with the `--path` flag.

{{< code/copyable >}}
cd samples/java/maven
./mvnw package
pack build example/app --path ./target/demo-0.0.1-SNAPSHOT.jar
{{< /code/copyable >}}

## Building from Source
The java buildpack can also build from Source using any of the following build tools:
* gradle
* leiningen
* maven
* sbt

The build should produce one the of supported artifact formats. After building the buildpack will replace provided application source code with the exploded archive.

The following example uses the `pack` CLI, to build an image from source code with `maven`. This build will produce an app image identical to produce by the previous example.

{{< code/copyable >}}
cd samples/java/maven
pack build example/app
{{< /code/copyable >}}

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

### Modular Components

#### JAR
**Detection Criteria**: A `META-INF/MANIFEST.MF` containing a Main-Class entry

If an exploded JAR is detected, the buildpack will:
* Add the application directory to the Class Path
* Add any Class-Path entries from  `META-INF/MANIFEST.MF` to the Class Path
* Add `executable-jar`, `web` and `task` process types that execute the following command

``
 

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