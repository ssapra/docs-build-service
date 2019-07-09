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
[build-step-credential-initializer] {"level":"info","ts":1562622962.7271485,"logger":"fallback-logger","caller":"creds-init/main.go:40","msg":"Credentials initialized.","commit":"002a41a"}
[build-step-credential-initializer]
[build-step-git-source-0] git-init:main.go:81: Successfully cloned "https://github.com/repository/sample-app" @ "493d972356b185cd6f086f903d604693c051b2ad" in path "/workspace"
[build-step-git-source-0]
[build-step-prepare]
[build-step-detect] Trying group 1 out of 1 with 2 buildpacks...
[build-step-detect] ======== Results ========
[build-step-detect] pass: Node.js Buildpack
[build-step-detect] pass: NPM Buildpack
[build-step-detect]
[build-step-restore] Cache '/cache': metadata not found, nothing to restore
[build-step-restore]
[build-step-analyze] Image 'gcr.io/sample-registry/test/build:latest' not found
[build-step-analyze]
[build-step-build] -----> Node.js Buildpack 0.0.6
[build-step-build] -----> NodeJS 6.16.0: Contributing to layer
[build-step-build]        Downloading from https://nodejs.org/dist/v6.16.0/node-v6.16.0-linux-x64.tar.gz
[build-step-build]        Verifying checksum
[build-step-build]        Expanding to /layers/org.cloudfoundry.buildpacks.nodejs/node
[build-step-build]        Writing NODE_HOME to shared
[build-step-build]        Writing NODE_ENV to shared
[build-step-build]        Writing NODE_MODULES_CACHE to shared
[build-step-build]        Writing NODE_VERBOSE to shared
[build-step-build]        Writing NPM_CONFIG_PRODUCTION to shared
[build-step-build]        Writing NPM_CONFIG_LOGLEVEL to shared
[build-step-build]        Writing WEB_MEMORY to shared
[build-step-build]        Writing WEB_CONCURRENCY to shared
[build-step-build]        Writing .profile.d/0_memory_available.sh
[build-step-build]
[build-step-build] -----> NPM Buildpack 0.0.7
[build-step-build] -----> node_modules 31cdf8958fc50703537518168b3bc65884a6c5497116aa8a8d77a20e39ab6bb5: Contributing to layer
[build-step-build] Installing node_modules
[build-step-build] sample-node-app@0.0.0 /workspace
[build-step-build] +-- express@4.16.4
[build-step-build] | +-- accepts@1.3.7
[build-step-build] | | +-- mime-types@2.1.24
[build-step-build] | | | `-- mime-db@1.40.0
[build-step-build] | | `-- negotiator@0.6.2
[build-step-build] | +-- array-flatten@1.1.1
[build-step-build] | +-- body-parser@1.18.3
[build-step-build] | | +-- bytes@3.0.0
[build-step-build] | | +-- http-errors@1.6.3
[build-step-build] | | | `-- inherits@2.0.3
[build-step-build] | | +-- iconv-lite@0.4.23
[build-step-build] | | | `-- safer-buffer@2.1.2
[build-step-build] | | `-- raw-body@2.3.3
[build-step-build] | +-- content-disposition@0.5.2
[build-step-build] | +-- content-type@1.0.4
[build-step-build] | +-- cookie@0.3.1
[build-step-build] | +-- cookie-signature@1.0.6
[build-step-build] | +-- debug@2.6.9
[build-step-build] | | `-- ms@2.0.0
[build-step-build] | +-- depd@1.1.2
[build-step-build] | +-- encodeurl@1.0.2
[build-step-build] | +-- escape-html@1.0.3
[build-step-build] | +-- etag@1.8.1
[build-step-build] | +-- finalhandler@1.1.1
[build-step-build] | | `-- unpipe@1.0.0
[build-step-build] | +-- fresh@0.5.2
[build-step-build] | +-- merge-descriptors@1.0.1
[build-step-build] | +-- methods@1.1.2
[build-step-build] | +-- on-finished@2.3.0
[build-step-build] | | `-- ee-first@1.1.1
[build-step-build] | +-- parseurl@1.3.3
[build-step-build] | +-- path-to-regexp@0.1.7
[build-step-build] | +-- proxy-addr@2.0.5
[build-step-build] | | +-- forwarded@0.1.2
[build-step-build] | | `-- ipaddr.js@1.9.0
[build-step-build] | +-- qs@6.5.2
[build-step-build] | +-- range-parser@1.2.1
[build-step-build] | +-- safe-buffer@5.1.2
[build-step-build] | +-- send@0.16.2
[build-step-build] | | +-- destroy@1.0.4
[build-step-build] | | `-- mime@1.4.1
[build-step-build] | +-- serve-static@1.13.2
[build-step-build] | +-- setprototypeof@1.1.0
[build-step-build] | +-- statuses@1.4.0
[build-step-build] | +-- type-is@1.6.18
[build-step-build] | | `-- media-typer@0.3.0
[build-step-build] | +-- utils-merge@1.0.1
[build-step-build] | `-- vary@1.1.2
[build-step-build] +-- jade@1.11.0
[build-step-build] | +-- character-parser@1.2.1
[build-step-build] | +-- clean-css@3.4.28
[build-step-build] | | +-- commander@2.8.1
[build-step-build] | | | `-- graceful-readlink@1.0.1
[build-step-build] | | `-- source-map@0.4.4
[build-step-build] | |   `-- amdefine@1.0.1
[build-step-build] | +-- commander@2.6.0
[build-step-build] | +-- constantinople@3.0.2
[build-step-build] | | `-- acorn@2.7.0
[build-step-build] | +-- jstransformer@0.0.2
[build-step-build] | | +-- is-promise@2.1.0
[build-step-build] | | `-- promise@6.1.0
[build-step-build] | |   `-- asap@1.0.0
[build-step-build] | +-- mkdirp@0.5.1
[build-step-build] | | `-- minimist@0.0.8
[build-step-build] | +-- transformers@2.1.0
[build-step-build] | | +-- css@1.0.8
[build-step-build] | | | +-- css-parse@1.0.4
[build-step-build] | | | `-- css-stringify@1.0.5
[build-step-build] | | +-- promise@2.0.0
[build-step-build] | | | `-- is-promise@1.0.1
[build-step-build] | | `-- uglify-js@2.2.5
[build-step-build] | |   +-- optimist@0.3.7
[build-step-build] | |   | `-- wordwrap@0.0.3
[build-step-build] | |   `-- source-map@0.1.43
[build-step-build] | +-- uglify-js@2.8.29
[build-step-build] | | +-- source-map@0.5.7
[build-step-build] | | +-- uglify-to-browserify@1.0.2
[build-step-build] | | `-- yargs@3.10.0
[build-step-build] | |   +-- camelcase@1.2.1
[build-step-build] | |   +-- cliui@2.1.0
[build-step-build] | |   | +-- center-align@0.1.3
[build-step-build] | |   | | +-- align-text@0.1.4
[build-step-build] | |   | | | +-- kind-of@3.2.2
[build-step-build] | |   | | | | `-- is-buffer@1.1.6
[build-step-build] | |   | | | +-- longest@1.0.1
[build-step-build] | |   | | | `-- repeat-string@1.6.1
[build-step-build] | |   | | `-- lazy-cache@1.0.4
[build-step-build] | |   | +-- right-align@0.1.3
[build-step-build] | |   | `-- wordwrap@0.0.2
[build-step-build] | |   `-- window-size@0.1.0
[build-step-build] | +-- void-elements@2.0.1
[build-step-build] | `-- with@4.0.3
[build-step-build] |   +-- acorn@1.2.2
[build-step-build] |   `-- acorn-globals@1.0.9
[build-step-build] `-- nconf@0.8.5
[build-step-build]   +-- async@1.5.2
[build-step-build]   +-- ini@1.3.5
[build-step-build]   +-- secure-keys@1.0.0
[build-step-build]   `-- yargs@3.32.0
[build-step-build]     +-- camelcase@2.1.1
[build-step-build]     +-- cliui@3.2.0
[build-step-build]     | +-- strip-ansi@3.0.1
[build-step-build]     | | `-- ansi-regex@2.1.1
[build-step-build]     | `-- wrap-ansi@2.1.0
[build-step-build]     +-- decamelize@1.2.0
[build-step-build]     +-- os-locale@1.4.0
[build-step-build]     | `-- lcid@1.0.0
[build-step-build]     |   `-- invert-kv@1.0.0
[build-step-build]     +-- string-width@1.0.2
[build-step-build]     | +-- code-point-at@1.1.0
[build-step-build]     | `-- is-fullwidth-code-point@1.0.0
[build-step-build]     |   `-- number-is-nan@1.0.1
[build-step-build]     +-- window-size@0.1.4
[build-step-build]     `-- y18n@3.2.1
[build-step-build]
[build-step-build]        Writing NODE_PATH to shared
[build-step-build]        Writing PATH to shared
[build-step-build] -----> cache 31cdf8958fc50703537518168b3bc65884a6c5497116aa8a8d77a20e39ab6bb5: Contributing to layer
[build-step-build] -----> Process types:
[build-step-build]        web: npm start
[build-step-build]
[build-step-build]
[build-step-export] Exporting layer 'app' with SHA sha256:d4cce8da12ac36ba584dd64d55ca5e828dc13707d18eea8b596bff75927fd3d4
[build-step-export] Exporting layer 'config' with SHA sha256:b616aa82930eb4d1360b048db518d51c45b86cc684c7200e9896267a81124e55
[build-step-export] Exporting layer 'launcher' with SHA sha256:2187c4179a3ddaae0e4ad2612c576b3b594927ba15dd610bbf720197209ceaa6
[build-step-export] Exporting layer 'org.cloudfoundry.buildpacks.nodejs:node' with SHA sha256:01a665b0d2123c870527835625b05935d6f95dcdcf614e07f6e8375c653dd69c
[build-step-export] Exporting layer 'org.cloudfoundry.buildpacks.npm:node_modules' with SHA sha256:f91092a7723e6a573564c673d32aea089deb1c3bd547c2f2888ed3f90853ed23
[build-step-export] *** Images:
[build-step-export]       gcr.io/sample-registry/test/build:latest - succeeded
[build-step-export]       gcr.io/sample-registry/test/build:b3.20190708.215527 - succeeded
[build-step-export]
[build-step-export] *** Digest: sha256:f5a1725fa5a29421870334ad7a16bad1492130c16febb43b75f84d0655b53141
[build-step-export]
[build-step-cache] Caching layer 'org.cloudfoundry.buildpacks.nodejs:7f26cd9a2845df23773755a428d61b74fd80d48a991e964d12e85ae90ced81a0' with SHA sha256:2e6bf47b26fd9ca6a308dd680b6bc2b201ef6e860c1e54e34fb387455f6bee98
[build-step-cache] Caching layer 'org.cloudfoundry.buildpacks.nodejs:node' with SHA sha256:01a665b0d2123c870527835625b05935d6f95dcdcf614e07f6e8375c653dd69c
[build-step-cache] Caching layer 'org.cloudfoundry.buildpacks.npm:cache' with SHA sha256:01aa7d0778ae3900f69ded6a8e0b3b27c906fdf357a9eeef0b6abd052659942c
[build-step-cache] Caching layer 'org.cloudfoundry.buildpacks.npm:node_modules' with SHA sha256:f91092a7723e6a573564c673d32aea089deb1c3bd547c2f2888ed3f90853ed23
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