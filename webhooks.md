---
title: Setting up Builder Webhooks  
owner: Partners
---

<strong><%= modified_date %></strong>

The Build Service polls the image registry to know when buildpacks have been updated. This allows the build service to
to rebuild images when updated buildpacks are available. 

Webhooks can be used instead of polling. This can be used to avoiding the polling delay when a builder has been updated.

## <a id='supported-image-registries'></a> Supported image registries

### Artifactory 

Follow the installation instructions for the [Artifactory Webhook Plugin](https://github.com/jfrog/artifactory-user-plugins/tree/master/webhook).

The plugin should be configured to send `docker.tagCreated` events to the <BUILD-SERVICE-API>/v1/artifactory/webhook endpoint.

An example arfifactory webhook configuration is provided below: 

```yaml
{
  "webhooks": {
    "ci": {
      "url": "<BUILD-SERVICE-API>/v1/artifactory/webhook",
      "events": [
        "docker.tagCreated"
      ],
      "repositories": ["<ARTIFACTORY-REPOSITORY>"],
      "format": "default"
    },
  }
}
```

**Note:** Make sure the configured artifactory repository is the the one used in `duffle relocate`   
    
### Docker Registry Notifications 

If you are using the [docker registry](https://docs.docker.com/registry/) you can setup registry notifications with the build service.

Follow the [configuration instructions for docker notifications](https://docs.docker.com/registry/notifications/).

You will need to add a notification endpoint to your registry configuration. An example is provided below:

```yaml
 notifications:
    endpoints:
      - name: build-service
        url: <BUILD-SERVICE-API>/v1/docker/notification
        timeout: 5s
        threshold: 5
        backoff: 10s
```   
**Note:** Make sure the configured docker registry is the one used in `duffle relocate`   
    

### Dockerhub

If you are using the [Docker Hub](https://hub.docker.com/) you can setup webhooks with the build service.

Follow the [Dockerhub Hub Webhook Instructions](https://docs.docker.com/registry/notifications/).

You will need to add a webhook for the relocated builder image. The name of this image can be easily found with:

```bash
kubectl get builder build-service-builder -n build-service-builds -o jsonpath='{.spec.image}'
```
  
The webhook should be configured to use the <BUILD-SERVICE-API>/v1/docker/webhook endpoint.

**Warning:** Docker Hub requires the build service to use a CA-Signed Certificate.
