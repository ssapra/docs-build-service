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