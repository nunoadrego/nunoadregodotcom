+++
title = "kubectl Dry Run"
date = "2022-05-22T16:00:00Z"
author = "Nuno Adrego"
authorTwitter = "nunoadrego" #do not include @
cover = ""
tags = ["kubectl", "kubernetes"]
keywords = ["kubectl", "Kubernetes"]
description = "A useful flag"
showFullContent = false
+++

![kubectl --dry-run=client](/static_en/kubectl-dry-run.png)

On day-to-day operations in Kubernetes, I find myself frequently using the kubectl flag `--dry-run=client`. I use it to generate definitions of objects in yaml by pairing it with `-o yaml`. For example:

```bash
kubectl run test --image busybox --dry-run=client -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: test
  name: test
spec:
  containers:
  - image: busybox
    name: test
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

I learned this flag while studying for [CKAD certification](https://www.cncf.io/certification/ckad/), and it was handy for the exam. However, I found it verbose at the time. “Why couldn’t I just pass `--dry-run`?” I thought. 🤔 There were other arguments: `none` and `server`. I didn’t dive in much and either used the `client` argument or didn’t specify the flag.

Fast forward to today, while watching the session *The Hitchhiker's Guide to Pod Security* by [Lachlan Evenson](https://twitter.com/LachlanEvenson) in KubeCon EU 2022, I found the usage of `--dry-run` but this time with `server` argument! 🤯 And it finally makes sense.

# none, client and server

`--dry-run` supports three values. The help states the following:

```
--dry-run='none':
	Must be "none", "server", or "client". If client strategy, only print the object that would be sent, without
	sending it. If server strategy, submit server-side request without persisting the resource.
```

## server

In Lachlan’s talk, he uses the `server` argument and not the `client` one. Why? Because he wants to validate his object against the [Pod Security](https://kubernetes.io/docs/concepts/security/pod-security-admission/) admission controller[^1]. That’s only possible if a request to the kube-apiserver goes through the typical stages up until persisting in storage.

[^1]: You can read more about [admission controllers](https://nunoadrego.com/posts/opa-gatekeeper-in-the-admission-controllers-world/#kubernetes-admission-controllers) in my previous post.

Let’s repeat the same command I ran before, but this time with the `server` argument:

```bash
kubectl run test --image busybox --dry-run=server -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2022-05-22T13:06:26Z"
  labels:
    run: test
  name: test
  namespace: default
  uid: c85d8c20-5a6d-4e40-8211-63bf08dea293
spec:
  containers:
  - image: busybox
    imagePullPolicy: Always
    name: test
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-9j4wv
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-9j4wv
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  phase: Pending
  qosClass: BestEffort
```

As we can see, there is a lot more information, and it is similar to what we would persist if we created the resource without a dry run.

## client

On the other hand, with the `client` argument, commands print the object's definition, and there are no admission validations or mutations made to it.

Does this mean that whith this argument the command doesn’t communicate with the kube-apiserver at all? 🤔 Let's find out!

```bash
unset KUBECONFIG
kubectl run test --image busybox --dry-run=client -o yaml
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

Oops. It seems we can’t use `client` argument without a kubeconfig/context set.

Now with a cluster running with kubeconfig and context set, let's try to create a Service to expose a Deployment that does not exist:

```bash
kubectl expose deploy test --dry-run=client -o yaml
Error from server (NotFound): deployments.apps "test" not found
```

No luck. Before generating the yaml for the Service, kubectl asks the kube-apiserver if the given Deployment we want to expose exists. I think we are convinced now.

Furthermore, I found that the result of `--dry-run=client` is used when we opt for `--dry-run=server`. We can observe this by increasing the [verbosity level of kubectl](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-output-verbosity-and-debugging).

```bash
kubectl -v 8 run test --image busybox --dry-run=server -o yaml
```

```bash
# truncated
I0522 14:14:56.913043   42312 request.go:1073] Request Body: {"kind":"Pod","apiVersion":"v1","metadata":{"name":"test","creationTimestamp":null,"labels":{"run":"test"}},"spec":{"containers":[{"name":"test","image":"busybox","resources":{}}],"restartPolicy":"Always","dnsPolicy":"ClusterFirst"},"status":{}}
I0522 14:14:56.913095   42312 round_trippers.go:463] POST https://127.0.0.1:54603/api/v1/namespaces/default/pods?dryRun=All&fieldManager=kubectl-run
I0522 14:14:56.913098   42312 round_trippers.go:469] Request Headers:
I0522 14:14:56.913103   42312 round_trippers.go:473]     Content-Type: application/json
I0522 14:14:56.913106   42312 round_trippers.go:473]     User-Agent: kubectl/v1.24.0 (darwin/arm64) kubernetes/4ce5a89
I0522 14:14:56.913110   42312 round_trippers.go:473]     Accept: application/json, */*
I0522 14:14:56.917812   42312 round_trippers.go:574] Response Status: 201 Created in 4 milliseconds
I0522 14:14:56.917831   42312 round_trippers.go:577] Response Headers:
I0522 14:14:56.917836   42312 round_trippers.go:580]     Audit-Id: 356e5ee5-5927-41ea-8fbe-285634118fc0
I0522 14:14:56.917841   42312 round_trippers.go:580]     Cache-Control: no-cache, private
I0522 14:14:56.917844   42312 round_trippers.go:580]     Content-Type: application/json
I0522 14:14:56.917848   42312 round_trippers.go:580]     X-Kubernetes-Pf-Flowschema-Uid: 7d779e83-3bee-40ca-a14f-f9e93da4b1e2
I0522 14:14:56.917852   42312 round_trippers.go:580]     X-Kubernetes-Pf-Prioritylevel-Uid: 7a339427-2bcc-4d7b-aa7b-30034d83a0d9
I0522 14:14:56.917855   42312 round_trippers.go:580]     Content-Length: 1946
I0522 14:14:56.917859   42312 round_trippers.go:580]     Date: Sun, 22 May 2022 13:14:56 GMT
I0522 14:14:56.917892   42312 request.go:1073] Response Body: {"kind":"Pod","apiVersion":"v1","metadata":{"name":"test","namespace":"default","uid":"6d9ed2c1-2cff-400d-950b-24d34efdf9be","creationTimestamp":"2022-05-22T13:14:56Z","labels":{"run":"test"},"managedFields":[{"manager":"kubectl-run","operation":"Update","apiVersion":"v1","time":"2022-05-22T13:14:56Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:labels":{".":{},"f:run":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"test\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-lqrqg","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1"," [truncated 922 chars]
# truncated
```

Check the `Request Body` of the truncated output. Sounds familiar, right? 😛

## none

Finally, `none` is like not using the flag at all, meaning it’s not a dry run, and the request will be made and persisted if it succeeds.

# **Summarising**

To generate objects' definitions from commands like `kubectl run`, `kubectl expose`, `kubectl create namespace`, or others, use `--dry-run=client`.

To validate or observe mutations done by admission controllers and get more sense of how a given object would be stored, use `--dry-run=server`.

Lastly, if we don’t want to do a dry run, we can either use `--dry-run=none` or do not pass the flag.
