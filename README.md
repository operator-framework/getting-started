# Getting Started

The [Operator Framework][org_operator_framework] ([intro blog post][site_blog_post]) is an open source toolkit to manage Kubernetes native applications, called operators, in an effective, automated, and scalable way. Operators take advantage of Kubernetes's extensibility to deliver the automation advantages of cloud services like provisioning, scaling, and backup/restore while being able to run anywhere that Kubernetes can run.

This guide shows how to build a simple [memcached][site_memcached] operator and how to manage its lifecycle from install to update to a new version. For that, we will use two center pieces of the framework:

* **Operator SDK**: Allows your developers to build an operator based on your expertise without requiring knowledge of Kubernetes API complexities.
* **Lifecycle Manager**: Helps you to install, update, and generally manage the lifecycle of all of the operators (and their associated services) running across your clusters.

## Build an operator using the Operator SDK

The Operator SDK makes it easier to build Kubernetes native applications, a process that can require deep, application-specific operational knowledge. The SDK not only lowers that barrier, but it also helps reduce the amount of boilerplate code needed for many common management capabilities, such as metering or monitoring.

This section walks through an example of building a simple memcached operator using tools and libraries provided by the Operator SDK. This walkthrough is not exhaustive; for an in-depth explanation of these steps, see the SDK's [user guide][doc_sdk_user_guide].

**Requirements**: Please make sure that the Operator SDK is [installed][doc_sdk_install_instr] on the development machine. Additionally, the Operator Lifecycle Manager must be [installed][doc_olm_install_instr] in the cluster (1.8 or above to support the apps/v1beta2 API group) before running this guide.

### Create a new project

Use the CLI to create a new `memcached-operator` project:


```sh
$ mkdir -p $GOPATH/src/github.com/example-inc/
$ cd $GOPATH/src/github.com/example-inc/
$ operator-sdk new memcached-operator
Create cmd/manager/main.go
...
Run dep ensure ...
...
Run dep ensure done
Run git init ...
...
Run git init done
$ cd memcached-operator
```

This creates the `memcached-operator` project.

Learn more about the project directory structure from the SDK [project layout][layout_doc] documentation.

### Manager

The main program for the operator `cmd/manager/main.go` initializes and runs the [Manager][manager_go_doc].

The Manager will automatically register the scheme for all custom resources defined under `pkg/apis/...` and run all controllers under `pkg/controller/...`.

The Manager can restrict the namespace that all controllers will watch for resources:

```Go
mgr, err := manager.New(cfg, manager.Options{Namespace: namespace})
```

By default this will be the namespace that the operator is running in. To watch all namespaces leave the namespace option empty:

```Go
mgr, err := manager.New(cfg, manager.Options{Namespace: ""})
```

## Add a new Custom Resource Definition

Add a new Custom Resource Definition (CRD) API called `Memcached`, with APIVersion `cache.example.com/v1alpha1` and Kind `Memcached`.

```sh
$ operator-sdk add api --api-version=cache.example.com/v1alpha1 --kind=Memcached
```

This will scaffold the `Memcached` resource API under `pkg/apis/cache/v1alpha1/...`.

### Define the Memcached spec and status

Modify the spec and status of the `Memcached` Custom Resource (CR) at `pkg/apis/cache/v1alpha1/memcached_types.go`:

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

After modifying the `*_types.go` file always run the following command to update the generated code for that resource type:

```sh
$ operator-sdk generate k8s
```

## Add a new Controller

Add a new [Controller][controller_go_doc] to the project that will watch and reconcile the `Memcached` resource:

```sh
$ operator-sdk add controller --api-version=cache.example.com/v1alpha1 --kind=Memcached
```

This will scaffold a new Controller implementation under `pkg/controller/memcached/...`.

For this example replace the generated Controller file `pkg/controller/memcached/memcached_controller.go` with the example [`memcached_controller.go`][memcached_controller] implementation.

The example Controller executes the following reconciliation logic for each `Memcached` CR:

* Create a memcached Deployment if it doesn't exist
* Ensure that the Deployment size is the same as specified by the `Memcached` CR spec
* Update the `Memcached` CR status with the names of the memcached pods

The next two subsections explain how the Controller watches resources and how the reconcile loop is triggered. Skip to the [Build](#build-and-run-the-operator) section to see how to build and run the operator.

### Resources watched by the Controller

Inspect the Controller implementation at `pkg/controller/memcached/memcached_controller.go` to see how the Controller watches resources.

The first watch is for the `Memcached` type as the primary resource. For each Add/Update/Delete event the reconcile loop will be sent a reconcile `Request` (a namespace/name key) for that `Memcached` object:

```Go
err := c.Watch(
    &source.Kind{Type: &cachev1alpha1.Memcached{}},
    &handler.EnqueueRequestForObject{},
  )
```

The next watch is for Deployments but the event handler will map each event to a reconcile `Request` for the owner of the Deployment. Which in this case is the `Memcached` object for which the Deployment was created. This allows the controller to watch Deployments as a secondary resource.

```Go
err := c.Watch(
    &source.Kind{Type: &appsv1.Deployment{}},
    &handler.EnqueueRequestForOwner{
        IsController: true,
        OwnerType:    &cachev1alpha1.Memcached{}},
    )
```

### Reconcile loop

Every Controller has a Reconciler object with a `Reconcile()` method that implements the reconcile loop. The reconcile loop is passed the [`Request`][request_go_doc] argument which is a Namespace/Name key used to lookup the primary resource object, `Memcached`, from the cache:

```Go
func (r *ReconcileMemcached) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    // Lookup the Memcached instance for this reconcile request
    memcached := &cachev1alpha1.Memcached{}
    err := r.client.Get(context.TODO(), request.NamespacedName, memcached)
    ...
}  
```

For a guide on Reconcilers, Clients, and interacting with resource Events, see the [Client API doc][doc_client_api].

## Build and run the operator

Before running the operator, the CRD must be registered with the Kubernetes apiserver:

```sh
$ kubectl create -f deploy/crds/cache_v1alpha1_memcached_crd.yaml
```

Once this is done, there are two ways to run the operator:

* As a Deployment inside a Kubernetes cluster
* As Go program outside a cluster

### 1. Run as a Deployment inside the cluster

Build the memcached-operator image and push it to a registry:

```sh
$ operator-sdk build quay.io/example/memcached-operator:v0.0.1
$ sed -i 's|REPLACE_IMAGE|quay.io/example/memcached-operator:v0.0.1|g' deploy/operator.yaml
# On OSX use:
$ sed -i "" 's|REPLACE_IMAGE|quay.io/example/memcached-operator:v0.0.1|g' deploy/operator.yaml
$ docker push quay.io/example/memcached-operator:v0.0.1
```

The Deployment manifest is generated at `deploy/operator.yaml`. Be sure to update the deployment image as shown above since the default is just a placeholder.

Setup RBAC and deploy the memcached-operator:

```sh
$ kubectl create -f deploy/service_account.yaml
$ kubectl create -f deploy/role.yaml
$ kubectl create -f deploy/role_binding.yaml
$ kubectl create -f deploy/operator.yaml
```

Verify that the memcached-operator is up and running:

```sh
$ kubectl get deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
memcached-operator       1         1         1            1           1m
```

### 2. Run locally outside the cluster

This method is preferred during development cycle to deploy and test faster.

Set the name of the operator in an environment variable:

```sh
export OPERATOR_NAME=memcached-operator
```

Run the operator locally with the default kubernetes config file present at `$HOME/.kube/config`:

```sh
$ operator-sdk up local --namespace=default
2018/09/30 23:10:11 Go Version: go1.10.2
2018/09/30 23:10:11 Go OS/Arch: darwin/amd64
2018/09/30 23:10:11 operator-sdk Version: 0.0.6+git
2018/09/30 23:10:12 Registering Components.
2018/09/30 23:10:12 Starting the Cmd.
```

You can use a specific kubeconfig via the flag `--kubeconfig=<path/to/kubeconfig>`.

## Create a Memcached CR

Create the example `Memcached` CR that was generated at `deploy/crds/cache_v1alpha1_memcached_cr.yaml`:

```sh
$ cat deploy/crds/cache_v1alpha1_memcached_cr.yaml
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "example-memcached"
spec:
  size: 3

$ kubectl apply -f deploy/crds/cache_v1alpha1_memcached_cr.yaml
```

Ensure that the memcached-operator creates the deployment for the CR:

```sh
$ kubectl get deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
memcached-operator       1         1         1            1           2m
example-memcached        3         3         3            3           1m
```

Check the pods and CR status to confirm the status is updated with the memcached pod names:

```sh
$ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
example-memcached-6fd7c98d8-7dqdr     1/1       Running   0          1m
example-memcached-6fd7c98d8-g5k7v     1/1       Running   0          1m
example-memcached-6fd7c98d8-m7vn7     1/1       Running   0          1m
memcached-operator-7cc7cfdf86-vvjqk   1/1       Running   0          2m
```

```sh
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

Change the `spec.size` field in the memcached CR from 3 to 4 and apply the change:

```sh
$ cat deploy/crds/cache_v1alpha1_memcached_cr.yaml
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "example-memcached"
spec:
  size: 4

$ kubectl apply -f deploy/crds/cache_v1alpha1_memcached_cr.yaml
```

Confirm that the operator changes the deployment size:

```sh
$ kubectl get deployment
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
example-memcached    4         4         4            4           5m
```

### Cleanup

Delete the operator and its related resources:

```sh
$ kubectl delete -f deploy/crds/cache_v1alpha1_memcached_cr.yaml
$ kubectl delete -f deploy/operator.yaml
$ kubectl delete -f deploy/role_binding.yaml
$ kubectl delete -f deploy/role.yaml
$ kubectl delete -f deploy/service_account.yaml
```

## Reference implementation

The above walkthrough follows the actual implementation process used to produce the `memcached-operator` in the SDK [samples repo][repo_sdk_samples_memcached].

## Manage the operator using the Operator Lifecycle Manager

The previous section has covered manually running an operator. In the next sections, we will explore using the [Operator Lifecycle Manager][operator_lifecycle_manager] (OLM) which is what enables a more robust deployment model for operators being run in production environments.

OLM helps you to install, update, and generally manage the lifecycle of all of the operators (and their associated services) on a Kubernetes cluster. It runs as an Kubernetes extension and lets you use `kubectl` for all the lifecycle management functions without any additional tools.



### Generate an operator manifest

The first step to leveraging OLM is to create a [Cluster Service Version][csv_design_doc] (CSV) manifest. An operator manifest describes how to display, create and manage the application, in this case memcached, as a whole. It is required for OLM to function.

The Operator SDK CLI can generate CSV manifests via the following command:
```console
$ operator-sdk olm-catalog gen-csv --csv-version 0.0.1
```
For more details see the SDK's [CSV generation doc][csv_generation_doc].


For the purpose of this guide, we will continue with this [predefined manifest][manifest_v1] file for the next steps. If you’d like, you can alter the image field within this manifest to reflect the image you built in previous steps, but it is unnecessary.

### Deploy the Operator

Deploying an operator is as simple as applying the operator’s CSV manifest to the desired namespace in the cluster.

First we need to create an [OperatorGroup][operator_group_doc] that specifies the namespaces that the operator will be targeting. Create the following OperatorGroup in the namespace where you will create the CSV. In this example the `default` namespace is used.

```YAML
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: memcached-operator-group
  namespace: default
  spec:
    targetNamespaces:
    - default
```

Next create the CSV.

```sh
$ curl -Lo memcachedoperator.0.0.1.csv.yaml https://raw.githubusercontent.com/operator-framework/getting-started/master/memcachedoperator.0.0.1.csv.yaml
$ kubectl apply -f memcachedoperator.0.0.1.csv.yaml
$ kubectl get ClusterServiceVersion memcachedoperator.v0.0.1 -o json | jq '.status'
```

After applying this manifest, nothing has happened yet, because the cluster does not meet the requirements specified in our manifest. Create the CustomResourceDefinition and RBAC rules for the `Memcached` type managed by the operator:

```sh
$ kubectl create -f deploy/crds/cache_v1alpha1_memcached_crd.yaml
$ kubectl create -f deploy/service_account.yaml
$ kubectl create -f deploy/role.yaml
$ kubectl create -f deploy/role_binding.yaml
```

Because OLM creates operators in a particular namespace when a manifest has been applied, administrators can leverage the native Kubernetes RBAC permission model to restrict which users are allowed to install operators.

### Create an application instance

The memcached operator is now running in the `memcached` namespace. Users interact with operators via instances of CustomResources; in this case, the resource has the Kind `Memcached`. Native Kubernetes RBAC also applies to CustomResources, providing administrators control over who can interact with each operator.

Creating instances of memcached in this namespace will now trigger the memcached operator to instantiate pods running the memcached server that are managed by the operator. The more CustomResources you create, the more unique instances of memcached will be managed by the memcached operator running in this namespace.

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

Manually applying an update to the operator is as simple as creating a new operator manifest with a `replaces` field that references the old operator manifest. OLM will ensure that all resources being managed by the old operator have their ownership moved to the new operator without fear of any programs stopping execution. It is up to the operators themselves to execute any data migrations required to upgrade resources to run under a new version of the operator.

The following command demonstrates applying a new [operator manifest][manifest_v2] using a new version of the operator and shows that the pods remain executing:

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

Hopefully, this guide was an effective demonstration of the value of the Operator Framework for building and managing operators. There is much more that we left out in the interest of brevity. The Operator Framework and its components are open source, so please feel encouraged to jump into each individually and learn what else you can do. If you want to discuss your experience, have questions, or want to get involved, join the Operator Framework [mailing list][mailing_list].

<!---  Reference URLs begin here -->

[operator_group_doc]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/Documentation/design/operatorgroups.md
[csv_design_doc]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/Documentation/design/building-your-csv.md
[csv_generation_doc]: https://github.com/operator-framework/operator-sdk/blob/master/doc/user/olm-catalog/generating-a-csv.md
[org_operator_framework]: https://github.com/operator-framework/
[site_blog_post]: https://coreos.com/blog/introducing-operator-framework
[operator_sdk]: https://github.com/operator-framework/operator-sdk
[operator_lifecycle_manager]: https://github.com/operator-framework/operator-lifecycle-manager
[site_memcached]: https://memcached.org/
[doc_sdk_user_guide]: https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md
[doc_sdk_install_instr]: https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md#install-the-operator-sdk-cli
[doc_olm_install_instr]: https://github.com/operator-framework/operator-lifecycle-manager/blob/master/Documentation/install/install.md
[layout_doc]: https://github.com/operator-framework/operator-sdk/blob/master/doc/project_layout.md
[manager_go_doc]: https://godoc.org/github.com/kubernetes-sigs/controller-runtime/pkg/manager#Manager
[controller_go_doc]: https://godoc.org/github.com/kubernetes-sigs/controller-runtime/pkg#hdr-Controller
[memcached_controller]: https://github.com/operator-framework/operator-sdk/blob/master/example/memcached-operator/memcached_controller.go.tmpl
[request_go_doc]: https://godoc.org/github.com/kubernetes-sigs/controller-runtime/pkg/reconcile#Request
[result_go_doc]: https://godoc.org/github.com/kubernetes-sigs/controller-runtime/pkg/reconcile#Result
[doc_client_api]: https://github.com/operator-framework/operator-sdk/blob/master/doc/user/client.md
[repo_sdk_samples_memcached]: https://github.com/operator-framework/operator-sdk-samples/tree/master/memcached-operator/
[manifest_v1]: memcachedoperator.0.0.1.csv.yaml
[manifest_v2]: memcachedoperator.0.0.2.csv.yaml
[mailing_list]: https://groups.google.com/forum/#!forum/operator-framework
