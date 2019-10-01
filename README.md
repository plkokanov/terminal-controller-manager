# Gardener Terminal Controller Manager
<img src="https://user-images.githubusercontent.com/5526658/65954528-77302b80-e446-11e9-9dc5-1bd4c580a556.png" alt="Terminal Controller Manager" width="180"/>

## Development Setup

Prerequisites:
- [kind](https://github.com/kubernetes-sigs/kind)
- [kustomize](https://github.com/kubernetes-sigs/kustomize)
- [kubectl](https://kubernetes.io/de/docs/tasks/tools/install-kubectl/)

To build and push the images, run `docker-build` and `docker-push` make targets. Adapt the image as necessary:

```bash
make docker-build docker-push IMG=example/my-repo
```

There are two options to deploy the `terminal-controller-manager` into your cluster
- single-cluster setup
  - The hosting and target cluster are the same (see `config/overlay/single-cluster`).
- multi-cluster setup
  - The `terminal-controller-manager` is deployed in a hosting cluster, which is called `runtime` cluster (see `config/overlay/multi-cluster/runtime`) but watches the terminal resources in another/target cluster, called `virtual-garden` (see `config/overlay/multi-cluster/virtual-garden`).
  - The admission webhook configurations of course are created in the virtual-garden.



## Single-Cluster Setup with `Kind`

To deploy the `terminal-controller-manager` into a single-cluster execute:

```bash
kind create cluster
```

Use the cluster with:

```
export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"

```

Run `deploy-singlecluster` make target. 

Optionally specify the `IMG` in case you built your own image, otherwise the default `eu.gcr.io/gardener-project/gardener/terminal-controller-manager:latest` is used.

```bash
make deploy-singlecluster
```

To verify that everything worked, check if the `terminal-controller-manager` pod is running in the `terminal-system` namespace:

```bash
kubectl -n terminal-system get po

NAME                                           READY   STATUS    RESTARTS   AGE
terminal-controller-manager-6d7b5bfbdd-l6vg9   2/2     Running   0          10s
```

### Create Example Terminal Resource
 
Run the `bootstrap-dev` make target to bootstrap your cluster with dummy namespaces, serviceaccounts, clusterroles and clusterrolebinding that are required for the example terminal resources `config/samples/terminal-cp.yaml` and `config/samples/terminal-shoot.yaml`:

```bash
make bootstrap-dev
``` 

Apply the terminal resources

```bash
kubectl apply -f config/samples/terminal-cp.yaml 
```

```bash
kubectl apply -f config/samples/terminal-shoot.yaml 
```

Check if the terminal resources were created. Note that the terminal resources get cleaned if they don't receive a heartbeat.

```bash
kubectl get terminals -n garden-coretmp

NAME                AGE
term-lukas-hib      33s
term-lukas-hib-cp   22s
```

The `terminal-controller-manager` should create, among other resources, a pod in the respective target namespace that is declared in the terminal resource.
E.g. for the example `terminal-cp.yaml` the target namespace is `shoot--core--mycluster`.

```bash
kubectl -n shoot--core--mycluster get po

NAME                      READY   STATUS    RESTARTS   AGE
term-627737417320032009   1/1     Running   0          17s
```

Exec into the pod and verify that you can talk to the api server:

```bash
kubectl -n shoot--core--mycluster exec -it term-627737417320032009 bash

root at term-627737417320032009 in / on 10.96.0.1 [shoot--core--mycluster]
$ k get po
NAME                      READY   STATUS    RESTARTS   AGE
term-627737417320032009   1/1     Running   0          25s
```

