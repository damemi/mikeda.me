---
title: "Hacking a Controller for OpenShift/Kubernetes"
date: 2016-08-22T20:05:02
draft: false
slug: hacking-controller-openshiftkubernetes
categories:
  - Uncategorized
wordpress_id: 49
---
**[Part 2: Coding for Kubernetes](http://www.mikeda.me/hacking-controller-openshiftkubernetes-pt-2/)[Part 3: Writing a controller](http://www.mikeda.me/hacking-controller-openshiftkubernetes-pt-3/)**

For OpenShift Online, we run several controllers in our cluster which serve functions such as provisioning persistent volumes and providing user analytics. But let's say you have your *own* OpenShift cluster, upon which you'd like to run a controller that interacts with the resources in that cluster. I'm going to run you through setting up OpenShift and Kubernetes in a way that allows you to develop your own controller. By the end of this guide, we'll have a simple controller that shows the cumulative running time for all pods in a namespace.

## Setting Up Your Environment

The prerequisites to develop for your OpenShift setup are:

- [OpenShift installed as per our "Contributing" docs](https://github.com/openshift/origin/blob/master/CONTRIBUTING.adoc#openshift-development) This includes installing [Go](http://golang.org/doc/install), [Git](http://git-scm.com/book/en/v2/Getting-Started-Installing-Git), and [Docker](https://docs.docker.com/installation/), as well as configuring your Go environment. That page will teach you how to build and run OpenShift from source, which is very helpful if you're interested in contributing to the project!

 	[Godep installed](https://github.com/tools/godep#install). This will be used for dependency management.

Assuming you've followed the instructions on that page to set up your GOPATH and have Origin cloned, the next step is to download the source code dependencies for OpenShift and Kubernetes. Do this with the following commands:
cd $GOPATH/src/github.com/openshift/origin
git checkout release-1.2

git clone git://github.com/kubernetes/kubernetes $GOPATH/src/k8s.io/kubernetes
cd $GOPATH/src/k8s.io/kubernetes
git remote add openshift git://github.com/openshift/kubernetes
git fetch openshift
git checkout v1.2.0-36-g4a3f9c5

git clone https://github.com/go-inf/inf.git $GOPATH/src/speter.net/go/exp/math/dec/inf

cd $GOPATH/src/github.com/openshift/origin
godep restore

What we're doing here is:

1. Checking out the most recent release branch of OpenShift
2. Cloning the Kubernetes repository
3. Adding OpenShift's vendored version of Kubernetes as a remote to our Kubernetes repository
4. Checking out the required release of OpenShift's Kubernetes *This can be found by opening **origin/Godeps/Godeps.json**, searching for "**Kubernetes**" and copying the version number specified in "**comment**"*

 	Cloning another dependency
 	And finally running godep restore to download the source for all the dependencies needed

At this point, we're ready to start coding!

## Creating Your Project

In this post, we're going to make a simple run-once program that lists the namespaces (projects) in a cluster. That means this program won't run continuously like you would think of a controller (we'll add that in a later post), but is more of a basic introduction to the client tools used to write such a program.

First, create a GitHub repo with the following file structure:
controller/
- cmd/
-- controller/
- pkg/
-- controller/

- **cmd/controller/ **will contain the main package file for your controller
- **pkg/controller/ **will contain source files for your controller package

Now create a file called **cmd/controller/cmd.go** with the following contents:
package main

import (
        "fmt"
        "log"
        "os"

        "github.com/damemi/controller/pkg/controller"
        _ "github.com/openshift/origin/pkg/api/install"
        osclient "github.com/openshift/origin/pkg/client"
        "github.com/openshift/origin/pkg/cmd/util/clientcmd"

        kclient "k8s.io/kubernetes/pkg/client/unversioned"
        "github.com/spf13/pflag"
)

func main() {
        var openshiftClient osclient.Interface
        config, err := clientcmd.DefaultClientConfig(pflag.NewFlagSet("empty", pflag.ContinueOnError)).ClientConfig()
        kubeClient, err := kclient.New(config)
        if err != nil {
                log.Printf("Error creating cluster config: %s", err)
                os.Exit(1)
        }
        openshiftClient, err = osclient.New(config)
        if err != nil {
                log.Printf("Error creating OpenShift client: %s", err)
                os.Exit(2)
        }
}

Save the file, close it, and run godep save ./... in your directory. You should see that your file structure has changed to:
controller/
- cmd/
-- controller/
- pkg/
-- controller/
- Godeps/
-- Godeps.json
-- Readme
- vendor/
-- github.com/
--- [...]
-- golang.org/
--- [...]
-- [...]
I've excluded some of the files, because they're just dependency source files. These are now included in your project, so feel free to commit and push this to your repo. One of the cool things about Godep vendoring in this way is that now, you can share your codebase and allow someone else to build it without needing to worry about submodules or other dependency issues.

*Note: *We won't be able to build our controller yet due to the fact that this code has some defined-and-unused variables, but it is enough to run Godep. For when we do start building our code, I'll be using a Makefile  with the following:
all:
        go install github.com/damemi/controller/cmd/controller
Just because it's easier to type "make" each time.

## Adding Some Functionality

As fun as setting up a project is, it's even more fun to make it do things. Create a file **pkg/controller/controller.go** with the following contents:
package controller

import (
        "fmt"

        osclient "github.com/openshift/origin/pkg/client"

        kclient "k8s.io/kubernetes/pkg/client/unversioned"
        kapi "k8s.io/kubernetes/pkg/api"
)

// Define an object for our controller to hold references to
// our OpenShift client
type Controller struct {
        openshiftClient *osclient.Client
        kubeClient *kclient.Client
}

// Function to instantiate a controller
func NewController(os *osclient.Client, ) *Controller {
        return &Controller{
                openshiftClient: os,
                kubeClient:      kc,
        }
}

// Our main function call
func (c *Controller) Run() {
        // Get a list of all the projects (namespaces) in the cluster
        // using the OpenShift client
        projects, err := c.openshiftClient.Projects().List(kapi.ListOptions{})
        if err != nil {
                fmt.Println(err)
        }

        // Iterate through the list of projects
        for _, project := range projects.Items {
                fmt.Printf("%s\n", project.ObjectMeta.Name)
        }
}

As you can see, we're using the [OpenShift API](https://godoc.org/github.com/openshift/origin/pkg/client) to request a [Project Interface](https://godoc.org/github.com/openshift/origin/pkg/client#ProjectInterface), which provides plenty of helper functions to interact with the projects in our cluster (in this case, we're using **List()**). The [Project API](https://godoc.org/github.com/openshift/origin/pkg/project/api#Project) is what allows us to actually interact with the meta data about each project object using the [kapi.ObjectMeta field](https://godoc.org/k8s.io/kubernetes/pkg/api#ObjectMeta). I highly recommend reading through the OpenShift and Kubernetes APIs to get an idea of what's really available for you.

Now let's also add the following lines to the **main()** function in our **cmd/controller/cmd.go** file:
c := controller.NewController(openshiftClient, kubeClient)
c.Run()

Making that entire function look like:
func main() {
        config, err := clientcmd.DefaultClientConfig(pflag.NewFlagSet("empty", pflag.ContinueOnError)).ClientConfig()
        kubeClient, err := kclient.New(config)
        if err != nil {
                log.Printf("Error creating cluster config: %s", err)
                os.Exit(1)
        }
        openshiftClient, err := osclient.New(config)
        if err != nil {
                log.Printf("Error creating OpenShift client: %s", err)
                os.Exit(2)
        }

        c := controller.NewController(openshiftClient, kubeClient)
        c.Run()
}

Now save, close, and run "make". Now, you should be able to just run **"controller"** from your command line and, assuming you have OpenShift running already and are logged in as **system:admin**, you should see some output like so:
default
openshift
openshift-infra

Hooray! We can connect to OpenShift and get information about our cluster. In the next post, we'll go into more detail about how OpenShift uses [Kubernetes' Resource API](https://godoc.org/k8s.io/kubernetes/pkg/kubectl/resource) to interact on a lower level with the cluster, as well as how to use the [Watch API](https://godoc.org/k8s.io/kubernetes/pkg/watch) to make our controller run asynchronously.

**[Click for Part 2: Kubernetes Resource Objects](http://www.mikeda.me/hacking-controller-openshiftkubernetes-pt-2/)**
