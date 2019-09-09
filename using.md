---
title: Tutorial for Pivotal Build Service
owner: Partners
---

This topic describes how to get started with Pivotal Build Service.

### <a id='configure-cli'></a> Configuring Pivotal Build Service CLI

Once you have deployed Pivotal Build Service, the `pb` CLI can be used to target it with the following commands:
```bash
pb api set <BUILD-SERVICE-API>
```
An example of the `<BUILD-SERVICE-API>` would be `https://build-service.example.com`. Use the `--skip-ssl-validation` flag if the Pivotal Build Service targets a UAA that has a self-signed CA cert.

You can confirm the targeted Pivotal Build Service using the below command:
```bash
pb api get
```

Once you are targeting the intended Pivotal Build Service, you can log into it as follows:

```bash
pb login
```

This command prompts you for a `username` and `password` which correspond to your UAA credentials. Additionally, the username and password can be passed in via environment variables with the following names:
`BUILD_SERVICE_USERNAME` and `BUILD_SERVICE_PASSWORD`. The CLI will default to pick these up and use them if they exist in the environment.


### <a id='create-team'></a> Creating a `team`

A `team` is an entity in Pivotal Build Service that is used to manage authentication for the images built by Pivotal Build Service and to manage registry and git credentials for the images managed by said team.  Only the users that belong to a team will be allowed to create images against said team. Additionally, they will be the only ones who can check the builds against an image.

```
pb team create example-team-name
```

**Current Constraints:**
* Cannot reference UAA user groups

#### <a id='add-secrets'></a> Add secrets to team

1. A file in which a registry credential is associated with a team.  Build Service will utilize this credentials to deliver container image builds to the user's specified registry.  The registry credential provided should belong to a user with `write` access on the registry.

    ```yaml
    team: example-team-name
    registry: registry.default.com
    username: <registry username>
    password: <registry password>
    ```
    And then run
    ```
    pb secrets registry apply -f path/to/<example-registry-creds>.yaml
    ```

    **Current Constraints**

    * Users can only pass one registry secret per command
    * The registry credential a given team uses can be updated by modifying the above file and running the 
    `pb secrets registry apply` command.

1. A file in which a git secrets is declared.  If a user wants Build Service to execute builds against app source code that lives in a private git repository, they must associate a git secret with the team they previously created (see step 1).

    ```yaml
    team: example-team-name
    repository: github.com
    username: testuser
    password: ********
    ```
    And then run
    ```
    pb secrets git apply -f path/to/<example-git-secret>.yaml
    ```

    **Note**  The git secret a given team uses can be updated by modifying the above file and running the 
    `pb secrets git apply` command.

    As you apply each of these files, the `pb` CLI will provide feedback indicating whether or not the commands succeeded.


#### <a id='add-ownwers'></a> Add Owners to team

Add additional team owners, by running the command

```
pb team user add some-user@email.com --team example-team-name
```

The owners will be allowed to create new images, manage secrets and users in the team

**Current Constrainst**
* The email address need to exist in UAA

### <a id='create-image'></a> Creating an `image`

An image defines the specification that Pivotal Build Service uses to create images for a user. Here is an example of an image configuration:

```yaml
team: example-team-name
source:
  git:
    url: https://github.com/example-org/sample-node-app
    revision: master
build:
  env:
  - name: BP_JAVA_VERSION
    value: 8.*
image:
  tag: registry.default.com/my-team-folder/node-app-image
```

It is composed of the following components:

1. The `team` that the image belongs to. It has to be the team you are a part of as well. You can only create images for teams you belong to.
1. The `source` defines the git location of the code that images will be built against. The `revision` can either be a branch, tag or a commit-sha. When targeted against a branch, a build is triggered for every new commit. In case this is a private git repo, its credentials must be specified in the `repositories` section of the team configuration.
1. The `build` defines additional configuration you would like your app to be built with.  The `env` is a list of environment variables that will be provided to the build. Each environment variable is an object with `name` and `value`.
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

**Current Constraints:**

* Users can only specify source code that lives in a git repository
* Pivotal Build Service does not rebuild images based on new OS packages (like cflinuxfs3)

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

This is the output of a successful build. Failed builds should indicate the reason of the failure in the logs.

The logs for a running build can be followed by using the `-f` flag, much like this:

```bash
pb image logs <image-tag> -b <build-number> -f
```
This should follow along with the progress of the build and terminate when the build completes.

**Current Constraints:**

* Pivotal Build Service only stores the ten most recent successful builds and ten most recent failed builds.

### <a id='delete'></a> Deleting `teams` and `images`

#### <a id='delete-image'></a> Delete `image`

The commands to delete a team and an image are similar to each other. To delete an image run:

```bash
pb image delete <image-tag>
```

If the operation is successful, the CLI will display the message: `Successfully deleted image <image-tag>`

This will delete all the builds that belong to the Pivotal Build Service image. This **WILL NOT** delete the images that have been generated by those builds. To delete those images, one would have to delete each of them manually from the registry.

Similarly, team deletion can be performed using:

#### <a id='delete-team'></a> Delete `team`

```bash
pb team delete <team-name>
```
If the operation is successful, the CLI will display the message: `Successfully deleted team <team-name>`

Deleting a team will also delete any registry credentials and git secrets associated with that team.

Teams **CANNOT** be deleted if they have images on Pivotal Build Service that belong to them. A team can be deleted only once all images owned by the team on Pivotal Build Service have been deleted.


#### <a id='delete-secret'></a> Delete `secret` from a team

Additionally, users can delete a registry credential or git secret associated with a given team by using

```
pb secrets registry delete <name-of-registry.io> -t example-team-name
```
and
```
pb secrets git delete github.com -t example-team-name
```

#### <a id='remove-owner'></a> Remove owner from team

Execute the following command

```
pb team user remove some-user@email.com --team example-team-name
```

Users can remove themselfs from a team.

The teams need to have at least 1 owners associated with tehm.

The user needs to be an owners of a team in order to remove other members.