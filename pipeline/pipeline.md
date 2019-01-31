### Resource sharing between tasks

Pipeline `Tasks` are allowed to pass resources from previous `Tasks` via the
[`from`](#from) field. This feature is implemented using the two
following alternatives:

- Persistent Volume Claims under the hood but however has an implication
  that tasks cannot have any volume mounted under path `/pvc`.

- [GCS storage bucket](https://cloud.google.com/storage/docs/json_api/v1/buckets)
  A storage bucket can be configured using a ConfigMap named [`config-artifact-bucket`](./../config/config-artifact-bucket.yaml). 
  with the following attributes:
- `location`: the address of the bucket (for example gs://mybucket)
- `bucket.service.account.secret.name`: the name of the secret that will contain the credentials for the service account
  with access to the bucket
- `bucket.service.account.secret.key`: the key in the secret with the required service account json
The bucket is configured with a retention policy of 24 hours after which files will be deleted


### How are resources shared between tasks?

`PipelineRun` uses PVC to share resources between tasks. PVC volume is mounted
on path `/pvc` by PipelineRun.

- If a resource in a task is declared as output then the `TaskRun` controller
  adds a step to copy each output resource to the directory path
  `/pvc/task_name/resource_name`.

- If an input resource includes `from` condition then the `TaskRun` controller
  adds a step to copy from PVC to directory path
  `/pvc/previous_task/resource_name`.

Another alternatives is to use a GCS storage bucket to share the artifacts. This can 
be configured using a ConfigMap with the name `config-artifact-bucket` with the following attributes:

- location: the address of the bucket (for example gs://mybucket)
- bucket.service.account.secret.name: the name of the secret that will contain the credentials for the service account
  with access to the bucket
- bucket.service.account.secret.key: the key in the secret with the required service account json.
  The bucket is recommended to be configured with a retention policy after which files will be deleted.

Both options provide the same functionality to the pipeline. The choice is based on the infrastructure used,
for example in some Kubernetes platforms, the creation of a persistent volume could be slower than 
uploading/downloading files to a bucket, or if the the cluster is running in multiple zones, the access to
the persistent volume can fail.