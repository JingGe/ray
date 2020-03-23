# Ray Kubernetes Operator (experimental)

NOTE: The operator is still under active development and not yet recommended for production deployments. Please see [the documentation](https://ray.readthedocs.io/en/latest/deploy-on-kubernetes.html#deploying-on-kubernetes) for current best practices.

This directory contains the source code for a Ray operator for Kubernetes.

The operator makes deploying and managing Ray clusters on top of Kubernetes painless - clusters are defined as a custom RayCluster resource and managed by a fault-tolerant Ray controller.
The Ray Operator is a Kubernetes operator to automate provisioning, management, autoscaling and operations of Ray clusters deployed to Kubernetes.

Some of the main features of the operator are:
- Management of first-class RayClusters via a [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-resources).
- Support for hetergenous worker types in a single Ray cluster.
- Built-in monitoring via Prometheus.

## File structure
> ```
> ray/deploy/ray-operator
> ├── api/v1alpha1  // Package v1alpha1 contains API Schema definitions for the ray v1alpha1 API group
> │   ├── groupversion_info.go // contains common metadata about the group-version
> │   ├── raycluster_types.go  // RayCluster field definitions, user should focus
> │   └── zz_generated.deepcopy.go // contains the autogenerated implementation of the aforementioned runtime.Object interface, which marks all of our root types as representing Kinds.
> │   
> │── config   // contains Kustomize YAML definitions required to launch our controller on a cluster,hold our CustomResourceDefinitions, RBAC configuration, and WebhookConfigurations.
> │  ├── certmanager  
> │  │   ├── certificate.yaml  // The following manifests contain a self-signed issuer CR and a certificate CR.
> │  │   ├── kustomization.yaml
> │  │   └── kustomizeconfig.yaml
> │  │
> │  ├── crd          
> │  │   └── bases
> │  │   │   └── ray.io_rayclusters.yaml  // RayCluster CRD yaml file
> │  │   └── patches
> │  │   │   ├── cainjection_in_rayclusters.yaml  // adds a directive for certmanager to inject CA into the CRD
> │  │   │   └── webhook_in_rayclusters.yaml  // enables conversion webhook for CRD
> │  │   │── kustomization.yaml
> │  │   └── kustomizeconfig.yaml
> │  │
> │  ├── default     // contains a Kustomize base for launching the controller in a standard configuration.
> │  │   ├── kustomization.yaml
> │  │   ├── manager_auth_proxy_patch.yaml // inject a sidecar container which is a HTTP proxy for the controller manager, it performs RBAC authorization against the Kubernetes API using SubjectAccessReviews.
> │  │   ├── manager_webhook_patch.yaml    // webhook yaml file
> │  │   └── webhookcainjection_patch.yaml // add annotation to admission webhook config
> │  │
> │  ├── manager      // launch your controllers as pods in the cluster.
> │  │   ├── kustomization.yaml
> │  │   └── manager.yaml     // manager yaml to create controller deployment, user should focus
> │  │
> │  ├── prometheus     
> │  │   ├── kustomization.yaml
> │  │   └── monitor.yaml     // Prometheus Monitor Service, user should focus
> │  │
> │  ├── rbac        // permissions required to run your controllers under their own service account.
> │  │   ├── auth_proxy_role.yaml
> │  │   ├── auth_proxy_role_binding.yaml
> │  │   ├── auth_proxy_service.yaml
> │  │   ├── kustomization.yaml
> │  │   ├── leader_election_role.yaml   // permissions to do leader election.
> │  │   ├── leader_election_role_binding.yaml
> │  │   └── role_binding.yaml
> │  │
> │  ├── samples      // sample RayCluster yaml, user should focus
> │  │   ├── ray_v1_raycluster.complete.yaml
> │  │   ├── ray_v1_raycluster.heterogeneous.yaml
> │  │   └── ray_v1_raycluster.mini.yaml
> │  │
> │  └── webhook
> │      ├── kustomization.yaml
> │      ├── kustomizeconfig.yaml
> │      ├── manifests.yaml
> │      └── service.yaml   // webhook-service
> │
> │── controller
> │   ├── common
> │   │   ├── constant.go
> │   │   ├── meta.go
> │   │   ├── pod.go
> │   │   └── service.go
> │   └── raycluster_controller.go
> │
> │── main.go
> └── Makefile
> ```

## Usage

This section walks through how to build and deploy the operator in a running Kubernetes cluster.

### Requirements
software  | version | link
:-------------  | :---------------:| -------------:
kubectl |  v1.11.3+    | [download](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
go  | v1.13+|[download](https://golang.org/dl/)
docker   | 17.03+|[download](https://docs.docker.com/install/)

The instructions assume you have access to a running Kubernetes cluster via ``kubectl``. If you want to test locally, consider using [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/).

### Running the unit tests

```
go build
go test ./...
```

You can also build the operator using Bazel:

```generate BUILD.bazel 
bazel run //:gazelle
```

```build script
bazel build //:ray-operator
```

### Building the controller

The first step to deploying the Ray operator is building the container image that contains the operator controller and pushing it to Docker Hub so it can be pulled down and run in the Kubernetes cluster.

```shell script
# From the ray/deploy/ray-operator directory.
# Replace DOCKER_ACCOUNT with your docker account or push to your preferred Docker image repository.
docker build -t $DOCKER_ACCOUNT:controller .
docker push $DOCKER_ACCOUNT:controller
```

In the future (once the operator is stabilized), an official controller image will be uploaded and available to users on Docker Hub.

### Installing the custom resource definition

The next step is to install the RayCluster custom resource definition into the cluster. Build and apply the CRD:

```shell script
kubectl kustomize config/crd | kubectl apply -f -
```

Refer to [raycluster_types.go](api/v1alpha1/raycluster_types.go) and [ray.io_rayclusters.yaml](config/crd/bases/ray.io_rayclusters.yaml) for the details of the CRD.

### Deploying the controller

First, modify the controller config to use the image you build. Replace "controller" in config/manager/kustomization.yaml with the name of your image. Then, build the controller config and apply it to the cluster:

```shell script
kubectl kustomize config/default | kubectl apply -f -
```

Currently, the controller runs in the ``ray-operator-system`` namespace. This will be configurable in the future.

```shell script
# Check that the controller is running.
$ kubectl get pods -n ray-operator-system
NAME                                               READY   STATUS    RESTARTS   AGE
ray-operator-controller-manager-66b9b97bcf-8l6lt   2/2     Running   0          30s

$ kubectl get deployments -n ray-operator-system
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
ray-operator-controller-manager   1/1     1            1           44m

# Delete the controller if need be.
$ kubectl delete deployment ray-operator-controller-manager -n ray-operator-system
```

### Running an example cluster

There are three example config files to deploy RayClusters included here: 

Sample  | Description
------------- | -------------
[RayCluster.mini.yaml](config/samples/ray_v1_raycluster.mini.yaml)   | Small example consisting of 1 head pod and 1 worker pod.
[RayCluster.heterogeneous.yaml](config/samples/ray_v1_raycluster.heterogeneous.yaml)  | Example with heterogenous worker types. 1 head pod and 2 worker pods, each of which has a different resource quota.
[RayCluster.complete.yaml](config/samples/ray_v1_raycluster.complete.yaml)  | Shows all available custom resouce properties.

```shell script
# Create a cluster.
$ kubectl create -f config/samples/ray_v1_raycluster.mini.yaml -n ray-operator-system
raycluster.ray.io/raycluster-sample created

# List running clusters.
$ kubectl get rayclusters
NAME                AGE
raycluster-sample   2m48s

# The created cluster should include a head pod, worker pod, and a head service.
$ kubectl get pods
NAME                                     READY   STATUS             RESTARTS   AGE
raycluster-sample-head-group-head-0      1/1     Running            0          101s
raycluster-sample-small-group-worker-0   1/1     Running            0          101s

$ kubectl get services
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes               ClusterIP   10.100.0.1      <none>        443/TCP    19d
raycluster-sample-head   ClusterIP   10.100.153.12   <none>        6379/TCP   28h

# Delete the cluster.
kubectl delete raycluster raycluster-sample
```