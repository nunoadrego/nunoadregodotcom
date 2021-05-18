+++
title = "How to Generate Static Kubernetes Architecture Diagrams From State"
date = ""
author = "Nuno Adrego"
authorTwitter = "nunoadrego" #do not include @
cover = ""
tags = ["kubernetes", "visualization"]
keywords = ["Kubernetes", "visualization"]
description = "When you don't want to mess with draw.io"
showFullContent = false
+++

Architecture diagrams play an important role in the documentation of a system.

When I need to draft a solution, my favorite tool is draw.io[^1]. It is simple to use, it has a field to search and include symbols, and it integrates with Google Drive.

[^1]: [draw.io](http://draw.io) was rebranded to [diagrams.net](http://diagrams.net).

Recently I had to revisit one of those drafts to create some documentation for a small system that was already deployed in Kubernetes. Since the initial draft, the system suffered some updates and I was not in the mood to redesign it in draw.io. _There must be a better way..._ 💭 And there is.

# k8sviz

My first search result was [k8sviz](https://github.com/mkimuram/k8sviz) by Masaki Kimura. By quickly reading the `README`, it seemed to be easy to install and run, so I gave it a try.

> k8sviz is a tool to generate Kubernetes architecture diagrams from the actual state in a namespace.

I also noticed this tool's diagrams are generated with Graphviz, the same technology PlantUML[^2] uses.

[^2]: [PlantUM](https://plantuml.com/) is a tool that allows the creation of sequence, usecase, class, and other diagrams as code.

## Installation

There are two installation options - Bash and Go. I opted for the Go version:

```bash
git clone https://github.com/mkimuram/k8sviz.git
cd k8sviz
make build
```

You must be able to use it by running:

```bash
./bin/k8sviz -h
```

However, if you prefer to access the binary from a different path in your system, you need to move both the binary and the icons' folder to a path such as `$HOME/.local/bin`, for example:

```bash
PATH_TO_INSTALL=$HOME/.local/bin
cp bin/k8sviz ${PATH_TO_INSTALL}
cp -r icons ${PATH_TO_INSTALL}
```

## Usage and Results

For demo purposes, I will use a standard minikube cluster and choose to visualize the `kube-system` namespace:

```bash
minikube start
kubectl get all -n kube-system
```

The cluster is up and running and I can see a Deployment, a DaemonSet, a Service, and some Pods.

Now let's run k8sviz:

```bash
k8sviz -n kube-system -kubeconfig /Users/nuno/.kube/config -o k8sviz_test.png -t png
```

As a result, we obtain the following output file:

![k8sviz](/static_en/k8sviz.png)

In case you want to see more results, take a look at the [examples section](https://github.com/mkimuram/k8sviz#examples) of k8sviz `README`.
