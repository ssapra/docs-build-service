---
title: Tutorial for Pivotal Build Service
owner: Partners
---

This topic describes how to get started with Pivotal Build Service.

### <a id='install'></a> Targeting and Logging into a running Build Service

Once you have deployed Build Service, the `pb` cli can be used to target it with the following commands:
```bash
pb api set <BUILD-SERVICE-API>
```
An example of the `<BUILD-SERVICE-API>` would be `https://build-service.example.com`. Use the `--skip-ssl-validation` flag if the Build Service targets a UAA that has an self-signed CA cert.

You can confirm the targeted Build Service using the below command:
```bash
pb api get
```

Once you are targeting the intended Build Service, you can login to it as follows:

```bash
pb login
```

This command prompts you for a `username` and `password` which correspond to your UAA credentials. Additionally, the username and password can be passed in via env variable with the following names:
`BUILD_SERVICE_USERNAME` and `BUILD_SERVICE_PASSWORD`. The cli will default to pick these up and use them if they exist in the env.  


### <a id='install'></a> Creating a `team`

A `team` is an entity on Build Service that is used to managed auth for the images Build Service builds and to manage registry and git credentials for the images managed by said team.

**All the credentials required during image creation need to be a part of the team configuration. This includes registry credentials for the built images and repository credentials for the source code if it lies in a private repository**

Only the users that belong to a team will be allowed to create images against said team. Additionally, they will be they only ones who can monitor the builds against an image.

You can configure a team using a file with the following `yaml` structure.

```yaml
name: example-team-name
registries:
- registry: registry.default.svc.cluster.io
  username: <username with write access to above registry>
  password: <password for above user>
- registry: example.artifactory.com
  username: <artifactory username>
  password: <artifactory user password>
repositories:
- domain: github.com
  username: <github-username>
  password: <password-for-github-user>
```

Save this file as `<example-team>.yaml` which can then be provided to Build Service:

```bash
pb team apply -f /path/to/<example-team>.yaml
```

If the operation is successful, the cli will display the message: `Successfully applied team example-team-name`

**Constraints:**
1. Only one user per team
1. Cannot reference UAA user groups

### <a id='install'></a> Creating an `image`

An image defines the specification that Build Service uses to create images for a user. Here is an example of an image configuration: 

```yaml
team: example-team-name
source:
  git:
    url: https://github.com/example-org/sample-node-app
    revision: master
image:
  tag: registry.default.svc.cluster.io/my-team-folder/node-app-image
```

It is composed of the following components:

1. The `team` that the image belongs to. It has to be the team you are a part of as well. You can only create images for teams you belong to.
1. The `source` that defines that src that images will be built against. The revision can either be a branch or a commit-sha. When targeted against a branch, a build is triggered for every new commit. In case this is a private git repo, its credentials must be specified in the `repositories` section of the team configuration.
1. An `image registry` that defines the destination registry of the builds for the image. The credentials for the target registry must be specified in the `registries` section of the team configuration.

The value of `image.tag` will be used to refer to the image once it has been created within Build Service. Updating this field will lead to the creation of a new image.

The above image configuration can be saved as `<my-example-image>.yaml`

The configuration of the team can be applied to build service:

```bash
pb image apply -f /path/to/<my-example-image>.yaml
```

New builds of an images when one or more of the following things change:
1. New buildpack versions are made available through an updated builder image
1. New commit on a branch build service is tracking
1. Updating the commit, branch, or git repo on the image's configuration file and re-applying it via `pb image apply`

**Constraints:**

1. Users can only specify source code that lives in a git repo  
1. The Build Service Alpha does not rebuild images based on new OS packages (like cflinuxfs3)

### <a id='install'></a> Monitoring `builds` against an `image`

You can list all the builds created against an image with:

```bash
pb image builds <image-tag>
```

The `<image-tag>` in the above command is the value of the field `image.tag` in the image's configuration. The output of the command might look similar to what's described below:

```
Build    Status     Image       Started Time           Finished Time
-----    ------     -----       ------------           -------------
    1    SUCCESS    f5a1725f    2019-07-08 21:55:27    2019-07-08 21:56:54
    2    SUCCESS    428fe93e    2019-07-08 21:56:55    2019-07-08 21:57:40
    3    FAILED       --        2019-07-08 21:58:55    2019-07-08 21:59:40
    4    BUILDING     --        2019-07-08 21:58:55          --
    -    PENDING      --              --                     --
```

1. The `Build` column describes the index of builds in the order that they were built.
1. The `Status` column describes the status of a previous or a running/pending build.
1. The `Image` column contains the SHA256 of the image successfully built.
1. The  `Started Time` and `Finished Time` columns described when a build was kicked off and when it was completed.

To get the logs of a particular build, run the following:

```bash
pb image logs <image-tag> -b <build-number>
```

The output of the command will look similar to this:

```
[build-step-credential-initializer] {"level":"info","ts":1562684107.3441668,"logger":"fallback-logger","caller":"creds-init/main.go:40","msg":"Credentials initialized.","commit":"002a41a"}
[build-step-credential-initializer]
[build-step-git-source-0] git-init:main.go:81: Successfully cloned "https://github.com/buildpack/sample-java-app" @ "abde24efc17802b7e2b3814e0ead63a460e66f5f" in path "/workspace"
[build-step-git-source-0]
[build-step-prepare]
[build-step-detect] Trying group 1 out of 3 with 27 buildpacks...
[build-step-detect] ======== Results ========
[build-step-detect] skip: Cloud Foundry Archive Expanding Buildpack
[build-step-detect] pass: Pivotal OpenJDK Buildpack
[build-step-detect] pass: Pivotal Build System Buildpack
[build-step-detect] pass: Cloud Foundry JVM Application Buildpack
[build-step-detect] pass: Cloud Foundry Spring Boot Buildpack
[build-step-detect] pass: Cloud Foundry Apache Tomcat Buildpack
[build-step-detect] pass: Cloud Foundry DistZip Buildpack
[build-step-detect] skip: Cloud Foundry Procfile Buildpack
[build-step-detect] skip: Pivotal AppDynamics Buildpack
[build-step-detect] skip: Pivotal AspectJ Buildpack
[build-step-detect] skip: Pivotal CA Introscope Buildpack
[build-step-detect] pass: Pivotal Client Certificate Mapper Buildpack
[build-step-detect] skip: Pivotal Elastic APM Buildpack
[build-step-detect] skip: Pivotal JaCoCo Buildpack
[build-step-detect] skip: Pivotal JProfiler Buildpack
[build-step-detect] skip: Pivotal JRebel Buildpack
[build-step-detect] skip: Pivotal New Relic Buildpack
[build-step-detect] skip: Pivotal OverOps Buildpack
[build-step-detect] skip: Pivotal Riverbed AppInternals Buildpack
[build-step-detect] skip: Pivotal SkyWalking Buildpack
[build-step-detect] skip: Pivotal YourKit Buildpack
[build-step-detect] skip: Cloud Foundry Azure Application Insights Buildpack
[build-step-detect] skip: Cloud Foundry Debug Buildpack
[build-step-detect] skip: Cloud Foundry Google Stackdriver Buildpack
[build-step-detect] skip: Cloud Foundry JDBC Buildpack
[build-step-detect] skip: Cloud Foundry JMX Buildpack
[build-step-detect] pass: Cloud Foundry Spring Auto-reconfiguration Buildpack
[build-step-detect]
[build-step-restore] Restoring cached layer 'io.pivotal.openjdk:openjdk-jdk'
[build-step-restore] Restoring cached layer 'io.pivotal.buildsystem:build-system-application'
[build-step-restore] Restoring cached layer 'io.pivotal.buildsystem:build-system-cache'
[build-step-restore] Restoring cached layer 'org.cloudfoundry.jvmapplication:executable-jar'
[build-step-restore] Restoring cached layer 'org.cloudfoundry.springboot:spring-boot'
[build-step-restore]
[build-step-analyze] Analyzing image 'registry.com/sample/demo@sha256:8ff708081ee10f7039f77275f1e6eb6359cae8d90028c79a5c493ced0dc63f68'
[build-step-analyze] Using cached layer 'io.pivotal.openjdk:openjdk-jdk'
[build-step-analyze] Writing metadata for uncached layer 'io.pivotal.openjdk:memory-calculator'
[build-step-analyze] Writing metadata for uncached layer 'io.pivotal.openjdk:openjdk-jre'
[build-step-analyze] Writing metadata for uncached layer 'io.pivotal.openjdk:security-provider-configurer'
[build-step-analyze] Writing metadata for uncached layer 'io.pivotal.openjdk:class-counter'
[build-step-analyze] Writing metadata for uncached layer 'io.pivotal.openjdk:java-security-properties'
[build-step-analyze] Writing metadata for uncached layer 'io.pivotal.openjdk:jvmkill'
[build-step-analyze] Writing metadata for uncached layer 'io.pivotal.openjdk:link-local-dns'
[build-step-analyze] Using cached layer 'io.pivotal.buildsystem:build-system-application'
[build-step-analyze] Using cached layer 'io.pivotal.buildsystem:build-system-cache'
[build-step-analyze] Using cached launch layer 'org.cloudfoundry.jvmapplication:executable-jar'
[build-step-analyze] Rewriting metadata for layer 'org.cloudfoundry.jvmapplication:executable-jar'
[build-step-analyze] Using cached launch layer 'org.cloudfoundry.springboot:spring-boot'
[build-step-analyze] Rewriting metadata for layer 'org.cloudfoundry.springboot:spring-boot'
[build-step-analyze] Writing metadata for uncached layer 'io.pivotal.clientcertificatemapper:client-certificate-mapper'
[build-step-analyze] Writing metadata for uncached layer 'org.cloudfoundry.springautoreconfiguration:auto-reconfiguration'
[build-step-analyze]
[build-step-build]
[build-step-build] Pivotal OpenJDK Buildpack 1.0.0-M9
[build-step-build]   OpenJDK JDK 11.0.3: Reusing cached layer
[build-step-build]   OpenJDK JRE 11.0.3: Reusing cached layer
[build-step-build]   Java Security Properties 1.0.0-M9: Reusing cached layer
[build-step-build]   Security Provider Configurer 1.0.0-M9: Reusing cached layer
[build-step-build]   Link-Local DNS 1.0.0-M9: Reusing cached layer
[build-step-build]   JVMKill Agent 1.16.0: Reusing cached layer
[build-step-build]   Class Counter 1.0.0-M9: Reusing cached layer
[build-step-build]   Memory Calculator 4.0.0: Reusing cached layer
[build-step-build]
[build-step-build] Pivotal Build System Buildpack 1.0.0-M9
[build-step-build]     Using wrapper
[build-step-build]     Linking Cache to /home/vcap/.m2
[build-step-build]   Compiled Application (146 files): Contributing to layer
[build-step-build] [INFO] Scanning for projects...
[build-step-build] [INFO]
[build-step-build] [INFO] --------------------< io.buildpacks.example:sample >--------------------
[build-step-build] [INFO] Building sample 0.0.1-SNAPSHOT
[build-step-build] [INFO] --------------------------------[ jar ]---------------------------------
[build-step-build] [INFO]
[build-step-build] [INFO] --- maven-resources-plugin:3.1.0:resources (default-resources) @ sample ---
[build-step-build] [INFO] Using 'UTF-8' encoding to copy filtered resources.
[build-step-build] [INFO] Copying 1 resource
[build-step-build] [INFO] Copying 4 resources
[build-step-build] [INFO]
[build-step-build] [INFO] --- maven-compiler-plugin:3.8.0:compile (default-compile) @ sample ---
[build-step-build] [INFO] Changes detected - recompiling the module!
[build-step-build] [INFO] Compiling 1 source file to /workspace/target/classes
[build-step-build] [INFO]
[build-step-build] [INFO] --- maven-resources-plugin:3.1.0:testResources (default-testResources) @ sample ---
[build-step-build] [INFO] Not copying test resources
[build-step-build] [INFO]
[build-step-build] [INFO] --- maven-compiler-plugin:3.8.0:testCompile (default-testCompile) @ sample ---
[build-step-build] [INFO] Not compiling test sources
[build-step-build] [INFO]
[build-step-build] [INFO] --- maven-surefire-plugin:2.22.1:test (default-test) @ sample ---
[build-step-build] [INFO] Tests are skipped.
[build-step-build] [INFO]
[build-step-build] [INFO] --- maven-jar-plugin:3.1.1:jar (default-jar) @ sample ---
[build-step-build] [INFO] Building jar: /workspace/target/sample-0.0.1-SNAPSHOT.jar
[build-step-build] [INFO]
[build-step-build] [INFO] --- spring-boot-maven-plugin:2.1.3.RELEASE:repackage (repackage) @ sample ---
[build-step-build] [INFO] Replacing main artifact with repackaged archive
[build-step-build] [INFO] ------------------------------------------------------------------------
[build-step-build] [INFO] BUILD SUCCESS
[build-step-build] [INFO] ------------------------------------------------------------------------
[build-step-build] [INFO] Total time:  9.479 s
[build-step-build] [INFO] Finished at: 2019-07-09T14:55:53Z
[build-step-build] [INFO] ------------------------------------------------------------------------
[build-step-build]   Removing source code
[build-step-build]
[build-step-build] Cloud Foundry JVM Application Buildpack 1.0.0-M9
[build-step-build]   Executable JAR: Reusing cached layer
[build-step-build]   Process types:
[build-step-build]     executable-jar: java -cp $CLASSPATH $JAVA_OPTS org.springframework.boot.loader.JarLauncher
[build-step-build]     task:           java -cp $CLASSPATH $JAVA_OPTS org.springframework.boot.loader.JarLauncher
[build-step-build]     web:            java -cp $CLASSPATH $JAVA_OPTS org.springframework.boot.loader.JarLauncher
[build-step-build]
[build-step-build] Cloud Foundry Spring Boot Buildpack 1.0.0-M9
[build-step-build]   Spring Boot 2.1.3.RELEASE: Reusing cached layer
[build-step-build]   Process types:
[build-step-build]     spring-boot: java -cp $CLASSPATH $JAVA_OPTS io.buildpacks.example.sample.SampleApplication
[build-step-build]     task:        java -cp $CLASSPATH $JAVA_OPTS io.buildpacks.example.sample.SampleApplication
[build-step-build]     web:         java -cp $CLASSPATH $JAVA_OPTS io.buildpacks.example.sample.SampleApplication
[build-step-build]
[build-step-build] Pivotal Client Certificate Mapper Buildpack 1.0.0-M9
[build-step-build]   Cloud Foundry Client Certificate Mapper 1.8.0: Reusing cached layer
[build-step-build]
[build-step-build] Cloud Foundry Spring Auto-reconfiguration Buildpack 1.0.0-M9
[build-step-build]   Spring Auto-reconfiguration 2.7.0: Reusing cached layer
[build-step-build]
[build-step-export] Reusing layers from image 'index.docker.io/matthewmcnew/demo@sha256:8ff708081ee10f7039f77275f1e6eb6359cae8d90028c79a5c493ced0dc63f68'
[build-step-export] Reusing layer 'app' with SHA sha256:02e0070ce11bac1829174ec1296dcb1f3f04a4c30a958e2c41ad5498f78898fe
[build-step-export] Reusing layer 'config' with SHA sha256:d4c588715ae43b01a3e52084c1176f4b4869f9e9d3c5f34b9e022222b186e006
[build-step-export] Reusing layer 'launcher' with SHA sha256:c8a8ddb80dd9923057bd12f9f69c6b093925a8925f3c37550a88b90f02699aa9
[build-step-export] Reusing layer 'io.pivotal.openjdk:java-security-properties' with SHA sha256:936083baf63eb580e9fa014b5ad9ddf478eea50e2dc5df365c26e9ad52c6d74b
[build-step-export] Reusing layer 'io.pivotal.openjdk:jvmkill' with SHA sha256:8109ccdcdb4c5a9c3e9f6416c5509798140d540e175af6c73dcd4a6a2d260c34
[build-step-export] Reusing layer 'io.pivotal.openjdk:link-local-dns' with SHA sha256:2b7b43de758ad440aa98a54fc0f35ca76586a3552e19cf7e4c7a0edd3180773f
[build-step-export] Reusing layer 'io.pivotal.openjdk:memory-calculator' with SHA sha256:cbf263efdf8fc3397a9df5b5a1b8a0d9abed5e25d796c7ef3ecd82f3dc48559e
[build-step-export] Reusing layer 'io.pivotal.openjdk:openjdk-jre' with SHA sha256:f69bf645a2e9cf26a421758bb2e3ff07a009a364ff68e69598e2772d33631e6f
[build-step-export] Reusing layer 'io.pivotal.openjdk:security-provider-configurer' with SHA sha256:617e8e83470c5802962ebbefee0c45d791c966ea8dbe1d78f794b191da6c7723
[build-step-export] Reusing layer 'io.pivotal.openjdk:class-counter' with SHA sha256:976f0598f70dd13afb92581b997e1918137086bf8e96b662b9ca92069befb786
[build-step-export] Reusing layer 'org.cloudfoundry.jvmapplication:executable-jar' with SHA sha256:1aa0be79085534fdd3dfc2ad2bae0a77fa78bfa160176574ee4234d14ca559cf
[build-step-export] Reusing layer 'org.cloudfoundry.springboot:spring-boot' with SHA sha256:effa8b80729cafa9f9a01b21a4badb5203510de0bb2e6b309ffd2593b0a28de7
[build-step-export] Reusing layer 'io.pivotal.clientcertificatemapper:client-certificate-mapper' with SHA sha256:5f3eb30976168cabcc374c49e8b9cfdc4d1b1abd2dc7a9ebcf951b04721e07e2
[build-step-export] Reusing layer 'org.cloudfoundry.springautoreconfiguration:auto-reconfiguration' with SHA sha256:93d94baf6d0dfc4981eb7d8ddfc4ae51f5c13cf87789b64ae8c4b015318a1b43
[build-step-export] *** Images:
[build-step-export]       registry.com/sample/demo:latest - succeeded
[build-step-export]       registry.com/sample/demo:b2.20190709.145448 - succeeded
[build-step-export]
[build-step-export] *** Digest: sha256:48a4ca8e4d8e8a9af26437588d0ce0e9d5c09b53aeb3ef64230a3d58d4b0dc90
[build-step-export]
[build-step-cache] Reusing layer 'io.pivotal.openjdk:openjdk-jdk' with SHA sha256:5554c7c06a266eb44a7cbdf0ecfaa14070e21af2b0bdfd1edd3b96f5168cd511
[build-step-cache] Reusing layer 'io.pivotal.buildsystem:build-system-cache' with SHA sha256:3b03fdd870a2dc1e924a040b604c25b76efafc1324ceb08eae8eae686fc3a940
[build-step-cache] Caching layer 'io.pivotal.buildsystem:build-system-application' with SHA sha256:f2a8a8445bb7f861296e3dd282a04efbbad62821927f8ed05b6491bcf915f35d
[build-step-cache] Reusing layer 'org.cloudfoundry.jvmapplication:executable-jar' with SHA sha256:1aa0be79085534fdd3dfc2ad2bae0a77fa78bfa160176574ee4234d14ca559cf
[build-step-cache] Reusing layer 'org.cloudfoundry.springboot:spring-boot' with SHA sha256:effa8b80729cafa9f9a01b21a4badb5203510de0bb2e6b309ffd2593b0a28de7
[build-step-cache]
```

This is the output of a successful build. Failed builds should indicate the reason of the failure in the logs and can be used during debugging.

The logs for a running build can be followed by using the `-f` flag, much like this:

```bash
pb image logs <image-tag> -b <build-number> -f
```
This should follow along with the progress of the build and terminate when the build completes.

**Constraints:**

Build Service stores the 10 most recent successful builds and 10 most recent failed builds.

### <a id='install'></a> Deleting `teams` and `images`

The commands to delete a team and an image are similar. To delete an image run:

```bash
pb image delete <image-tag>
```

If the operation is successful, the cli will display the message: `Successfully delete image <image-tag>`

This will delete all the builds that belong. This **WILL NOT** delete the images that have been created by those builds. Presently, do delete those images, one would have to do them manually from the registry.

Similarly, team deletion can be performed using:

```bash
pb team delete <team-name>
```
If the operation is successful, the cli will display the message: `Successfully applied team <team-name>`

Teams **CAN NOT** be deleted if they have images that belong to them. A team can be deleted only once all images owned by the team have been deleted.