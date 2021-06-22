+++
title = "Playing With Hashicorp Waypoint on Kubernetes With minikube"
date = "2021-06-21T00:00:00Z"
author = "Nuno Adrego"
authorTwitter = "nunoadrego" #do not include @
cover = ""
tags = ["waypoint", "hashicorp", "kubernetes"]
keywords = ["Waypoint", "HashiCorp", "Kubernetes"]
description = "Build, Deploy and Release"
showFullContent = false
+++

One of the latest additions to the [HashiCorp](https://www.hashicorp.com/) line of products is [Waypoint](https://www.waypointproject.io/). Launched at the end of 2020 as an open source project, it is giving the first steps towards the goal of simplifying Developers' lives as well as Terraform did for Infrastructure folks.

> A consistent developer workflow to build, deploy, and release applications across any platform.
> -- <cite>Mitchell Hashimoto[^1]</cite>

[^1]: [Announcing HashiCorp Waypoint](https://www.hashicorp.com/blog/announcing-waypoint)

One of the things that immediately caught my attention was the fact of being a platform-agnostic solution. This means that whether you use Kubernetes, HashiCorp Nomad, Amazon ECS, Google Cloud Run, Azure Container Instances, Docker, and [more](https://www.waypointproject.io/plugins), you can use Waypoint to Build, Deploy, and Release your applications. This is something I haven't seen around, at least with mass adoption.

Since I've been involved in a team that works in an Internal Developer Platform, I couldn't resist taking a look and give Waypoint a try in a local environment.

# Installation

Depending on your Operating System, instructions differ. Take a look at the [official page](https://www.waypointproject.io/downloads).

For macOS users this does the trick:

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/waypoint
```

Now you should be able to use Waypoint CLI.

## Platform

In terms of platform I chose Kubernetes - it's the one I'm most familiar with. I will try others in the near future.

As a result, we need to launch a local Kubernetes cluster and run a command to [expose services of type `LoadBalancer`](https://minikube.sigs.k8s.io/docs/handbook/accessing/#using-minikube-tunnel) (it will be needed for a Waypoint service).

```bash
minikube start
minikube tunnel
```

Leave the shell with the last command running and perform the next steps in a new one.

## Waypoint Server And Waypoint Runner

Waypoint has the concepts of Server and Runner. The server is mandatory and it's what allows collaboration between members. Runners' purpose is to execute remote operations such as builds, deploys, polling projects for changes, etc.

Both can be installed with one command:

```bash
waypoint install -platform=kubernetes -k8s-advertise-internal -k8s-context minikube -accept-tos
```

Now you must be able to authenticate in the UI with:

```bash
waypoint ui -authenticate
```

![Waypoint UI](/static_en/waypoint-ui.png)

# Build, Deploy and Release

Let's build, deploy and release an application! We will use a basic Golang application for demo purposes and leverage [CNCF project Builpacks](https://buildpacks.io/) to create a container image. You can find the source code in my [GitHub repository](https://github.com/nunoadrego/waypoint/tree/main/waypoint-on-minikube).

`hello.go` file:

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", HelloServer)
	http.ListenAndServe(":8080", nil)
}

func HelloServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello Waypoint!")
}
```

`go.mod` file:

```go
module github.com/nunoadrego/waypoint/waypoint-on-minikube

// +heroku goVersion go1.16
go 1.16
```

And finally, the `waypoint.hcl` file:

```hcl
project = "waypoint-on-minikube"

labels = { "tech" = "golang" }

app "hello-waypoint" {
  build {
    use "pack" {}
    registry {
      use "docker" {
        image = "hello-waypoint"
        tag   = "0.0.1"
        local = true
      }
    }
  }

  deploy {
    use "kubernetes" {
      probe_path   = "/"
      replicas     = 1
      service_port = 8080
      probe {
        initial_delay = 4
      }
      labels = {
        env = "local"
      }
      annotations = {
        demo = "yes"
      }
    }
  }

  release {
    use "kubernetes" {
      port          = 8080
      load_balancer = true
    }
  }
}
```

The `waypoint.hcl` file could be simplified. I added more parameters than needed for exploration purposes. In this case, since it is a new application, I chose the `kubernetes` plugin, but an additional one, [`kubernetes-apply`](https://www.waypointproject.io/plugins/kubernetes#kubernetes-apply-platform), is available if you already have Kubernetes resource files.

Moreover, note that I chose docker as a local registry for the container image. To have access to that image inside our minikube Kubernetes cluster, we will run a command that configures your environment to re-use minikube's Docker daemon.

```bash
eval $(minikube docker-env)
```

Let's bootstrap:

```bash
waypoint init
```

The `waypoint init` command creates the Project `waypoint-on-minikube` and inside it the Application `hello-waypoint`.

![Waypoint Application UI](/static_en/waypoint-application-ui.png)

And finally:

```bash
waypoint up
```

You should see a URL pointing to your application.

The `waypoint up` command performs three steps. You can also run them one by one with `waypoint build`, `waypoint deploy`, and `waypoint release`.

# Interesting Features

There are two interesting features that Waypoint brings to developers: `logs` and `exec`.

With `waypoint logs`, we can see real-time application logs in the command line with minimal retention. They can also be seen in the UI. According to Waypoint docs, these [logs are not meant to replace other comprehensive logging solutions](https://www.waypointproject.io/docs/logs). This feature makes me think of `kubectl logs`.

![Waypoint UI](/static_en/waypoint-logs.png)

Regarding `waypoint exec`, it allows you to execute commands in the context of your application. For example, if the platform is Kubernetes, it allows you to drop a shell inside a container. Again, similar to the `kubectl exec` command.

Let's use it to print our `hello.go` file inside the container:

```bash
waypoint exec bash
Connected to deployment v1
heroku@hello-waypoint-01f8ntadxrdwncpwzebk4yzkdh-69844c45c4-62htp:/$ cd ~
heroku@hello-waypoint-01f8ntadxrdwncpwzebk4yzkdh-69844c45c4-62htp:~$ cat hello.go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	log.Printf("Starting Hello Waypoint App.")
	http.HandleFunc("/", HelloServer)
	http.ListenAndServe(":8080", nil)
}

func HelloServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello Waypoint!")
}
```

# Cleanup

Time to clean our environment. We will destroy our Application, uninstall Waypoint (Server and Runner), and destroy our Kubernetes cluster.

```bash
waypoint destroy -auto-approve
waypoint server uninstall -platform=kubernetes -auto-approve
minikube delete
```

# Final Remarks

Waypoint is an interesting project that might fulfill the needs of some organizations. The fact it abstracts the platform in which developers are deploying giving a consistent workflow is well played by HashiCorp.

I would personally recommend it for startups giving their first steps towards an Internal Developer Platform. Nevertheless, don't forget that it is a super early stage project!

I didn't cover it in the essay, but there is also a [GitOps feature launched in Waypoint 0.3](https://www.hashicorp.com/blog/announcing-hashicorp-waypoint-0-3-0). I wasn't able to replicate with my current local environment (due to the use of a local container registry), but I will explore more and bring news soon. 😄

Lastly, if you are curious to see how Waypoint would work with a given technology, take a look at the [waypoint-examples](https://github.com/hashicorp/waypoint-examples) git repository.

I'm looking forward to seeing the evolution of Waypoint in the next releases as well as test it on more platforms.
