## Reference
### Maven Buildpack
#### Behavior
#### Contribution
##### Dependencies
##### Environment Variables
##### Bill of Materials
#### Build Configuration
##### Environment Variables
##### Bindings
#### Bill of Materials Contributions





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