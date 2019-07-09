<h2> Using Build Service </h2>

**last modiefied 7/16/2019**

**Note:** This file assumes that the user has read installing.md and has deployed all the build service bits included in the CNAB bundle, installed the `pb` CLI, and deployed build service with CA certs necessary to communicate with image registries.

<h3> Declarative vs Imperative </h3>

Understanding the differences between these two models as they relate to producing container images from source code is key to deriving the most value possible from Pivotal Build Service.  Pivotal Application Service, based on Cloud Foundry, currently follows an imperative model.  The user instructs the platform to perform various tasks through the `cf push` model.  Similarly, the OSS [Cloud Native Buildpacks project](http://buildpacks.io), which Build Service utilizes heavily for its build process, follows an imperative model.  Users perform commands like `build` and `rebase` using the `pack` CLI to trigger one-time execution of these commands.

The `pb` (Pivotal Build) CLI doesn’t have commands like `push`, `build`, or `rebase`.  Instead, users utilize `pb` to declare and modify two configurations.  

1) The “Image” config:  Users describe their app through several inputs, and build service will deliver new images that resolve discrepancies between how the app is described and the latest version of the image. When the user requests the latest version of an application dependency, and that dependency becomes available, a new image is automatically built to replace the old image.

2) The “Team” config:   An extension to build service that provides access control to image configs using UAA.

Now that we’ve covered the concepts at a high level, let’s look at some workflow examples.  Since creating an image config requires that build service knows about image registry credentials, we encourage platform operators to first apply a team configuration.  

<h3> “Team” Config </h3>

Create and populate a file we will call `team.yml` for the sake of this example.  In `team.yml`, the platform operator will name the team and specify registry credentials build service will need to deliver a stream of images over time.  Users can specify multiple registries and credentials per team.  A sample `team.yml` could look like this:  
```
$ cat team.yml

name: myteam
registries:
- registry: index.docker.io
  username: jon_user
  password: some_secret
```
To inform build service about this team and associate the specified registry credentials with the team, run `pb team apply -f ~/path/to/team.yml`

These team configurations can be updated by team owners (the UAA account that created the team) by editing the contents of `team.yml` and running `pb team apply -f ~/path/to/team.yml` to inform build service of the changes.  Team owners can also delete a team config if no image builds referencing that team name have been triggered.  

**Constraints:**

* Only one user per team
* Cannot reference UAA user groups
* Registry credentials must be specified in `team.yml`

As we release new versions of the Alpha and eventually deliver GA, we will dramatically improve this experience.  We plan to move registry credential configuration into the CLI experience, allow multiple users per team, utilize UAA user groups, introduce user roles (eg. Admins).  

<h3>"Image Config”</h3>

**Note:**  The below instructions assume that a user has already applied a team through the above workflow.

Create and populate a file we will call `image.yml` for the sake of this example.  In `image.yml`, users specify the name of a team, location of source code, a git commit or branch, and preferred image registry location (utilizes credentials declared in “team.yml”).  A sample `image.yml` could look like this:  
```
$ cat image.yml

name: myteam
registries:
- registry: index.docker.io
  username: john_user
  password: some_secret
```

A user declares the image config by running `pb image apply -f ~/path/to/image.yml`.  Applying a new image config will kick off an initial build.   Before digging into build observability and logging, let’s first review the various triggers that will cause build service to rebuild images.  For the alpha, these triggers are:

* New buildpack versions are made available through a new builder image
* New source code detected
* New commit on a branch build service is listening to
* Changing the image config by modifying `image.yml` and running `pb image apply -f ~/path/to/image.yml`
    * Examples of these actions include:  specifying a different commit, branch, or git repo  

**Constraints:**

* Users can only specify source code that lives in a git repo  
* In subsequent releases, users will be able to create an image config using local source code
* The Build Service Alpha does not rebuild images based on new OS packages (like cflinuxfs3) 
    * Subsequent releases of build service will rebuild images based on new OS package images and rebase of vulnerable rootfs layers of deployed apps

**Build Observability:**  

To access historical information about image builds executed by build service, run `pb image <image tag>`.  Here is a of what the terminal output would look like.  

```
Build     Status    Image           Started Time               Finished Time
---------------------------------------------------------------------------------------------
1       SUCCESS   sha256:a2b5d2   2019-04-15 03:25:12        2019-04-15...                                  
2       FAILED    --              2019-04-16 04:36:12        2019-04-16...        
--      BUILDING  --              2019-04-17 06:01:07        --
--      PENDING   --              --                         --
```

To retrieve detailed cluster logs about a specific build, run  
```
pb image logs <image-tag>  [flags]

Flags:
  -b, --build string   build ID (required)
  -f, --follow         follow the logs for the build
```

**Constraints:**

For now, Build Service only stores the most recent 10 successful and 10 most recent failed builds.  Subsequent releases will include much deeper build histories, thereby enabling users to audit builds executed by build service.  

  
