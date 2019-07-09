---
title: Tutorial for Pivotal Build Service
owner: Partners
---

This topic describes how to get started with Pivotal Build Service.

### <a id='configure-cli'></a> Configuring Pivotal Build Service CLI

Once you have deployed Pivotal Build Service, the `pb` cli can be used to target it with the following commands:
```bash
pb api set <BUILD-SERVICE-API>
```
An example of the `<BUILD-SERVICE-API>` would be `https://build-service.example.com`. Use the `--skip-ssl-validation` flag if the Pivotal Build Service targets a UAA that has an self-signed CA cert.

You can confirm the targeted Pivotal Build Service using the below command:
```bash
pb api get
```

Once you are targeting the intended Pivotal Build Service, you can login to it as follows:

```bash
pb login
```

This command prompts you for a `username` and `password` which correspond to your UAA credentials. Additionally, the username and password can be passed in via environment variables with the following names:
`BUILD_SERVICE_USERNAME` and `BUILD_SERVICE_PASSWORD`. The cli will default to pick these up and use them if they exist in the environment.


### <a id='create-team'></a> Creating a `team`

A `team` is an entity on Pivotal Build Service that is used to manage authentication for the images built by Pivotal Build Service and to manage registry and git credentials for the images managed by said team.

**All the credentials required during image creation need to be a part of the team configuration. This includes registry credentials for the built images and repository credentials for the source code if it lies in a private repository**

Only the users that belong to a team will be allowed to create images against said team. Additionally, they will be the only ones who can check the builds against an image.

You can configure a team using a file with the following `yaml` structure.

```yaml
name: example-team-name
registries:
- registry: registry.default.com
  username: <registry username>
  password: <registry password>
- registry: example.artifactory.com
  username: <artifactory username>
  password: <artifactory password>
repositories:
- domain: github.com
  username: <github-username>
  password: <password-for-github-user>
```

The registry credentials provided should belong to a user with `write` access on the registry.

Save this file as `<example-team>.yaml` which can then be provided to Pivotal Build Service:

```bash
pb team apply -f /path/to/<example-team>.yaml
```

If the operation is successful, the cli will display the message: `Successfully applied team example-team-name`

**Constraints:**
1. Only one user per team
1. Cannot reference UAA user groups

### <a id='create-image'></a> Creating an `image`

An image defines the specification that Pivotal Build Service uses to create images for a user. Here is an example of an image configuration:

```yaml
team: example-team-name
source:
  git:
    url: https://github.com/example-org/sample-node-app
    revision: master
image:
  tag: registry.default.com/my-team-folder/node-app-image
```

It is composed of the following components:

1. The `team` that the image belongs to. It has to be the team you are a part of as well. You can only create images for teams you belong to.
1. The `source` defines the git location of the code that images will be built against. The `revision` can either be a branch, tag or a commit-sha. When targeted against a branch, a build is triggered for every new commit. In case this is a private git repo, its credentials must be specified in the `repositories` section of the team configuration.
1. An `image registry` defines the destination registry of the builds for the image. The credentials for the target registry must be specified in the `registries` section of the team configuration. This should also match the domain of one of the registries provided in the team configuration.

The value of `image.tag` will be used to refer to the image once it has been created within Pivotal Build Service. Updating this field will lead to the creation of a new image.

The above image configuration can be saved as `<my-example-image>.yaml`

The configuration of the image can be applied to Pivotal Build Service:

```bash
pb image apply -f /path/to/<my-example-image>.yaml
```

Pivotal Build Service auto-rebuilds images when one or more of the following things change:
1. New buildpack versions are made available through an updated builder image
1. New commit on a branch or tag Pivotal Build Service is tracking
1. Updating the commit, branch, or git repo on the image's configuration file and re-applying it via `pb image apply`

**Constraints:**

1. Users can only specify source code that lives in a git repository
1. Pivotal Build Service does not rebuild images based on new OS packages (like cflinuxfs3)

### <a id='monitor-builds'></a> Monitoring `builds` for an `image`

You can list all the builds created for an image with:

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
[build-step-detect] pass: Cloud Foundry Spring Boot Buildpack
[build-step-detect] pass: Cloud Foundry Apache Tomcat Buildpack
...
[build-step-detect] skip: Cloud Foundry JMX Buildpack
[build-step-detect] pass: Cloud Foundry Spring Auto-reconfiguration Buildpack
[build-step-detect]
[build-step-restore] Restoring cached layer 'io.pivotal.openjdk:openjdk-jdk'
...
[build-step-restore] Restoring cached layer 'org.cloudfoundry.springboot:spring-boot'
[build-step-restore]
[build-step-analyze] Analyzing image 'registry.com/sample/demo@sha256:8ff708081ee10f7039f77275f1e6eb6359cae8d90028c79a5c493ced0dc63f68'
[build-step-analyze] Using cached layer 'io.pivotal.openjdk:openjdk-jdk'
...
[build-step-analyze] Rewriting metadata for layer 'org.cloudfoundry.springboot:spring-boot'
[build-step-analyze] Writing metadata for uncached layer 'io.pivotal.clientcertificatemapper:client-certificate-mapper'
[build-step-analyze] Writing metadata for uncached layer 'org.cloudfoundry.springautoreconfiguration:auto-reconfiguration'
[build-step-analyze]
[build-step-build]
[build-step-build] Pivotal OpenJDK Buildpack 1.0.0-M9
[build-step-build]   OpenJDK JDK 11.0.3: Reusing cached layer
[build-step-build]   OpenJDK JRE 11.0.3: Reusing cached layer
[build-step-build]   JVMKill Agent 1.16.0: Reusing cached layer
[build-step-build]   Class Counter 1.0.0-M9: Reusing cached layer
[build-step-build]   Memory Calculator 4.0.0: Reusing cached layer
[build-step-build]
...
[build-step-build]     task:        java -cp $CLASSPATH $JAVA_OPTS io.buildpacks.example.sample.SampleApplication
[build-step-build]     web:         java -cp $CLASSPATH $JAVA_OPTS io.buildpacks.example.sample.SampleApplication
[build-step-build]
[build-step-build] Pivotal Client Certificate Mapper Buildpack 1.0.0-M9
[build-step-build] Cloud Foundry Spring Auto-reconfiguration Buildpack 1.0.0-M9
[build-step-build]   Spring Auto-reconfiguration 2.7.0: Reusing cached layer
[build-step-build]
[build-step-export] Reusing layers from image 'index.docker.io/matthewmcnew/demo@sha256:8ff708081ee10f7039f77275f1e6eb6359cae8d90028c79a5c493ced0dc63f68'
[build-step-export] Reusing layer 'app' with SHA sha256:02e0070ce11bac1829174ec1296dcb1f3f04a4c30a958e2c41ad5498f78898fe
...
[build-step-export] Reusing layer 'org.cloudfoundry.springautoreconfiguration:auto-reconfiguration' with SHA sha256:93d94baf6d0dfc4981eb7d8ddfc4ae51f5c13cf87789b64ae8c4b015318a1b43
[build-step-export] *** Images:
[build-step-export]       registry.com/sample/demo:latest - succeeded
[build-step-export]       registry.com/sample/demo:b2.20190709.145448 - succeeded
[build-step-export]
[build-step-export] *** Digest: sha256:48a4ca8e4d8e8a9af26437588d0ce0e9d5c09b53aeb3ef64230a3d58d4b0dc90
[build-step-export]
[build-step-cache] Reusing layer 'io.pivotal.openjdk:openjdk-jdk' with SHA sha256:5554c7c06a266eb44a7cbdf0ecfaa14070e21af2b0bdfd1edd3b96f5168cd511
[build-step-cache] Reusing layer 'io.pivotal.buildsystem:build-system-cache' with SHA sha256:3b03fdd870a2dc1e924a040b604c25b76efafc1324ceb08eae8eae686fc3a940
...
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

Pivotal Build Service stores the 10 most recent successful builds and 10 most recent failed builds.

### <a id='delete'></a> Deleting `teams` and `images`

The commands to delete a team and an image are similar. To delete an image run:

```bash
pb image delete <image-tag>
```

If the operation is successful, the cli will display the message: `Successfully deleted image <image-tag>`

This will delete all the builds that belong to the Pivotal Build Service image. This **WILL NOT** delete the images that have been generated by those builds. To delete those images, one would have to delete each of them manually from the registry.

Similarly, team deletion can be performed using:

```bash
pb team delete <team-name>
```
If the operation is successful, the cli will display the message: `Successfully deleted team <team-name>`

Teams **CAN NOT** be deleted if they have images on Pivotal Build Service that belong to them. A team can be deleted only once all images owned by the team on Pivotal Build Service have been deleted.
