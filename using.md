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


### <a id='create-team'></a> Creating a `team` and managing members

A `team` is an entity in Pivotal Build Service that is used to manage authentication for the images built by Pivotal Build Service and to manage registry and git credentials for the images managed by said team.  Only the users that belong to a team will be allowed to create images against said team. Additionally, they will be the only ones who can check the builds against an image.

First, a user can create a team and give it a name by running `pb team create <team-name>`.  By creating a team, that user is added to the team.  Team members can add other users to a team by referencing their UAA username by running `pb team user add <uaa-user@company.com> <example-team-name>`.  Users can remove team members by running `pb team user remove <uaa-user@company.com> <example-team-name>`. 

**Current Constraints:**

* Team structure is flat, all members of a team are also team “owners” and are allowed to create new images, manage secrets, and add/remove other users from the team
* Cannot reference UAA user groups
* The email address must exist in UAA
* Each team must have at least one team member, if a particular team is no longer useful, users can delete a team by running `pb team delete <team-name>`
    * Additional details for deletion workflows are captured in a below section

Next, to effectively utilize the team they created, users must associate image registry credentials and git credentials (if the source code lives in a private git repo) with the team. 

1) Associating a registry credential with a team.  Build Service will utilize these credentials to deliver container image builds to the user's specified registry.  The registry credential provided should belong to a user with `write` access on the registry. 

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

* Users can only add a secret at a time
* The registry credential that a given team uses can be updated by modifying the above file and running the `pb secrets registry apply` command. 

2) Associating a git credential with a team.  If a user wants Build Service to execute builds against app source code that lives in a private git repository, they must associate a git secret with the team they previously created.

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

**Note**  The git secret a given team uses can be updated by modifying the above file and running the `pb secrets git apply` command. 

As you apply the registry and git secret files, the `pb` CLI will provide feedback indicating whether or not the commands succeeded.


### <a id='create-image'></a> Creating an `image`

An image defines the specification that Pivotal Build Service uses to create images for a user. Here is an example of an image configuration:

```yaml
team: example-team-name
source:
  git:
    url: https://github.com/example-org/sample-java-app
    revision: master
build:
  env:
  - name: BP_JAVA_VERSION
    value: 8.*
image:
  tag: registry.default.com/my-team-folder/java-app-image
```

It is composed of the following components:

1. The `team` that the image belongs to. It has to be the team you are a part of as well. You can only create images for teams you belong to.
1. The `source` defines the git location of the code that images will be built against. The `revision` can either be a branch, tag or a commit-sha. When targeted against a branch, a build is triggered for every new commit. 
1. The `build` defines additional configuration you would like your app to be built with.  The `env` is a list of environment variables that will be provided to the build. Each environment variable is an object with `name` and `value`.
1. An `image registry` defines the destination registry of the builds for the image. The credentials for the target registry must be specified in the `registries` section of the team configuration. This should also match the domain of one of the registries provided in the team configuration.

The value of `image.tag` will be used to refer to the image once it has been created within Pivotal Build Service. Updating this field will lead to the creation of a new image.

The above image configuration can be saved as `<my-example-image>.yaml`

**Creating an `image` against source code in a git repo**

The above configuration of the image can be applied to Pivotal Build Service:

```bash
pb image apply -f /path/to/<my-example-image>.yaml
```

**Creating an image using local source code**

Build Service supports builds against source code that lives in a git repository or locally on a users machine.  However, users can only specify one source code location.  To replicate the above image creation workflow using local source code, users can modify the file by removing the git fields.  The resulting file would look like this:

```yaml
team: example-team-name
build:
  env:
  - name: BP_JAVA_VERSION
    value: 8.*
image:
  tag: registry.default.com/my-team-folder/sample-java-app
```

Users would apply this image configuration and specify a path to their application.

```bash
pb image apply -f /path/to/<my-example-image>.yaml -p /path/to/app_directory
```

### <a id='create-image'></a> Rebuilds

Pivotal Build Service auto-rebuilds images when one or more of the following build inputs change:
1. New buildpack versions are made available through an updated builder image
1. New commit on a branch or tag Pivotal Build Service is tracking
1. Updating the commit, branch, git repo, or build fields on the image's configuration file and re-applying it via `pb image apply`
1. Uploading a new copy of local source via `pb image apply -p`

**Current Constraints:**

* Pivotal Build Service does not rebuild images based on new OS packages (like cflinuxfs3)

### <a id='monitor-builds'></a> Monitoring `builds` for an `image`

You can list all the builds created for an image with:

```bash
pb image builds <image-tag>
```

The `<image-tag>` in the above command is the value of the field `image.tag` in the image's configuration. The output of the command might look similar to what's described below:

```
Build    Status    Started Time           Finished Time          Reason    Digest  
-----    ------    ------------           -------------          ------    ------
    1    SUCCESS   2019-09-09 21:55:27    2019-07-08 21:56:54    CONFIG    *************************************************             
    2    SUCCESS   2019-09-09 21:56:55    2019-07-08 21:57:40    COMMIT    *************************************************
    3    FAILED    2019-09-09 21:58:55    2019-07-08 21:59:40    CONFIG+   --
    4    BUILDING  2019-09-09 21:58:55    --                     BUILDER   --
    -    PENDING   --                     --                     UNKNOWN   --
```

1. The `Build` column describes the index of builds in the order that they were built.
1. The `Status` column describes the status of a previous or a running/pending build image.
1. The  `Started Time` and `Finished Time` columns described when a build was kicked off and when it was completed.
1. The `Reason` column provides the user with information as to why an image rebuild occured. These reasons include
* `CONFIG`
    * Occurs when a change is made to commit, branch, git repo, or build fields on the image's configuration file and the user ran `pb image apply`
* `COMMIT`
    * Occurs when new source code is committed to a branch or tag build service is monitoring for changes
* `BUILDER`
    * Occurs when new buildpack versions are made available through an updated builder image

**Note:** It is possible for a rebuild to occur for more than one `Reason`.  In this instance, a `+` sign is appended to the `Reason` and the primary `Reason` is displayed.  Priority for the `Reason` is ranked `CONFIG`, `COMMIT`, `BUILDER`, in descending order       
 
 5. The `Digest` column contains the SHA256 of the image successfully built.  Users can reference this SHA to perform helpful Docker commands like `pull` and `inspect`

**Retrieving logs of a particular build**

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
* Users cannot retrieve logs by referencing the image digest, they must use the build number

### <a id='delete'></a> Deleting `teams` and `images`

**Delete `image`**

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
