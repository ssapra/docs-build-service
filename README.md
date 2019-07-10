# Pivotal Build Service

Build Service is a source code to container (S2I) solution that utilizes the OSS [Cloud Native Buildpacks project](http://buildpacks.io) to execute reproducible builds that adhere to [modern container standards](
https://github.com/opencontainers/image-spec/blob/master/spec.md).  Additionally, Build Service employs declarative logic to automate image re-builds.

Understanding the differences between declarative and imperative programming models as they relate to producing container images from source code is key to deriving the most value possible from Build Service.  Pivotal Application Service, based on Cloud Foundry, currently follows an imperative model.  The user instructs the platform to perform various tasks through the `cf push` model.  Similarly, the [Cloud Native Buildpacks project](http://buildpacks.io), follows an imperative model.  Users perform commands like `build` and `rebase` using the `pack` CLI to trigger one-time execution of these commands.

The `pb` (Pivotal Build) CLI doesn’t have commands like `push`, `build`, or `rebase`.  Instead, users utilize `pb` to declare and modify two configurations.  

1) The “Image” config:  Users describe their app through several inputs, and build service will deliver new images that resolve discrepancies between how the app is described and the latest version of the image. When the user requests the latest version of an application dependency, and that dependency becomes available, a new image is automatically built.

2) The “Team” config:   An extension to build service that provides access control to image configs using UAA.

Through utilizing these configs, platform engineers and application developers can more easily and securly enjoy the benefits of containerized software development at enterprise scale.  

## Helpful Links

* For more information on isntalling Build Service, please visit [Installing.md](https://github.com/pivotal-cf/docs-build-service/blob/master/installing.md)

* For more information on using the `pb` CLI to control build service, visit [Using.md](https://github.com/pivotal-cf/docs-build-service/blob/master/using.md)   

## Troubleshooting

Reach out to your Pivotal account team or reach out the Build Service team directly:

* Stephen Levine - Product Manager/Senior Software Engineer (slevine@pivotal.io)
* Matt Gibson - Product Manager (mgibson@pivotal.io)
* Matt McNew - Senior Software Engineer (mmcnew@pivotal.io)
