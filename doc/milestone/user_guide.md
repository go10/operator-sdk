# User Guide

This guide walks through an example of building a simple memcached-operator using tools and libraries provided by the Operator SDK.

## Prerequisites

- [dep][dep_tool] version v0.4.1+.
- [go][go_tool] version v1.10+.
- [docker][docker_tool] version 17.03+.
- [kubectl][kubectl_tool] version v1.9.0+.
- Access to a kubernetes v.1.9.0+ cluster.

**Note**: This guide uses [minikube][minikube_tool] version v0.25.0+ as the local kubernetes cluster and quay.io for the public registry.

## Install the Operator SDK CLI

The Operator SDK has a CLI tool that helps the developer to create, build, and deploy a new operator project.

Checkout the desired release tag and install the SDK CLI tool:
```
$ git checkout tags/v0.0.2
$ go install github.com/coreos/operator-sdk/commands/operator-sdk
```

This installs the CLI binary `operator-sdk` at `$GOPATH/bin`.

## Create a new project

Use the CLI to create a new memcached-operator project:

```
$ cd $GOPATH/src/github.com/example-inc/
$ operator-sdk new memcached-operator --api-version=cache.example.com/v1alpha1 --kind=Memcached
$ cd memcached-operator
```

This creates the memcached-operator project specifically for watching the Memcached resource with APIVersion `cache.example.com/v1apha1` and Kind `Memcached`.

To learn more about the project directory structure, see [project layout][layout_doc] doc.

## Customize the operator logic

For this example the memcached-operator will execute the following reconciliation logic for each `Memcached` CR:
- Create a memcached Deployment if it doesn't exist
- Ensure that the Deployment size is the same as specified by the `Memcached` CR spec
- Update the `Memcached` CR status with the names of the memcached pods

### Watch the Memcached CR

By default, the memcached-operator watches `Memcached` resource events as shown in `cmd/memcached-operator/main.go`.

```Go
func main() {
  sdk.Watch("cache.example.com/v1alpha1", "Memcached", "default", 5)
  sdk.Handle(stub.NewHandler())
  sdk.Run(context.TODO())
}
```

### Define the Memcached spec and status

Modify the spec and status of the `Memcached` CR at `pkg/apis/cache/v1alpha1/types.go`:

```Go
type MemcachedSpec struct {
	// Size is the size of the memcached deployment
	Size int32 `json:"size"`
}
type MemcachedStatus struct {
	// Nodes are the names of the memcached pods
	Nodes []string `json:"nodes"`
}
```
Update the generated code for the CR:

```
$ operator-sdk generate k8s
```

### Define the Handler

The reconciliation loop for an event is defined in the `Handle()` function at `pkg/stub/handler.go`.

Replace the default handler with the reference [memcached handler][memcached_handler] implementation.

> Note: The provided handler implementation is only meant to demonstrate the use of the SDK APIs and is not representative of the best practices of a reconciliation loop.

### Build and run the operator

Build the memcached-operator image and push it to a registry:
```
$ operator-sdk build quay.io/example/memcached-operator:v0.0.1
$ docker push quay.io/example/memcached-operator:v0.0.1
```

Kubernetes deployment manifests are generated in `deploy/operator.yaml`. The deployment image is set to the container image specified above.

Deploy the memcached-operator:

```
kubectl create -f deploy/rbac.yaml
kubectl create -f deploy/operator.yaml
```

Verify that the memcached-operator is up and running:

```
$ kubectl get deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
memcached-operator       1         1         1            1           1m
```

### Create a Memcached CR

Run the following command to create a `Memcached` custom resource:

```sh
$ cat <<EOF | kubectl apply -f -
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "example-memcached"
spec:
  size: 3
EOF
```

Ensure that the memcached-operator creates the deployment for the CR:

```
$ kubectl get deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
memcached-operator       1         1         1            1           2m
example-memcached        3         3         3            3           1m
```

Check the pods and CR status to confirm the status is updated with the memcached pod names:

```
$ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
example-memcached-6fd7c98d8-7dqdr     1/1       Running   0          1m
example-memcached-6fd7c98d8-g5k7v     1/1       Running   0          1m
example-memcached-6fd7c98d8-m7vn7     1/1       Running   0          1m
memcached-operator-7cc7cfdf86-vvjqk   1/1       Running   0          2m
```

```
$ kubectl get memcached/example-memcached -o yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  clusterName: ""
  creationTimestamp: 2018-03-31T22:51:08Z
  generation: 0
  name: example-memcached
  namespace: default
  resourceVersion: "245453"
  selfLink: /apis/cache.example.com/v1alpha1/namespaces/default/memcacheds/example-memcached
  uid: 0026cc97-3536-11e8-bd83-0800274106a1
spec:
  size: 3
status:
  nodes:
  - example-memcached-6fd7c98d8-7dqdr
  - example-memcached-6fd7c98d8-g5k7v
  - example-memcached-6fd7c98d8-m7vn7
```

### Update the size

Change the `spec.size` field in the memcached CR from 3 to 4:
```
$ kubectl get memcached/example-memcached -o yaml | sed 's|size: 3|size: 4|g' | kubectl apply -f -
```

Confirm that the operator changes the deployment size:
```
$ kubectl get deployment
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
example-memcached    4         4         4            4           5m
```

### Cleanup

Clean up the resources:

```
kubectl delete memcached example-memcached
kubectl delete -f deploy/operator.yaml
```

[memcached_handler]: ../../example/memcached-operator/handler.go
[layout_doc]:../project_layout.md
[dep_tool]:https://golang.github.io/dep/docs/installation.html
[go_tool]:https://golang.org/dl/
[docker_tool]:https://docs.docker.com/install/
[kubectl_tool]:https://kubernetes.io/docs/tasks/tools/install-kubectl/
[minikube_tool]:https://github.com/kubernetes/minikube#installation