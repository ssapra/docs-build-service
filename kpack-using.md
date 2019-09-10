This topic describes how to get started with kpack, a collection of open source resource controllers and CRDs that contain declarative build logic.  The following documentation assumes the reader has installed kpack as well as the following pre-requirements 

- Kubernetes cluster
- `kubectl` cli
- Docker V2 Registry




### <a id='configure-cli'></a> Creating an Image Resource

There are several resource types that will work together to produce an image resource.  Below, we will review how to configure and create each resource. Note that the yaml resources contain a `name` field. Users should reference these names when configuring other resource types. In order to apply a particular resource type, use `kubectl apply -f ~/path/to/resource.yml`

1) Creating a builder resource

Prior to using kpack to execute builds, the user must first create a "builder" resource. Builder images contain a list of buildpacks and their corresponding versions and reference to a `stack` image. kpack will utilize these inputs to execute builds.  

In the resource template provided below, we include a reference to the `cloudfoundry/cnb:bionic` image, which is based on Ubuntu Bionic. This builder image lives in a registry, and kpack will execute rebuilds when the image is updated with new buildpacks. You will reference the `name` of this builder resource when configuring the image resource (step 5)

    ```yaml
    apiVersion: build.pivotal.io/v1alpha1
    kind: Builder
    metadata:
      name: sample-builder
    spec:
      image: cloudfoundry/cnb:bionic
      imagePullSecrets: # optional, if not set builder must be public
      - name: builder-secret
    ```

2) Create a secret so that kpack can push image builds to your desired registry. The `name` of this credential will be referenced to create a service account (step 4).   

   1. GCR example
      ```yaml
      apiVersion: v1
      kind: Secret
      metadata:
        name: basic-docker-user-pass
        annotations:
          build.pivotal.io/docker: gcr.io
      type: kubernetes.io/basic-auth
      stringData:
        username: <username>
        password: <password>
      ```

    1. Docker Hub example
        ```yaml
        apiVersion: v1
        kind: Secret
        metadata:
          name: basic-docker-user-pass
          annotations:
            build.pivotal.io/docker: index.docker.io
        type: kubernetes.io/basic-auth
        stringData:
          username: <username>
          password: <password>
        ```
3) Create a secret for pull access from the desired git repository. The example below is for a github repository.  You can also specify a personal access token.  If you are building against source code that lives in a public registry, you do not need to configure a git secret.  The `name` of this credential will be referenced to create a service account (step 4).

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: basic-git-user-pass
      annotations:
        build.pivotal.io/git: https://github.com
    type: kubernetes.io/basic-auth
    stringData:
      username: <username>
      password: <password>
    ```

4) Create a service account that uses the docker registry secret and the git repository secret. When configuring the image resource, reference the `name` of your registry credential and the `name` of your git credential.  

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: service-account
    secrets:
      - name: basic-docker-user-pass
      - name: basic-git-user-pass

5. Apply an image configuration to the cluster. In addition to specifying the source code url and any build time environment variable names and values your app needs, users can also exercise additional control over how kpack executes builds. Users can do this by specifying cache size, build history limit, and resources used.  

    If you would like to build an image from a git repo:
 
    ```yaml
    apiVersion: build.pivotal.io/v1alpha1
    kind: Image
    metadata:
      name: sample-image
    spec:
      tag: gcr.io/project-name/app
      serviceAccount: service-account
      builderRef: sample-builder
      cacheSize: "1.5Gi" # Optional, if not set then the caching feature is disabled
      failedBuildHistoryLimit: 5 # Optional, if not present defaults to 10
      successBuildHistoryLimit: 5 # Optional, if not present defaults to 10
      source:
        git:
          url: https://github.com/buildpack/sample-java-app.git
          revision: master
      build: # Optional
        env:
          - name: BP_JAVA_VERSION
            value: 8.*
        resources:
          limits:
            cpu: 100m
            memory: 1G
          requests:
            cpu: 50m
            memory: 512M
    ```

    If you would like to build an image from an hosted zip or jar:
 
    ```yaml
    apiVersion: build.pivotal.io/v1alpha1
    kind: Image
    metadata:
      name: sample-image
    spec:
      tag: gcr.io/project-name/app
      serviceAccount: service-account
      builderRef: sample-builder
      cacheSize: "1.5Gi" # Optional, if not set then the caching feature is disabled
      failedBuildHistoryLimit: 5 # Optional, if not present defaults to 10
      successBuildHistoryLimit: 5 # Optional, if not present defaults to 10
      source:
        blob:
          url: https://storage.googleapis.com/build-service/sample-apps/spring-petclinic-2.1.0.BUILD-SNAPSHOT.jar
      build: # Optional
        env:
          - name: BP_JAVA_VERSION
            value: 8.*
        resources:
          limits:
            cpu: 100m
            memory: 1G
          requests:
            cpu: 50m
            memory: 512M
    ```

### <a id='configure-cli'></a> Helpful commands 

**See the builds for the image**

  ```builds
    kubectl get cnbbuilds # before the first builds completes you will see a unknown (building) status
    ---------------
    NAME                          SHA   SUCCEEDED
    sample-image-build-1-ea3e6fa9         Unknown  

    ```

After a build has completed you will be able to see the built digest

**Tailing Logs from Builds**

Use the log tailing utility in `cmd/logs`

```bash
go build ./cmd/logs
```

The logs tool allows you to view the logs for all builds for an image: 

```bash
./logs -image <IMAGE-NAME>
```

To view logs from a specific build use the build flag:  

```bash
./logs -image <IMAGE-NAME> -build <BUILD-NUMBER>
```



## Local Development Install

Access to a Kubernetes cluster is needed in order to install the kpack controllers.

```bash
kubectl cluster-info # ensure you have access to a cluster
./hack/apply.sh <IMAGE/NAME> # <IMAGE/NAME> is a writable and publicly accessible location 
```

If generation of code is needed you should run the following commands:
```bash
./hack/update-codegen.sh
```

### Running Tests

* To run the e2e tests, kpack must be installed and running on a cluster
* The IMAGE_REGISTRY environment variable must point at a registry with local write access 

    ```bash
    IMAGE_REGISTRY=gcr.io/<some-project> go test ./...
    ```

 

   
