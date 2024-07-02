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

## Performing a rolling update

Updating a deployment without service interruption to users are also a core part of Kubernetes. This is called rolling updates.

Let's update the image of the Deployment from the before.

```sh
❯ kubectl set image deployment busybox-deployment busybox=httpd
deployment.apps/busybox-deployment image updated
```

Kubernetes immediately does its thing.

```sh
❯ kubectl rollout status deployment busybox-deployment
Waiting for deployment "busybox-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "busybox-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "busybox-deployment" successfully rolled out

❯ kubectl get pods
NAME                                  READY   STATUS        RESTARTS     AGE
alpine                                1/1     Running       0            104m
busybox-deployment-77b4cc8db8-2wxqx   1/1     Terminating   1 (4h ago)   4h28m
busybox-deployment-77b4cc8db8-k2z47   1/1     Terminating   1 (4h ago)   4h23m
busybox-deployment-77b7f87778-7pxps   1/1     Running       0            25s
busybox-deployment-77b7f87778-w2n5q   1/1     Running       0            36s
nginx-pod                             1/1     Running       0            42m
```

After a while the update has completed. The old pods were safely been terminated, when the new pods were up and ready to receive traffic. 

```sh
❯ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
alpine                                1/1     Running   0          104m
busybox-deployment-77b7f87778-7pxps   1/1     Running   0          42s
busybox-deployment-77b7f87778-w2n5q   1/1     Running   0          53s
nginx-pod                             1/1     Running   0          42m
```

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
busybox-deployment-77b7f87778-7pxps   1/1     Running   0          11m
busybox-deployment-77b7f87778-w2n5q   1/1     Running   0          11m
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
busybox-deployment-77b7f87778-7pxps   1/1     Running            0             19m
busybox-deployment-77b7f87778-w2n5q   1/1     Running            0             14m
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

## Interact with a Container

You can start a shell inside a running Container and attach your terminal to it.

Run an alpine container (which is a light-weight version of Linux) and prevent it from completing and shutting down with the sleep command.

```sh
❯ kubectl run alpine --image alpine -- sleep 1d
pod/alpine created
```

Check that it comes up and runs.

```sh
❯ k get pods
NAME          READY   STATUS             RESTARTS       AGE
alpine        1/1     Running            0              15s
```

Spawn a shell inside the container and attach your terminal to it in interactive mode.

```sh
❯ k exec -it alpine -- sh
/ $ cat /etc/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.20.1
PRETTY_NAME="Alpine Linux v3.20"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
/ $
```

## Exposing a Pod

This part is to showcase that Pods can be exposed externally outside the cluster. They can also be exposed to other Pods within the cluster.

Run a Pod from the nginx web server image and expose port 80 on the container.

```sh
❯ kubectl run nginx-pod --image nginx --port 80
pod/nginx-pod created
```

Check the Pods.

```sh
❯ k get pod
NAME                                  READY   STATUS      RESTARTS        AGE
alpine                                1/1     Running     0               62m
busybox-deployment-77b7f87778-7pxps   1/1     Running     1 (3h18m ago)   3h46m
busybox-deployment-77b7f87778-w2n5q   1/1     Running     1 (3h18m ago)   3h41m
hello-world-pod                       0/1     Completed   0               3h27m
nginx-pod                             1/1     Running     0               37s
```

Expose the Pod by creating a Service, which is another kind of Kubernetes object. They provide a stable endpoints (IP and port) for workloads. Remember, Pods are ephemeral and so are their IP adresses. So to ensure that applications can realiably connect to eachother, we need Services.

The Service `type` is specified as `NodePort`, which means that the Service makes the Pod accessible from outside the Kubernetes cluster by opening a specific port on each node in the cluster.

```sh
❯ kubectl expose pod nginx-pod --port 80 --type NodePort --name nginx-pod-service
service/nginx-pod-service exposed
```

Let's check the list of services.

```sh
❯ kubectl get services
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes          ClusterIP   10.43.0.1      <none>        443/TCP        270d
nginx-pod-service   NodePort    10.43.129.77   <none>        80:30898/TCP   4s
```

We see our service there and note the port number: `30898`. This is the port number which has been opened on the nodes in the cluster.

There is, however, only one node in this cluster and it is accessible via `localhost`. Let's `curl` localhost on that port.

```sh
❯ curl localhost:30898
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

We get a reply with the default nginx homepage.

## Distribution of workloads

To showcase the automatic distribution of workloads between available worker nodes, we need to switch cluster. I have a different cluster running on my local machine consisting of two worker nodes and a master node.

To do that we have to interact with the kubeconfig file, which just stores information about clusters that you can connect to and credentials. A 'context' is a combination of a cluster and a set of credentials.

Let's check the available contexts.

```sh
❯ kubectl config get-contexts
CURRENT   NAME                   CLUSTER                AUTHINFO                     NAMESPACE
          k3d-two-node-cluster   k3d-two-node-cluster   admin@k3d-two-node-cluster
*         rancher-desktop        rancher-desktop        rancher-desktop
```

Switch to the other cluster/context.

```sh
❯ kubectl config use-context k3d-two-node-cluster
Switched to context "k3d-two-node-cluster".
```

Let's check the nodes.

```sh
❯ kubectl get nodes
NAME                            STATUS   ROLES                  AGE   VERSION
k3d-two-node-cluster-server-0   Ready    control-plane,master   10d   v1.28.8+k3s1
k3d-two-node-cluster-agent-0    Ready    <none>                 10d   v1.28.8+k3s1
k3d-two-node-cluster-agent-1    Ready    <none>                 10d   v1.28.8+k3s1
```

Now let's run two busybox Pods and then check which nodes they get scheduled to.

```sh
❯ kubectl run busybox-1 --image busybox -- sleep 1d
pod/busybox-1 created

❯ kubectl run busybox-2 --image busybox -- sleep 1d
pod/busybox-2 created

❯ kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE                           NOMINATED NODE   READINESS GATES
busybox-1   1/1     Running   0          76s   10.42.1.29   k3d-two-node-cluster-agent-1   <none>           <none>
busybox-2   1/1     Running   0          56s   10.42.0.23   k3d-two-node-cluster-agent-0   <none>           <none>
```

We can see that the Pods are assigned to different nodes.

## Tips

You can always get help and examples.

```sh
❯ kubectl get --help
Display one or many resources.

 Prints a table of the most important information about the specified resources. You can filter the list using a label
selector and the --selector flag. If the desired resource type is namespaced you will only see results in your current
namespace unless you pass --all-namespaces.

 By specifying the output as 'template' and providing a Go template as the value of the --template flag, you can filter
the attributes of the fetched resources.

Use "kubectl api-resources" for a complete list of supported resources.

Examples:
  # List all pods in ps output format
  kubectl get pods

  # List all pods in ps output format with more information (such as node name)
  kubectl get pods -o wide

(...)
```

```sh
❯ kubectl explain pods
KIND:       Pod
VERSION:    v1

DESCRIPTION:
    Pod is a collection of containers that can run on a host. This resource is
    created by clients and scheduled onto hosts.

FIELDS:
(...)
```