+++
title = "OPA Gatekeeper in the Admission Controllers' World"
date = "2021-10-05T15:00:00Z"
author = "Nuno Adrego"
authorTwitter = "nunoadrego" #do not include @
cover = ""
tags = ["opa", "gatekeeper", "admission controllers", "kubernetes"]
keywords = ["OPA", "Gatekeeper", "Admission Controllers", "Kubernetes"]
description = "Enforcing CRD-based policies"
showFullContent = false
+++

A couple of weeks ago, while surfing the [Cloud Native Computing Foundation](https://www.cncf.io) (CNCF) website, I stumbled upon one of its graduate projects - [Open Policy Agent](https://www.openpolicyagent.org) (OPA). After reading some documentation on OPA's website, I got interested and decided to explore this open-source project.

OPA consists of a general-purpose policy engine. The main goal is to make decisions based on Input, Policies (written in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/)), and Data while decoupling the logic from the services' code.

![OPA](/static_en/opa.png "Source: https://www.openpolicyagent.org/docs")

There are many [use-cases for OPA](https://www.openpolicyagent.org/docs/latest/ecosystem/). Since I've been investing in learning more about Kubernetes, I opted for a project that integrates OPA and Kubernetes - [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs).

# Kubernetes Admission Controllers

Admission Controllers intercept and process requests made to the Kubernetes API. This means that if a request is denied, it's not persisted in `etcd` nor executed.

Some well-known admission controllers are [*ResourceQuota*](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#resourcequota), [*LimitRanger*](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#limitranger), [*NamespaceLifecycle*](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#namespacelifecycle), etc. They can be enabled with the `enable-admission-plugins` flag in the [*kube-apiserver*](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/).

There are, however, two special admission controllers: [MutatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook) and [ValidatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook). These controllers allow the [extension of Kubernetes API functionality via webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/). 

![Admission Webhooks](/static_en/admission-webhooks.png "Source: https://banzaicloud.com/blog/k8s-admission-webhooks")

Admission Controllers can be classified as "mutating", "validating" or both. Mutating controllers may modify the objects they admit. Validating controllers return a binary response - yes or no - according to the object contents.

# OPA Gatekeeper

Gatekeeper v3.0 allows the creation of policies based on Custom Resource Definitions (CRDs). It provides validating and mutating admission control through a customizable admission webhook.

![Gatekeeper v3.0](/static_en/gatekeeper-v3.png "Source: https://kubernetes.io/blog/2019/08/06/opa-gatekeeper-policy-and-governance-for-kubernetes")

## Hands-on Example

Let's imagine that you want to enforce the following policy:

> Pods can only be launched in a Namespace with a ResourceQuota defined.
> 

How can we enforce it in our Kubernetes cluster? In this hands-on example, we will go through it step by step. You can find the source code in my [Gatekeeper GitHub repository](https://github.com/nunoadrego/gatekeeper).

### Bootstrap

Create Kubernetes cluster with one admission plugin only:

```bash
minikube start --extra-config=apiserver.enable-admission-plugins=NodeRestriction
```

Install Gatekeeper with Helm (or other [options](https://open-policy-agent.github.io/gatekeeper/website/docs/install#installation)):

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts # first time only
helm install gatekeeper gatekeeper/gatekeeper -n gatekeeper-system --create-namespace
```

### Create And Test Policy

In this example, we will need to sync some data to OPA. It's not possible to know if a `Namespace` has a `ResourceQuota` without having access to all `ResourceQuotas`. The sync is possible with the [Config](https://open-policy-agent.github.io/gatekeeper/website/docs/sync/) resource:

```yaml
# sync.yaml
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config
  namespace: gatekeeper-system
spec:
  sync:
    syncOnly:
      - group: ""
        version: "v1"
        kind: "ResourceQuota"
```

```bash
kubectl apply -f sync.yaml
```

Then, we will create a `ConstraintTemplate`[^1] that contains the Rego policy:

[^1]: If you want to learn more about the fields of a ConstraintTemplate, check the [OPA Constraint Framework](https://github.com/open-policy-agent/frameworks/tree/master/constraint).

```yaml
# template.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresourcequota
  annotations:
    description: Requires Pods to launch in Namespaces with ResourceQuotas.
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResourceQuota
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresourcequota

        violation[{"msg": msg}] {
          input.review.object.kind == "Pod"
          requestns := input.review.object.metadata.namespace
          not data.inventory.namespace[requestns]["v1"]["ResourceQuota"]
          msg := sprintf("container <%v> could not be created because the <%v> namespace does not have ResourceQuotas defined", [input.review.object.metadata.name,input.review.object.metadata.namespace])
        }
```

```yaml
kubectl apply -f template.yaml
```

Finally, we create the `Constraint`, based on the previous `ConstraintTemplate`:

```yaml
# constraint.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResourceQuota
metadata:
  name: namespace-must-have-resourcequota
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - gatekeeper-system
```

```yaml
kubectl apply -f constraint.yaml
```

We are ready to test it!

To start, we will create a `Namespace` without a `ResourceQuota` and try to launch a `Pod` in it:

```yaml
# ns-without-quota.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: unbounded-namespace
```

```yaml
# blocked-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: blocked-pod
  namespace: unbounded-namespace
spec:
  containers:
  - image: nginx
    name: blocked-pod
```

```yaml
kubectl apply -f ns-without-quota.yaml
kubectl apply -f blocked-pod.yaml # Should give Error
```

The second `kubectl apply` should give an error (`Error from server`). If it does, our desired policy is being enforced as it should.

Now let's try to launch a `Pod` in a `Namespace` with a `ResourceQuota` associated:

```yaml
# ns-with-quota.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bounded-namespace
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: bounded-namespace-quota
  namespace: bounded-namespace
spec:
  hard:
    requests.cpu: 1
    requests.memory: 1Gi
```

```yaml
# allowed-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: allowed-pod
  namespace: bounded-namespace
spec:
  containers:
  - image: nginx
    name: allowed-pod
    resources:
      requests:
        cpu: 0.5
        memory: 500Mi
```

```yaml
kubectl apply -f ns-with-quota.yaml
kubectl apply -f allowed-pod.yaml # Should work
```

This time the `Pod` should be successfully launched since we are not violating the *namespace-must-have-resourcequota* `Constraint`.

What about previous Pods that were launched before the `Constraint` was created? Gatekeeper has an [Audit](https://open-policy-agent.github.io/gatekeeper/website/docs/audit) functionality that allows you to see violations.

For example:

```bash
kubectl describe K8sRequiredResourceQuota namespace-must-have-resourcequota
```

```
Name:         namespace-must-have-resourcequota
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  constraints.gatekeeper.sh/v1beta1
Kind:         K8sRequiredResourceQuota
Metadata:
  (...)
Spec:
  Match:
    Excluded Namespaces:
      gatekeeper-system
    Kinds:
      API Groups:

      Kinds:
        Pod
Status:
  (...)
  Total Violations:  7
  Violations:
    Enforcement Action:  deny
    Kind:                Pod
    Message:             container <coredns-78fcd69978-7vzw4> could not be created because the <kube-system> namespace does not have ResourceQuotas defined
    Name:                coredns-78fcd69978-7vzw4
    Namespace:           kube-system
    Enforcement Action:  deny
    Kind:                Pod
    Message:             container <etcd-minikube> could not be created because the <kube-system> namespace does not have ResourceQuotas defined
    Name:                etcd-minikube
    Namespace:           kube-system
    (...)
Events:                  <none>
```

In this case, we can see seven violations in the *kube-system* `Namespace` that we did not exclude in our `Contraint`.

### Bonus: Gatekeeper Policy Manager

There is an interesting read-only web UI called [Gatekeeper Policy Manager](https://github.com/sighupio/gatekeeper-policy-manager) (GPM). It's possible to run [locally](https://github.com/sighupio/gatekeeper-policy-manager#running-locally) in a container or deploy in the cluster with [Kustomize](https://github.com/sighupio/gatekeeper-policy-manager#deploying-gpm) or Helm:

```bash
git clone https://github.com/sighupio/gatekeeper-policy-manager.git
helm install gpm gatekeeper-policy-manager/chart --namespace gatekeeper-system --set config.secretKey=dummy
```

The result:

![Constraint Templates](/static_en/gpm-constraint-templates.png)

![Constraints](/static_en/gpm-constraints.png)

### Cleanup

```bash
helm list -A # list helm releases
helm uninstall gpm -n gatekeeper-system
helm uninstall gatekeeper -n gatekeeper-system
minikube delete
```

# Alternatives

Some alternatives to Gatekeeper that I did not explore but also look promising:

- [Kyverno](https://kyverno.io/) (CNCF Sandbox Project)
- [Kubewarden](https://www.kubewarden.io/) (SUSE Rancher)

# Final Remarks

OPA and Gatekeeper can be a good last line of defense by enforcing policies before anything is persisted. The Audit functionality can also be useful to detect existing violations that would otherwise be hard to find.

Go ahead and explore the [OPA Gatekeeper Library](https://github.com/open-policy-agent/gatekeeper-library) GitHub Repository that contains examples that can serve as inspiration.
