# Cluster Authentication Task

This task creates a [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
file that can be used to configure access to the different clusters.
A common use case for this task is to deploy your `application/function` on different clusters.

The task will use the provided parameters to create a `kubeconfig` file that can be used by other steps
in the pipeline task to access the target cluster. The kubeconfig will be placed at 
`/workspace/kubeconfigFile/kubeconfig`.

This task provides users variety of ways to authenticate:
- Authenticate using tokens.
- Authenticate using client key and client certificates.


## Install the Task

```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/v1beta1/cluster/cluster-authentication.yaml
```

## Inputs

### Parameters

* **Name**: Name of the `cluster`.
* **Type**: Type of the resource (_default:_
  `cluster`)
* **URL**: Address of the target cluster (_e.g.:_ 
  `https://hostname:port`)
* **Username**: Username for basic authentication to the cluster
(_default:_ `""`)
* **Password**: Password for basic authentication to the cluster
(_default:_ `""`)
* **Cadata**: Contains PEM-encoded certificate authority certificates
(_default:_ `""`)
* **ClientKeyData**: Contains PEM-encoded data from a client key file for TLS
(_default:_ `""`)
* **ClientCertificateData**: Contains PEM-encoded data from a client cert file for TLS 
(_default:_ `""`)
* **Namespace**: Default namespace to use on unspecified requests
(_default:_ `""`)
* **Token**: Bearer token for authentication to the cluster
(_default:_ `""`)
* **Insecure**:  If true, skips the validity check for the server's certificate. 
This will make your HTTPS connections insecure
(_default:_ `true`)


## Usage

This task uses a `shared workspace` with [`PVC`](https://kubernetes.io/docs/concepts/storage/persistent-volumes). 
Kubeconfig file is written at `/workspace/<your-cluster-name>/kubeconfig` 
and then copies this file to the shared workspace in the `kubeconfigFile` directory.

Task can be used with the other task in the pipeline to authenticate the cluster.
In this example, pipeline has a task `authenticate-cluster` that generates a 
`kubeconfig file` for the cluster and the `test-task` uses that kubeconfig file and verifiy whether the
application has an access to the cluster or not by using some `kubectl/oc` commands.

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: cluster-authentication-pipeline
spec:
  workspaces:
    - name: shared-workspace
  tasks:
    - name: cluster-authentication
      taskRef:
        name: cluster-authentication
      workspaces:
        - name: kubeconfigFile
          workspace: shared-workspace
      params:
        - name: name
          value: cluster-bot
        - name: username
          value: admin
        - name: url
          value: https://api.ci-ln-dgx7m7b-d5d6b.origin-ci-int-aws.dev.rhcloud.com:6443
        - name: clientCertificateData
          value: LS0tLS1....
        - name: clientKeyData
          value: LS0tLS1....
    - name: authentication-test
      taskRef:
        name: authentication-test
      workspaces:
        - name: kubeconfigFile
          workspace: shared-workspace
      params:
        - name: filename
          value: kubeconfig
      runAfter:
        - cluster-authentication
```


`Test-task` uses shared-workspace to fetch the kubeconfig file form the
kubeconfigFile directory and uses oc commands to check whether the `cluster` is configured or not.

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: authentication-test
spec:
  params:
    - name: filename
      description: kubeconfig file name
      type: string
  workspaces:
    - name: kubeconfigFile
      readOnly: true
  steps:
    - name: get
      image: quay.io/openshift/origin-cli:latest
      script: |

        export KUBECONFIG="$(workspaces.kubeconfigFile.path)/$(inputs.params.filename)"

        #check that the cluster is configured
        oc get pods
```

Workspace with `PVC` is used, as shown below.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kubeconfig-pvc
spec:
  resources:
    requests:
      storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
   ```
   
 Finally, Pipelinerun is used to execute the tasks in the pipeline and get the results.
 ```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: cluster-authentication-pipeline-run
spec:
  pipelineRef:
    name: cluster-authentication-pipeline
  workspaces:
    - name: shared-workspace
      persistentvolumeclaim:
        claimName: kubeconfig-pvc

```
   
