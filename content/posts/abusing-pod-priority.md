+++
title = "Abusing Pod Priority"
date = "2022-08-27T17:00:00Z"
author = "Nuno Adrego"
authorTwitter = "nunoadrego" #do not include @
cover = ""
tags = ["podpriority", "priorityclass", "preemption", "eviction", "kubernetes"]
keywords = ["PodPriority","PriorityClass", "Preemption", "Eviction", "Kubernetes"]
description = "Pod Priority from a malicious user standpoint"
showFullContent = false
+++

![kubectl --dry-run=client](/static_en/pod-priority-abuse.jpg)

Killer Coda is a well-known platform that hosts interactive environments for studying cloud native technologies. While doing their [CKA scenarios](https://killercoda.com/killer-shell-cka), I found an intriguing one called *Scheduling Priority*.

Since I was not familiar with *Pod Priority* or `PriorityClass` concepts at the time, I did the usual - searched for them in the Kubernetes docs.

At the top of the [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) page, we can see a red warning:

> **Warning:**
> 
> 
> In a cluster where not all users are trusted, a malicious user could create Pods at the highest possible priority, causing other Pods to be evicted/not get scheduled. An administrator can use ResourceQuota to prevent users from creating pods at high priorities.
> 
> See [limit Priority Class consumption by default](https://kubernetes.io/docs/concepts/policy/resource-quotas/#limit-priority-class-consumption-by-default) for details.
> 

This message got my attention, and I wanted to see it in action.

# Pod Priority

The feature's name says it all: *Pod Priority* is a way to give more importance to some `Pods` than others. `PriorityClasses` manage the different levels of priority.

A `PriorityClass` definition looks like this:
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
preemptionPolicy: PreemptLowerPriority
value: 1000
globalDefault: false
description: This priority class should be used for X pods only.
```

To assign priority to a `Pod`, the `spec` of the `Pod` must contain the field `priorityClassName` with the correspondent `PriorityClass`, just as below:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test
  name: test
spec:
  containers:
  - image: busybox
    name: test
  restartPolicy: Always
  priorityClassName: high-priority
```

It's also possible to configure one `PriorityClass` with `globalDefault: true`. After that, all new `Pods` without an explicit `priorityClassName` will be mutated[^1] and receive the default priority of the cluster.

[^1]: You can read more about [mutating admission controllers](https://nunoadrego.com/posts/opa-gatekeeper-in-the-admission-controllers-world/#kubernetes-admission-controllers) in a previous post.

After this brief introduction, we are ready to move to the following hands-on sections! 🧪

All commands and configuration files are available in my [GitHub repository](https://github.com/nunoadrego/pod-priority).

# Spin up a cluster, explore and create a deployment

Let’s start by spinning up a fresh Kubernetes cluster locally with minikube:

```bash
minikube start
```

With the output of the following command, we can conclude that our cluster has one node:

```bash
kubectl get nodes
```

```bash
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   5s    v1.24.3
```

That node has 4 CPU cores allocatable with 18% already requested:

```bash
kubectl describe node minikube
```

```bash
# truncated
Allocatable:
  cpu:                4
  ephemeral-storage:  61255492Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             8039920Ki
  pods:               110
# truncated
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                750m (18%)  0 (0%)
  memory             170Mi (2%)  170Mi (2%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  hugepages-32Mi     0 (0%)      0 (0%)
  hugepages-64Ki     0 (0%)      0 (0%)
# truncated
```

For this scenario, we will create a *virtuous* deployment with 2 CPU cores as requests to simulate a well-behaved 👼 application. Notice that we **don't assign** a `PriorityClass` to it:

```bash
kubectl apply -f virtuous-deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: virtuous
  name: virtuous
spec:
  replicas: 1
  selector:
    matchLabels:
      app: virtuous
  template:
    metadata:
      labels:
        app: virtuous
    spec:
      containers:
      - command:
        - sh
        - -c
        - sleep 1d
        image: busybox
        name: virtuous
        resources:
          requests:
            cpu: 2
```

Speaking of which, do we have `PriorityClasses` in the cluster?

```bash
kubectl get priorityclass
```

```bash
NAME                      VALUE        GLOBAL-DEFAULT   AGE
system-cluster-critical   2000000000   false            32s
system-node-critical      2000001000   false            32s
```

Yes, we do. Two PriorityClasses: `system-cluster-critical` and `system-node-critical`, with the latter being the one with the highest priority. Let’s see if we have `Pods` with `PriorityClasses` specified in our cluster:

```bash
kubectl get pods --all-namespaces -o custom-columns=POD:.metadata.name,PRIORITY_CLASS:.spec.priorityClassName,PRIORITY:.spec.priority,CPU_REQUESTS:'.spec.containers[*].resources.requests.cpu'
```

```bash
POD                                PRIORITY_CLASS            PRIORITY     CPU_REQUESTS
virtuous-7bc68f75fb-xpjz2          <none>                    0            2
coredns-6d4b75cb6d-htbdx           system-cluster-critical   2000000000   100m
etcd-minikube                      system-node-critical      2000001000   100m
kube-apiserver-minikube            system-node-critical      2000001000   250m
kube-controller-manager-minikube   system-node-critical      2000001000   200m
kube-proxy-2zgz7                   system-node-critical      2000001000   <none>
kube-scheduler-minikube            system-node-critical      2000001000   100m
storage-provisioner                <none>                    0            <none>
```

We have two `Pods` without `PriorityClass` and two `Pods` without CPU requests. We also have one `Pod` with the lowest `PriorityClass` defined (`system-cluster-critical`). Since we didn’t specify a `PriorityClass` for the *virtuous* `Pod`, its priority is zero.

# Attack

Time for the malicious user to get some action. 😈

If we describe our node again:

```bash
kubectl describe node minikube
```

```bash
#truncated
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                2750m (68%)  0 (0%)
  memory             170Mi (2%)   170Mi (2%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)
  hugepages-32Mi     0 (0%)       0 (0%)
  hugepages-64Ki     0 (0%)       0 (0%)
#truncated
```

Making the math (4-2.75), we only have 1.25 cores of CPU available to be requested. What happens if we request 3.3 cores of CPU and use the highest `PriorityClass` in the cluster? 🐒

```yaml
kubectl apply -f evil-deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: evil
  name: evil
spec:
  replicas: 1
  selector:
    matchLabels:
      app: evil
  template:
    metadata:
      labels:
        app: evil
    spec:
      containers:
      - command:
        - sh
        - -c
        - sleep 1d
        image: busybox
        name: evil
        resources:
          requests:
            cpu: 3.3
      priorityClassName: system-node-critical
```

The result is:

```bash
kubectl get pods --all-namespaces
```

```yaml
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
default       evil-7568b8bbf5-4fq5d              1/1     Running   0             37s
default       virtuous-7bc68f75fb-pf95j          0/1     Pending   0             37s
kube-system   coredns-6d4b75cb6d-85x7r           0/1     Pending   0             37s
kube-system   etcd-minikube                      1/1     Running   0             114s
kube-system   kube-apiserver-minikube            1/1     Running   0             114s
kube-system   kube-controller-manager-minikube   1/1     Running   0             114s
kube-system   kube-proxy-2zgz7                   1/1     Running   0             102s
kube-system   kube-scheduler-minikube            1/1     Running   0             114s
kube-system   storage-provisioner                1/1     Running   1 (71s ago)   113s
```

Both *virtuous* and *coredns* `Pods` terminated, and new ones are now pending! Evil Pod is running.

If you take a closer look, we can understand why. Our evil `Pod` requests 3.3 cores of CPU and has the highest *Pod Priority* specified (`system-node-critical`). Since the only node in the cluster has 1.25 cores of CPU available to be requested, there is a need to kill `Pods` with lower priority. In this case, since *virtuous* `Pod` and *coredns* were the ones with the lowest priority and with CPU requests specified, the Kubernetes scheduler preempted them.

If the *evil* `Pod` didn't specify a `PriorityClass`, it would be pending due to a failed schedule.

We didn't highlight *Preemption* in the introduction section for drama purposes. `PriorityClasses` can state their `preemptionPolicy`, which by default is `PreemptLowerPriority`, but can also be `Never`. Both `PriorityClasses` shipped with Kubernetes clusters (`system-cluster-critical` and `system-node-critical`) have the policy `PreemptLowerPriority`.

# Clean up

```bash
minikube delete
```

# Conclusion

Pod Priority can be useful for some use cases such as prioritizing critical applications, but definitely can catch us off guard if we don’t have the right guardrails in place. This post illustrates potential consequences of not having them.
