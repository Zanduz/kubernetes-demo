# Kubernetes demo

By Aleksander Frese

DTU DevOps course.

## Orientation

Let's first get an overview of the cluster, both on node-level and workload-level.

```sh
❯ kubectl get nodes
NAME                   STATUS   ROLES                  AGE    VERSION
lima-rancher-desktop   Ready    control-plane,master   270d   v1.29.6+k3s1
```

We have a single node, that acts as both master and worker node.

```sh
❯ kubectl get namespaces
NAME              STATUS   AGE
kube-system       Active   10d
kube-public       Active   10d
kube-node-lease   Active   10d
default           Active   10d
```

We see the standard namespaces present in a fresh Kubernetes cluster.

## Pod

Lets run a simple busybox container. Note the sleep command, which is there to keep the Pod runnning.

```sh
❯ kubectl run busybox-pod --image busybox -- sleep 1d
pod/busybox-pod created
```

Check that it gets created and runs.

```sh
❯ kubectl get pod
NAME          READY   STATUS    RESTARTS   AGE
busybox-pod   1/1     Running   0          27s
```

Check the Pod YAML.

```sh
❯ kubectl get pod busybox-pod -o yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
  (...)
spec:
  containers:
  - image: busybox
    (...)
```

We can also check the YAML manifest that would be generated and sent to Kubernetes by the run command without actually committing anything to the cluster.

```sh
❯ kubectl run busybox-pod --image busybox --dry-run=client -o yaml -- sleep 1d
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
  (...)
spec:
  containers:
  - image: busybox
    (...)
  (...)
(...)
```

## Deployment

Now let's run the busybox container via a Deployment.

```sh
❯ kubectl create deployment busybox-deployment --image busybox -- sleep 1d
deployment.apps/busybox-deployment created
```

Check that it get created.

```sh
❯ kubectl get deploy
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
busybox-deployment   1/1     1            1           12s
```

Check the Pod that was created by the Deployment with a randomly generated suffix. Note that the Pod we created before is still there.

```sh
❯ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
busybox-deployment-77b4cc8db8-9mcn6   1/1     Running   0          15s
busybox-pod                           1/1     Running   0          6m11s
```

## Pod vs. Deployment

Let's now see what happens when you delete the Pods.

Delete the Pod managed by the Deployment.

```sh
❯ kubectl delete pod busybox-deployment-77b4cc8db8-9mcn6
pod "busybox-deployment-77b4cc8db8-9mcn6" deleted
```

Check the Pods.

```sh
❯ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
busybox-deployment-77b4cc8db8-2wxqx   1/1     Running   0          38s
busybox-pod                           1/1     Running   0          9m57s
```

A new Pod was created in its place.

Now delete the stand-alone Pod we initially created.

```sh
❯ kubectl delete pod busybox-pod
pod "busybox-pod" deleted
```

And check the Pods again.

```sh
❯ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
busybox-deployment-77b4cc8db8-2wxqx   1/1     Running   0          2m14s
```

No new Pod was created in its place.

## Scaling

Now let's scale the Deployment up to two replicas.

```sh
❯ kubectl scale deployment busybox-deployment --replicas 2
deployment.apps/busybox-deployment scaled
```

Check the Pods after a bit.

```sh
❯ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
busybox-deployment-77b4cc8db8-2wxqx   1/1     Running   0          4m9s
busybox-deployment-77b4cc8db8-kfvv8   1/1     Running   0          24s
```

We now see two Pods, both with a random suffix.

Let's try to delete one of them and see what happens.

```sh
❯ kubectl delete pod busybox-deployment-77b4cc8db8-kfvv8
pod "busybox-deployment-77b4cc8db8-kfvv8" deleted
```

Check the Pods.

```sh
❯ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
busybox-deployment-77b4cc8db8-2wxqx   1/1     Running   0          6m32s
busybox-deployment-77b4cc8db8-k2z47   1/1     Running   0          60s
```

Kubernetes created a new Pod in its place to keep the replica count at 2.

## Namespacing

To quickly demo the logical grouping capability in Kubernetes, let's create two new namespaces and run some Pods in them.

Again, let's quickly check the list of existing namespaces.

```sh
❯ kubectl get namespaces
NAME              STATUS   AGE
default           Active   270d
kube-node-lease   Active   270d
kube-public       Active   270d
kube-system       Active   270d
```

Create the namespaces.

```sh
❯ kubectl create namespace ns1
namespace/ns1 created

❯ kubectl create namespace ns2
namespace/ns2 created
```

Check that the namespaces are now in the list.

```sh
❯ kubectl get namespaces
NAME              STATUS   AGE
default           Active   270d
kube-node-lease   Active   270d
kube-public       Active   270d
kube-system       Active   270d
ns1               Active   4m49s
ns2               Active   4m47s
```

Run a Pod in both namespaces.

```sh
❯ kubectl --namespace ns1 run busybox-pod-ns1 --image busybox -- sleep 1d
pod/busybox-pod-ns1 created

❯ kubectl --namespace ns2 run busybox-pod-ns2 --image busybox -- sleep 1d
pod/busybox-pod-ns2 created
```

Let's check the Pods.

```sh
❯ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
busybox-deployment-77b4cc8db8-2wxqx   1/1     Running   0          11m
busybox-deployment-77b4cc8db8-k2z47   1/1     Running   0          5m31s
```

We don't see our Pods. Why? Because by default we are working with the `default` namespace.

We must specify which namespace we want to work with.

```sh
❯ kubectl get pods --namespace ns1
NAME              READY   STATUS    RESTARTS   AGE
busybox-pod-ns1   1/1     Running   0          32s
```

We correctly see the first Pod we created in the `ns1` namespace.

```sh
❯ kubectl get pods --namespace ns2
NAME              READY   STATUS    RESTARTS   AGE
busybox-pod-ns2   1/1     Running   0          2m31s
```

And there we see the second Pod in the `ns2` namespace.

## Logs

I'd like to show how to view logs for a container in a Pod.

Let's run a container from the hello-world image.

```sh
❯ kubectl run hello-world-pod --image hello-world
pod/hello-world-pod created
```

Check the Pods.

```sh
❯ kubectl get pods
NAME                                  READY   STATUS             RESTARTS      AGE
busybox-deployment-77b4cc8db8-2wxqx   1/1     Running            0             19m
busybox-deployment-77b4cc8db8-k2z47   1/1     Running            0             14m
hello-world-pod                       0/1     Completed          0             34s
```

We see the Pod was created an it ran to completion.

Let's check the logs.

```sh
❯ kubectl logs hello-world-pod

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```