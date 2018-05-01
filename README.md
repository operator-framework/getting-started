# Getting Started

The [Operator Framework](https://github.com/operator-framework/) ([intro blog post][blog_post]) is an open source toolkit to manage Kubernetes native applications, called Operators, in an effective, automated, and scalable way. Operators take advantage of Kubernetes's extensibility to deliver the automation advantages of cloud services like provisioning, scaling, and backup/restore while being able to run anywhere that Kubernetes can run.

This guide shows how to build a simple Memcached Operator and how to manage its lifecycle from install to update to a new version. For that, we will use two center pieces of the framework:
* **Operator SDK**: Allows your developers to build an Operator based on your expertise without requiring knowledge of Kubernetes API complexities.
* **Operator Lifecycle Manager**: Helps you to install, update, and generally manage the lifecycle of all of the Operators (and their associated services) running across your clusters.

> **Requirements**: Please make sure that both the [Operator SDK][operator_sdk] and [Operator Lifecycle Manager][operator_lifecycle_manager] are installed before running this guide.

## Build an Operator using the Operator SDK

The Operator SDK makes it easier to build Kubernetes native applications, a process that can require deep, application-specific operational knowledge. The SDK not only lowers that barrier, but it also helps reduce the amount of boilerplate code needed for many common management capabilities, such as metering or monitoring.

This section walks through an example of building a simple Memcached Operator using tools and libraries provided by the Operator SDK.

### Create a new project

Use the CLI to create a new `memcached-operator` project:

```sh
$ cd $GOPATH/src/github.com/example-inc/
$ operator-sdk new memcached-operator --api-version=cache.example.com/v1alpha1 --kind=Memcached
$ cd memcached-operator
```

This creates the `memcached-operator` project specifically for watching the `Memcached` resource with APIVersion `cache.example.com/v1apha1` and Kind `Memcached`.

Learn more about the project directory structure from the SDK [project layout][layout_doc] documentation.

### Customize the Operator logic

For this example, the Memcached Operator will execute the following reconciliation logic for each `Memcached` custom resource:

* Create a Memcached Deployment if it doesn't exist
* Ensure that the Deployment size is the same as specified by the `Memcached` [CustomResource][kubernetes_cr] (CR) spec
* Update the `Memcached` CR status with the names of the memcached pods

#### Watch for the Memcached custom resource definition

By default, the memcached-operator watches `Memcached` resource events as shown in `cmd/memcached-operator/main.go`.

```Go
func main() {
  sdk.Watch("cache.example.com/v1alpha1", "Memcached", "default", 5)
  sdk.Handle(stub.NewHandler())
  sdk.Run(context.TODO())
}
```

#### Define the Memcached spec and status

Modify the spec and status of the `Memcached` CRD at `pkg/apis/cache/v1alpha1/types.go`:

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

```sh
$ operator-sdk generate k8s
```

#### Define the Handler

The reconciliation loop for an event is defined in the `Handle()` function at `pkg/stub/handler.go`.

Replace this file with the reference implementation found [here][handler_go]. You will need to update the highlighted line if you have changed the import path of this project to something other than “example-inc”.

> **Note**: The provided handler implementation is only meant to demonstrate the use of the SDK APIs and is not representative of the best practices of a reconciliation loop.

### Build and run the Operator

Build the `memcached-operator` image and push it to a registry. Please make sure you have a an account on [Quay.io][quay_io] for the next step, or substitute your preferred container registry. On the registry, [create a new public image][create_public_image] repository named “memcached-operator”.


```sh
$ operator-sdk build quay.io/example/memcached-operator:v0.0.1
$ docker push quay.io/example/memcached-operator:v0.0.1
```

Kubernetes deployment manifests are generated in `deploy/operator.yaml`. The deployment image is set to the container image specified above.

Deploy the Memcached Operator:

```sh
$ kubectl create -f deploy/rbac.yaml
$ kubectl create -f deploy/operator.yaml
```

Verify that `memcached-operator` is up and running:

```sh
$ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
memcached-operator-75c4b4c665-8jnj5   1/1       Running   0          20s
```

Now let’s clean everything up:

```sh
$ kubectl delete -f deploy/operator.yaml
$ kubectl delete -f deploy/rbac.yaml
```
## Manage the Operator using the Operator Lifecycle Manager

The previous section has covered manually running an Operator. In the next sections, we will explore using the Operator Lifecycle Manager which is what enables a more robust deployment model for Operators being ran in production environments.

The Operator Lifecycle Manager helps you to install, update, and generally manage the lifecycle of all of the Operators (and their associated services) on a Kubernetes cluster. It runs as an Kubernetes extension and lets you use `kubectl` for all the lifecycle management functions without any additional tools.

### Generate an Operator manifest

The first step to leveraging the Operator Lifecycle Manager is to create a manifest. An Operator manifest describes how to display, create and manage the application, in this case Memcached, as a whole. It is required for the Operator Lifecycle Manager to function.

For the purpose of this guide, we will continue with this [predefined manifest][manifest_v1] file for the next steps. If you’d like, you can alter the image field within this manifest to reflect the image you built in previous steps, but it is unnecessary. In the future, the Operator SDK CLI will generate an Operator manifest for you, a feature that is planned for the next release of the Operator SDK.

### Deploy the Operator 

Deploying an Operator is as simple as applying the Operator’s manifest to the desired namespace in the cluster.

```sh
$ curl -Lo memcachedoperator.0.0.1.csv.yaml https://raw.githubusercontent.com/operator-framework/getting-started/master/memcachedoperator.0.0.1.csv.yaml
$ kubectl apply -f memcachedoperator.0.0.1.csv.yaml
$ kubectl get ClusterServiceVersion-v1s memcachedoperator.v0.0.1 -o json | jq '.status'
```

After applying this manifest, nothing has happened yet, because the cluster does not met the requirements specified in our manifest. Create the CustomResourceDefinition and RBAC rules for the Memcached type managed by the Operator:

```sh
$ kubectl apply -f deploy/rbac.yaml
$ sed '/---/q' deploy/operator.yaml | kubectl apply -f -
```

Because the Operator Lifecycle Manager creates Operators in a particular namespace when a manifest has been applied, administrators can leverage the native Kubernetes RBAC permission model to restrict which users are allowed to install Operators. 

### Create an application instance 

The Memcached Operator is now running in the `memcached` namespace. Users interact with Operators via instances of CustomResources; in this case, the resource has the Kind `Memcached`. Native Kubernetes RBAC also applies to CustomResources, providing administrators control over who can interact with each Operator. 

Creating instances of Memcached in this namespace will now trigger the Memcached Operator to instantiate pods running the memcached server that are managed by the Operator. The more CustomResources you create, the more unique instances of Memcached will be managed by the Memcached Operator running in this namespace. 

```sh
$ cat <<EOF | kubectl apply -f -
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "memcached-for-wordpress"
spec:
  size: 1
EOF
$ cat <<EOF | kubectl apply -f -
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "memcached-for-drupal"
spec:
  size: 1
EOF
$ kubectl get Memcached
NAME                      AGE
memcached-for-drupal      22s
memcached-for-wordpress   27s
$ kubectl get pods
NAME                                       READY     STATUS    RESTARTS   AGE
memcached-app-operator-66b5777b79-pnsfj    1/1       Running   0          14m
memcached-for-drupal-5476487c46-qbd66      1/1       Running   0          3s
memcached-for-wordpress-65b75fd8c9-7b9x7   1/1       Running   0          8s
```

### Update an application

Manually applying an update to the Operator is as simple as editing the initially applied Operator manifest and applying it to the cluster. The Operator Lifecycle Manager will ensure that all resources being managed by the old Operator have their ownership moved to the new Operator without fear of any programs stopping execution. It is up to the Operators themselves to execute any data migrations required to upgrade resources to run under a new version of the Operator.

The following command demonstrates applying a new [manifest_v2][Operator manifest] using a new version of the Operator and shows that the pods remain executing:

```sh
$ curl -Lo memcachedoperator.0.0.2.csv.yaml https://raw.githubusercontent.com/operator-framework/getting-started/master/memcachedoperator.0.0.2.csv.yaml
$ kubectl apply -f memcachedoperator.0.0.2.csv.yaml
$ kubectl get pods
NAME                                       READY     STATUS    RESTARTS   AGE
memcached-app-operator-66b5777b79-pnsfj    1/1       Running   0          3s
memcached-for-drupal-5476487c46-qbd66      1/1       Running   0          14m
memcached-for-wordpress-65b75fd8c9-7b9x7   1/1       Running   0          14m
```

## Conclusion

Hopefully, this guide was an effective demonstration of the value of the Operator Framework for building and managing Operators. There is much more that we left out in the interest of brevity. The Operator Framework and its components are open source, so please feel encouraged to jump into each individually and learn what else you can do. If you want to discuss your experience, have questions, or want to get involved, join the Operator Framework mailing list [here][mailing_list].

<!---  Reference URLs begin here -->

[blog_post]: https://coreos.com/blog/introducing-operator-framework
[operator_sdk]: https://github.com/operator-framework/operator-sdk
[operator_lifecycle_manager]: https://github.com/operator-framework/operator-lifecycle-manager
[layout_doc]: https://github.com/operator-framework/operator-sdk/blob/master/doc/project_layout.md
[quay_io]: https://www.quay.io
[kubernetes_cr]: https://kubernetes.io/docs/concepts/api-extension/custom-resources/
[handler_go]: https://github.com/operator-framework/getting-started/blob/master/handler.go.tmpl#L7
[create_public_image]: https://quay.io/new/
[manifest_v1]: memcachedoperator.0.0.1.csv.yaml
[manifest_v2]: memcachedoperator.0.0.2.csv.yaml
[mailing_list]: https://groups.google.com/forum/#!forum/operator-framework
