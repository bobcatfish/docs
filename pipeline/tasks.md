# Tasks

This document defines `Tasks` and their capabilities.

---

- [What is a Task?](#what-is-a-task)
- [Syntax](#syntax)
  - [Steps](#steps)
  - [Inputs](#inputs)
  - [Outputs](#outputs)
  - [Service Account](#service-account)
  - [Volumes](#volumes)
  - [Templating](#templating)
- [Examples](#examples)

## What is a Task?

A `Task` (or a `ClusterTask`) is a collection of sequential steps you would want to run as part of
your continuous integration flow. A task will run inside a container on your
cluster.

A `Task` declares:

- [Inputs](#inputs)
- [Outputs](#outputs)
- [Steps](#steps)

A `Task` is available within a namespace, and `ClusterTask` is available across entire Kubernetes cluster.

A `Task` functions exactly like a `ClusterTask`, and as such all references to `Task` below are also describing `ClusterTask`.

## Syntax

To define a configuration file for a `Task` resource, you can specify the
following fields:

- Required:
  - [`apiVersion`][kubernetes-overview] - Specifies the API version, for example
    `pipeline.knative.dev/v1alpha1`.
  - [`kind`][kubernetes-overview] - Specify the `Task` resource object.
  - [`metadata`][kubernetes-overview] - Specifies data to uniquely identify the
    `Task` resource object, for example a `name`.
  - [`spec`][kubernetes-overview] - Specifies the configuration information for
    your `Task` resource object. Build steps must be defined through either of
    the following fields:
    - [`steps`](#steps) - Specifies one or more container images that you want
      to run in your `Task`.
- Optional:
  - [`inputs`](#inputs) - Specifies parameters and [`PipelineResources`](resources.md)
    needed by your `Task`
  - [`outputs`](#outputs) - Specifies [`PipelineResources`](resources.md) needed by your `Task`
  - [`serviceAccount`](#service-account) - Specifies a `ServiceAccount`
    resource object that enables your build to run with the defined
    authentication information.
  - [`volumes`](#volumes) - Specifies one or more volumes that you want to make
    available to your build.
  - [`timeout`](#timeout) - Specifies timeout after which the Task will fail.

[kubernetes-overview]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#required-fields

The following example is a non-working sample where most of the possible
configuration fields are used:

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Task
metadata:
  name: example-task-name
spec:
  serviceAccountName: task-auth-example
  source:
    git:
      url: https://github.com/example/build-example.git
      revision: master
  inputs:
    resources:
    - name: workspace
      type: git
    params:
    - name: pathToDockerFile
      description: The path to the dockerfile to build
      default: /workspace/workspace/Dockerfile
  outputs:
    resources:
    - name: builtImage
      type: image
  steps:
  - name: ubuntu-example
    image: ubuntu
    args: ["ubuntu-build-example", "SECRETS-example.md"]
  steps:
  - image: gcr.io/example-builders/build-example
    args: ['echo', '${inputs.resources.params.pathToDockerFile}']
  steps:
  - name: dockerfile-pushexample
    image: gcr.io/example-builders/push-example
    args: ["push", "${outputs.resources.builtImage.url}"]
    volumeMounts:
    - name: docker-socket-example
      mountPath: /var/run/docker.sock
  volumes:
  - name: example-volume
    emptyDir: {}
```

### Steps

The `steps` field is required. You define one or more `steps` fields to define
the body of a `Task`.

Each `steps` in a `Task` must specify a container image that adheres to the
[container contract](./container-contract.md). For each of the `steps` fields,
or container images that you define:

- The container images are run and evaluated in order, starting
  from the top of the configuration file.
- Each container image runs until completion or until the first failure is
  detected.

### Inputs

A `Task` can declare the inputs it needs, which can be either or both of:

* [`parameters`](#parameters)
* [`input resources](#input-resources)

#### Parameters

Tasks can declare input parameters that must be supplied to the task during a
TaskRun. Some example use-cases of this include:

- A Task that needs to know what compilation flags to use when building an
  application.
- A Task that needs to know what to name a built artifact.
- A Task that supports several different strategies, and leaves the choice up to
  the other.

##### Usage

The following example shows how Tasks can be parameterized, and these parameters
can be passed to the `Task` from a `TaskRun`.

Input parameters in the form of `${inputs.params.foo}` are replaced inside of
the [`steps`](#steps) (see also [templating](#templating)).

The following `Task` declares an input parameter called 'flags', and uses it in
the `steps.args` list.

```yaml
apiVersion: pipeline.knative.dev/v1alpha1
kind: Task
metadata:
  name: task-with-parameters
spec:
  inputs:
    params:
      - name: flags
        value: string
  steps:
    - name: build
      image: my-builder
      args: ["build", "--flags=${inputs.params.flags}"]
```

The following `TaskRun` supplies a value for `flags`:

```yaml
apiVersion: pipeline.knative.dev/v1alpha1
kind: TaskRun
metadata:
  name: run-with-parameters
spec:
  taskRef:
    name: task-with-parameters
  inputs:
    params:
      - name: "flags"
        value: "foo=bar,baz=bat"
```

#### Input resources

Use input [`PipelineResources`](resources.md) field to provide your
`Task` with data or context that is needed by your `Task`.

Input resources, like source code (git) or artifacts, are dumped at path
`/workspace/task_resource_name` within a mounted
[volume](https://kubernetes.io/docs/concepts/storage/volumes/)
and is available to all [`steps`](#steps) of your `Task`. The path that the
resources are mounted at can be overridden with the `targetPath` value.

### Outputs

Output [`PipelineResources`](resources.md), are expected in directory path
`/workspace/output/resource_name`.

- If resource has an output "action" like upload to blob storage, then the
  container step is added for this action.
- If there is PVC volume present (TaskRun holds owner reference to PipelineRun)
  then copy step is added as well.

- If the resource is declared only in output but not in input for task then the
  copy step includes resource being copied to PVC to path
  `/pvc/task_name/resource_name` from `/workspace/output/resource_name` like the
  following example.

  ```yaml
  kind: Task
  metadata:
    name: get-gcs-task
    namespace: default
  spec:
    outputs:
      resources:
        - name: gcs-workspace
          type: storage
  ```

- If the resource is declared both in input and output for task the then copy
  step includes resource being copied to PVC to path
  `/pvc/task_name/resource_name` from `/workspace/random-space/` if input
  resource has custom target directory (`random-space`) declared like the
  following example.

  ```yaml
  kind: Task
  metadata:
    name: get-gcs-task
    namespace: default
  spec:
    inputs:
      resources:
        - name: gcs-workspace
          type: storage
          targetPath: random-space
    outputs:
      resources:
        - name: gcs-workspace
          type: storage
  ```

  - If resource is declared both in input and output for task without custom
    target directory then copy step includes resource being copied to PVC to
    path `/pvc/task_name/resource_name` from `/workspace/random-space/` like the
    following example.

  ```yaml
  kind: Task
  metadata:
    name: get-gcs-task
    namespace: default
  spec:
    inputs:
      resources:
        - name: gcs-workspace
          type: storage
    outputs:
      resources:
        - name: gcs-workspace
          type: storage
  ```

### Service Account

Specifies the `name` of a `ServiceAccount` resource object. Use the
`serviceAccount` field to run your `Task` with the privileges of the
specified service account. If no `serviceAccount` field is specified, your
`Task` runs using the
[`default` service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server)
that is in the
[namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
of the `Task` resource object.

For examples and more information about specifying service accounts, see the
[`ServiceAccount`](./auth.md) reference topic.

### Volumes

Optional. Specifies one or more
[volumes](https://kubernetes.io/docs/concepts/storage/volumes/) that you want to
make available to your build, including all the build steps. Add volumes to
complement the volumes that are implicitly
[created during a build step](./builder-contract.md).

For example, use volumes to accomplish one of the following common tasks:

- [Mount a Kubernetes secret](./auth.md).

- Create an `emptyDir` volume to act as a cache for use across multiple build
  steps. Consider using a persistent volume for inter-build caching.

- Mount a host's Docker socket to use a `Dockerfile` for container image builds.

### Templating

## Examples

Use these code snippets to help you understand how to define your Knative
builds.

Tip: See the collection of simple
[test builds](https://github.com/knative/build/tree/master/test) for additional
code samples, including working copies of the following snippets:

- [`git` as `source`](#using-git)
- [`gcs` as `source`](#using-gcs)
- [`custom` as `source`](#using-custom)
- [Mounting extra volumes](#using-an-extra-volume)
- [Pushing an image](#using-steps-to-push-images)
- [Authenticating with `ServiceAccount`](#using-a-serviceaccount)
- [Timeout](#using-timeout)

#### Using `git`

Specifying `git` as your `source`:

```yaml
spec:
  source:
    git:
      url: https://github.com/knative/build.git
      revision: master
  steps:
    - image: ubuntu
      args: ["cat", "README.md"]
```

#### Using `gcs`

Specifying `gcs` as your `source`:

```yaml
spec:
  source:
    gcs:
      type: Archive
      location: gs://build-crd-tests/rules_docker-master.zip
  steps:
    - name: list-files
      image: ubuntu:latest
      args: ["ls"]
```

#### Using `custom`

Specifying `custom` as your `source`:

```yaml
spec:
  source:
    custom:
      image: gcr.io/cloud-builders/gsutil
      args: ["rsync", "gs://some-bucket", "."]
  steps:
    - image: ubuntu
      args: ["cat", "README.md"]
```

#### Using an extra volume

Mounting multiple volumes:

```yaml
spec:
  steps:
    - image: ubuntu
      entrypoint: ["bash"]
      args: ["-c", "curl https://foo.com > /var/my-volume"]
      volumeMounts:
        - name: my-volume
          mountPath: /var/my-volume

    - image: ubuntu
      args: ["cat", "/etc/my-volume"]
      volumeMounts:
        - name: my-volume
          mountPath: /etc/my-volume

  volumes:
    - name: my-volume
      emptyDir: {}
```

#### Using `steps` to push images

Defining a `steps` to push a container image to a repository.

```yaml
spec:
  parameters:
    - name: IMAGE
      description: The name of the image to push
    - name: DOCKERFILE
      description: Path to the Dockerfile to build.
      default: /workspace/Dockerfile
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor
      args:
        - --dockerfile=${DOCKERFILE}
        - --destination=${IMAGE}
```

#### Using a `ServiceAccount`

Specifying a `ServiceAccount` to access a private `git` repository:

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: test-build-with-serviceaccount-git-ssh
  labels:
    expect: succeeded
spec:
  serviceAccountName: test-build-robot-git-ssh
  source:
    git:
      url: git@github.com:knative/build.git
      revision: master

  steps:
    - name: config
      image: ubuntu
      command: ["/bin/bash"]
      args: ["-c", "cat README.md"]
```

Where `serviceAccountName: test-build-robot-git-ssh` references the following
`ServiceAccount`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-build-robot-git-ssh
secrets:
  - name: test-git-ssh
```

And `name: test-git-ssh`, references the following `Secret`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-git-ssh
  annotations:
    build.knative.dev/git-0: github.com
type: kubernetes.io/ssh-auth
data:
  # Generated by:
  # cat id_rsa | base64 -w 0
  ssh-privatekey: LS0tLS1CRUdJTiBSU0EgUFJJVk.....[example]
  # Generated by:
  # ssh-keyscan github.com | base64 -w 0
  known_hosts: Z2l0aHViLmNvbSBzc2g.....[example]
```

Note: For a working copy of this `ServiceAccount` example, see the
[build/test/git-ssh](https://github.com/knative/build/tree/master/test/git-ssh)
code sample.

#### Using `timeout`

Specifying `timeout` for your `build`:

```yaml
spec:
  timeout: 20m
  source:
    git:
      url: https://github.com/knative/build.git
      revision: master
  steps:
    - image: ubuntu
      args: ["cat", "README.md"]
```

### Example Task

For example, a `Task` to encapsulate a `Dockerfile` build might look
something like this:

**Note:** Building a container image using `docker build` on-cluster is _very
unsafe_. Use [kaniko](https://github.com/GoogleContainerTools/kaniko) instead.
This is used only for the purposes of demonstration.

```yaml
spec:
  params:
    # This has no default, and is therefore required.
    - name: IMAGE
      description: Where to publish the resulting image.

    # These may be overridden, but provide sensible defaults.
    - name: DIRECTORY
      description: The directory containing the build context.
      default: /workspace
    - name: DOCKERFILE_NAME
      description: The name of the Dockerfile
      default: Dockerfile

  steps:
    - name: dockerfile-build
      image: gcr.io/cloud-builders/docker
      workingDir: "${DIRECTORY}"
      args:
        [
          "build",
          "--no-cache",
          "--tag",
          "${IMAGE}",
          "--file",
          "${DOCKERFILE_NAME}",
          ".",
        ]
      volumeMounts:
        - name: docker-socket
          mountPath: /var/run/docker.sock

    - name: dockerfile-push
      image: gcr.io/cloud-builders/docker
      args: ["push", "${IMAGE}"]
      volumeMounts:
        - name: docker-socket
          mountPath: /var/run/docker.sock

  # As an implementation detail, this template mounts the host's daemon socket.
  volumes:
    - name: docker-socket
      hostPath:
        path: /var/run/docker.sock
        type: Socket
```

In this example, `params` describes arguments for the `Task`.
The `description` is used for diagnostic messages during validation (and maybe
in the future for UI). The `default` value enables a template to have a
graduated complexity, where options are overridden only when the user strays
from some set of sane defaults.

### Example TaskRuns

For the sake of illustrating re-use, here are several example [`TaskRuns`](taskrun.md)
(including referenced [`PipelineResources`](resource.md)) instantiating the `Task` above
(`dockerfile-build-and-push`).

Build `mchmarny/rester-tester`:

```yaml
# The PipelineResource
metadata:
  name: mchmarny-repo
spec:        
  type: git
  params:
  - name: url
    value: https://github.com/mchmarny/rester-tester.git
```

```yaml
# The TaskRun
spec:
  taskRef:
    name: dockerfile-build-and-push
  inputs:
    resources:
      - name: workspace
        resourceRef:
          name: mchmarny-repo
    params:
    - name: IMAGE
      value: gcr.io/my-project/rester-tester
```

Build `googlecloudplatform/cloud-builder`'s `wget` builder:

```yaml
# The PipelineResource
metadata:
  name: cloud-builder-repo
spec:        
  type: git
  params:
  - name: url
    value: https://github.com/googlecloudplatform/cloud-builders.git
```

```yaml
# The TaskRun
spec:
  taskRef:
    name: dockerfile-build-and-push
  inputs:
    resources:
      - name: workspace
        resourceRef:
          name: cloud-builder-repo
    params:
      - name: IMAGE
        value: gcr.io/my-project/wget
      # Optional override to specify the subdirectory containing the Dockerfile
      - name: DIRECTORY
        value: /workspace/wget
```

Build `googlecloudplatform/cloud-builder`'s `docker` builder with `17.06.1`:

```yaml
# The PipelineResource
metadata:
  name: cloud-builder-repo
spec:        
  type: git
  params:
  - name: url
    value: https://github.com/googlecloudplatform/cloud-builders.git
```

```yaml
# The TaskRun
spec:
  taskRef:
    name: dockerfile-build-and-push
  inputs:
    resources:
      - name: workspace
        resourceRef:
          name: cloud-builder-repo
    params:
      - name: IMAGE
        value: gcr.io/my-project/docker
      # Optional overrides
      - name: DIRECTORY
        value: /workspace/docker
      - name: DOCKERFILE_NAME
        value: Dockerfile-17.06.1
```

---

Except as otherwise noted, the content of this page is licensed under the
[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/),
and code samples are licensed under the
[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0).
