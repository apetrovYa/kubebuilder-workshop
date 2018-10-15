# Kubebuilder Workshop

The Kubebuilder Workshop is aimed at providing hands-on experience building Kubernetes APIs using kubebuilder.
By the end of the workshop, participants will have built a Kubernetes native API for running MongoDB instances.

Once the API is installed into a Kubernetes cluster, users should be able to create new MongoDB instance similar
to the one in [this blog post](https://kubernetes.io/blog/2017/01/running-mongodb-on-kubernetes-with-statefulsets/) by
specifying a MongoDB file and running `kubectl apply -f` on it.

The MongoDB API will manage (Create / Update) a Kubernetes **StatefulSet** and **Service** that runs a MongoDB instance.

Example file:

```yaml
apiVersion: databases.k8s.io/v1alpha1
kind: MongoDB
metadata:
  name: mongo-instance
spec:
  replicas: 3
  storage: 100Gi
```

## **Prerequisites**

**Important:** Do these steps before moving on

[kubebuilder-workshop-prereqs](https://github.com/pwittrock/kubebuilder-workshop-prereqs)

## Overview

Following is an overview of the steps required to implement the MongoDB API.

1. Create the scaffolded boilerplate for the MongoDB Resource and Controller
1. Update the MongoDB Resource scaffold with a Schema
1. Update the MongoDB Controller `add` stub to Watch StatefulSets, Services, and MongoDBs
1. Update the MongoDB controller `Reconcile` stub to create / update StatefulSets and Services

## Scaffold the boilerplate for the MongoDB Resource and Controller

Scaffold the boilerplate for a new MongoDB Resource type and Controller

**Note:** This will also build the project and run the tests to make sure the resource and controller are hooked up
correctly.

- `kubebuilder create api --group databases --version v1alpha1 --kind MongoDB`
  - enter `y` to have it create boilerplate for the Resource
  - enter `y` to have it create boilerplate for the Controller
  
## Update the MongoDB Resource scaffold with a Schema

Define the MongoDB API Schema *Spec*for in `pkg/apis/databases/v1alpha/mongodb_types.go`.

Spec contains 2 optional fields:

- `replicas` (int32)
- `storage` (string)

**Note:** Copy the following Spec, optionally revisit later to add more fields.

```go
type MongoDBSpec struct {
	// +optional
	Replicas *int32 `json:"replicas,omitempty"`

	// +optional
	Storage *string `json:"storage,omitempty"`
}
```

**Note:** Fields have been made optional by:

- setting `// +optional`
- making them pointers with `*`
- adding the `omitempty` struct tag

Documentation:

- [Resource Definition](https://book.kubebuilder.io/basics/simple_resource.html)
- [PodSpec Example](https://github.com/kubernetes/api/blob/master/core/v1/types.go#L2715)

## Implement the MongoDB Controller

The MongoDB Controller should manage a StatefulSet and Service for running MongoDB.

### Update `add` with Watches

Update the `add` function to Watch the Resources you will be creating / updating in
`pkg/controller/mongodb/mongodb_controller.go`.

- *No-Op* - Watch MongoDB (EnqueueRequestForObject) - this was scaffolded for you
- *Add* - Watch Services - and map to the Owning MongoDB instance (EnqueueRequestForOwner) - you need to add this
- *Add* - Watch StatefulSets - and map to the Owning MongoDB instance (EnqueueRequestForOwner) - you need to add this
- *Delete* - Watch Deployments - you aren't managing Deployments

Documentation:

- [Simple Watch](https://book.kubebuilder.io/basics/simple_controller.html#adding-a-controller-to-the-manager)
- [Advanced Watch](https://book.kubebuilder.io/beyond_basics/controller_watches.html)

### Update `Reconcile` with object creation

Update the `Reconcile` function to Create / Update the StatefulSet and Service objects to run MongoDB in
`pkg/controller/mongodb/mongodb_controller.go`.

- Use the provided `GenerateService` function to get a Service struct instance
- Use the struct to either Create or Update a Service to run MongoDB
  - **Warning**: For Services, be careful to only update the *Selector* and *Ports* so as not to overwrite ClusterIP.
- Use the provided `GenerateStatefuleSet` function to get a StatefulSet struct instance
- Use the struct to either Create or Update a StatefulSet to run MongoDB
  - **Note:** For StatefulSet you *can* update the full Spec if you want

Documentation:

- [Reconcile](https://book.kubebuilder.io/basics/simple_controller.html#implementing-controller-reconcile)

- **Optional:** for when running in cluster - update the RBAC rules to give perms for StatefulSets and
  Services (needed for if running as a container in a cluster)
  - `// +kubebuilder:rbac:groups=apps,resources=statefulesets,verbs=get;list;watch;create;update;patch;delete`
  - `// +kubebuilder:rbac:groups=,resources=services,verbs=get;list;watch;create;update;patch;delete`
  - `// +kubebuilder:rbac:groups=databases.k8s.io,resources=mongodbs,verbs=get;list;watch;create;update;patch;delete`

## Try your API in a Kubernetes Cluster

Now that you have finished implementing the MongoDB API, lets try it out in a Kubernetes cluster.

### Install the Resource into the Cluster

- `make install` # install the CRDs

### Run the Controller locally

- `make run` # run the controller as a local process

### Edit the sample MongoDB file

```yaml
apiVersion: databases.k8s.io/v1alpha1
kind: MongoDB
metadata:
  name: mongo-instance
spec:
  replicas: 1
  storage: 100Gi
```

- edit `config/samples/databases_v1alpha1_mongodb.yaml`
- create the mongodb instance
  - `kubectl apply -f config/samples/databases_v1alpha1_mongodb.yaml`
  - observe output from Controller

### Check out the Resources in the cluster

- look at created resources
  - `kubectl get monogodbs`
  - `kubectl get statefulsets`
  - `kubectl get services`
  - `kubectl get pods`
    - **note**: the containers may be creating - wait for them to come up
  - `kubectl describe pods`
  - `kubectl logs mongo-instance-mongodb-statefulset-0 mongo`
- delete the mongodb instance
  - `kubectl delete -f config/samples/databases_v1alpha1_mongodb.yaml`
- look for garbage collected resources (they should be gone)
  - `kubectl get monogodbs`
  - `kubectl get statefulsets`
  - `kubectl get services`
  - `kubectl get pods`

### Connect to the running MongoDB instance from within the cluster using a Pod

- `kubectl run mongo-test -t -i --rm --image mongo bash`
- `mongo <address of service>:27017`

## Experiment some more

- Try deleting the StatefulSet - what happens when you look for it?
- Try deleting the Service - what happens when you look for it?
- Try adding fields to control new things such as the Port
- Try adding a *Spec*, what useful things can you put in there?

## Bonus Objectives

If you finish early, or want to continue working on your API after the workshop, try these exercises.

### Build your API into a container and publish it

- requires [kustomize](https://github.com/kubernetes-sigs/kustomize)
- `IMG=foo make docker-build` && `IMG=foo make docker-push`
- `kustomize build config/default > mongodb_api.yaml`
- `kubectl apply -f mongodb_api.yaml`
- Get logs from the Controller using `kubectl logs`

### Add Simple Schema Validation

- [Validation tags docs](https://book.kubebuilder.io/basics/simple_resource.html)

### Publish Events from the Controller code that can been seen with `kubectl describe`

- [Event docs](https://book.kubebuilder.io/beyond_basics/creating_events.html)

### Add more Operational Logic to the Reconcile

Add logic to the Reconcile to handle MongoDB lifecycle events such as upgrades / downgrades.

### Define and add a list of Conditions to the Status, then set them from the Controller

- Example: [Node Conditions](https://kubernetes.io/docs/concepts/architecture/nodes/#condition)

### Setup Defaulting and complex Validation

- [Webhook docs](https://book.kubebuilder.io/beyond_basics/what_is_a_webhook.html)
