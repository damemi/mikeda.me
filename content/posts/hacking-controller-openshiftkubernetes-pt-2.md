---
title: "Hacking a Controller for OpenShift/Kubernetes, Pt. 2"
date: 2016-08-23T17:10:17
draft: false
slug: hacking-controller-openshiftkubernetes-pt-2
categories:
  - Uncategorized
wordpress_id: 67
---
**[Part 1: Introduction to the OpenShift client](http://www.mikeda.me/hacking-controller-openshiftkubernetes/)[Part 3: Writing a controller](http://www.mikeda.me/hacking-controller-openshiftkubernetes-pt-3/)**

In [my last post](http://www.mikeda.me/hacking-controller-openshiftkubernetes/), I went over how to set up an OpenShift environment for developing. That tutorial used the [OpenShift Client API](https://godoc.org/github.com/openshift/origin/pkg/client) to make function calls which interacted with our cluster. However, there may be a situation where you need more direct interaction with your cluster's resources (or perhaps you are interested in contributing to the [open-source OpenShift Origin repository](https://github.com/openshift/origin)), and even the provided function calls aren't enough to satiate your needs. In this case, it's good to know how OpenShift interacts with Kubernetes directly to serve your cluster resources hot-and-ready.

## Kubernetes Resource Objects

Going back to the code we used in the last post, you can recall that we used an [OpenShift Project Interface](https://godoc.org/github.com/openshift/origin/pkg/client#ProjectInterface) to get a list of the projects in our cluster. If you examine the [code for the Project Interface](https://github.com/openshift/origin/blob/master/pkg/client/projects.go#L44-L52), you can see that it uses a [REST Client](https://godoc.org/k8s.io/kubernetes/pkg/client/restclient#RESTClient) to get the requested information. However, certain commands such as oc get (which is really just a [wrapper](https://github.com/openshift/origin/blob/cb596dc6d579f4b76e95d9eb9e5f147542968203/pkg/cmd/cli/cmd/wrappers.go) for Kubernetes' kubectl get command) rely on the [Kubernetes Client API](https://godoc.org/k8s.io/kubernetes/pkg/kubectl/resource) to [request the necessary resource objects](https://github.com/openshift/origin/blob/b81f65c6fc829e387a130e5041ceba2b2a12e71a/vendor/k8s.io/kubernetes/pkg/kubectl/cmd/get.go#L179-L187). How exactly Kubernetes achieves this can be a bit confusing, so let's modify our code from the last blog post to use a Kubernetes client (as opposed to the OpenShift client) and walk through it as an example of the advantages using OpenShift can give you as a developer.

Update your **pkg/controller/controller.go **file to look like this:
package controller

import (
        "fmt"

        osclient "github.com/openshift/origin/pkg/client"
        "github.com/openshift/origin/pkg/cmd/util/clientcmd"

        "github.com/spf13/pflag"
        kapi "k8s.io/kubernetes/pkg/api"
        "k8s.io/kubernetes/pkg/api/meta"
        "k8s.io/kubernetes/pkg/kubectl/resource"
        kclient "k8s.io/kubernetes/pkg/client/unversioned"
        "k8s.io/kubernetes/pkg/runtime"
)

type Controller struct {
        openshiftClient *osclient.Client
        kubeClient      *kclient.Client
        mapper          meta.RESTMapper
        typer           runtime.ObjectTyper
        f               *clientcmd.Factory
}

func NewController(os *osclient.Client) *Controller {

        // Create mapper and typer objects, for use in call to Resource Builder
        f := clientcmd.New(pflag.NewFlagSet("empty", pflag.ContinueOnError))
        mapper, typer := f.Object()

        return &Controller{
                openshiftClient: os,
                kubeClient:      kc,
                mapper:          mapper,
                typer:           typer,
                f:               f,
        }
}

func (c *Controller) Run() {
        /*
                // Old code from last post using OpenShift client
                projects, err := c.openshiftClient.Projects().List(kapi.ListOptions{})
                if err != nil {
                        fmt.Println(err)
                }
                for _, project := range projects.Items {
                        fmt.Printf("%s\n", project.ObjectMeta.Name)
                }
        */

        // Resource Builder function call, to get Result object
        r := resource.NewBuilder(c.mapper, c.typer, resource.ClientMapperFunc(c.f.ClientForMapping), kapi.Codecs.UniversalDecoder()).
                ResourceTypeOrNameArgs(true, "projects").
                Flatten().
                Do()

        // Use Visitor interface to iterate over Infos in previous Result
        err := r.Visit(func(info *resource.Info, err error) error {
                fmt.Printf("%s\n", info.Name)
                return nil
        })
        if err != nil {
                fmt.Println(err)
        }
}

Build and run and you should see the same output as you did before. So what did we change here?

**Some new imports**

We added the clientcmd and pflag packages so we can use them to create a [Factory object](https://godoc.org/k8s.io/kubernetes/pkg/kubectl/cmd/util#Factory), which gives us our mapper and typer (more on that in a bit). This part could have been done in our main **cmd/controller/cmd.go** file, with the Factory object passed to the new controller as a parameter, but for brevity I just added it here. meta and runtime are also for the mapper and typer, respectively. Finally, resource  allows us to interact with the [Kubernetes Resource Builder](https://godoc.org/k8s.io/kubernetes/pkg/kubectl/resource#Builder) client functions.

**Resource Builder**

The resource package provides us with client functions of the Builder type. The call to **NewBuilder() **takes four arguments: a [RESTMapper](https://godoc.org/k8s.io/kubernetes/pkg/api/meta#RESTMapper), an [ObjectTyper](https://godoc.org/k8s.io/kubernetes/pkg/runtime#ObjectTyper), a [ClientMapper](https://godoc.org/k8s.io/kubernetes/pkg/kubectl/resource#ClientMapper), and a [Decoder](https://godoc.org/k8s.io/kubernetes/pkg/runtime#Decoder) (the names give a pretty good idea of what each object does, but I've linked to their docs pages if you want to know more). The Builder type provides numerous functions which serve as parameters in a request for resources. In this case, I call ResourceTypeOrNameArgs(true, "projects") and Flatten() on my Builder. The **ResourceTypeOrNameArgs() **call lets me specify which type of resource I'd like, and request any objects of that specific type by name. Since I just want all of the projects in the cluster, though, I set the first parameter to "true" ([which allows me to blankly select all resources](https://godoc.org/k8s.io/kubernetes/pkg/kubectl/resource#Builder.ResourceTypeOrNameArgs)). The Resource Builder **Flatten()** function returns the results as an iterable list of Info objects (but that's getting a little ahead of ourselves). Finally, Do() returns a [Result object](https://godoc.org/k8s.io/kubernetes/pkg/kubectl/resource#Result).

**The "Result"**

In my opinion, this is kind of a semantic misnomer. For the developer new to Kubernetes, it would be assumed that the "result" is the data you originally requested. In reality, it's an object containing metadata about the result as well as a way to *access* the actual data, through structures called [Infos](https://godoc.org/k8s.io/kubernetes/pkg/kubectl/resource#Info). There are a few ways to get to these Info objects, one is to simply call .Infos() on the Result object to return a list of Infos. Another, slightly more elegant method, is to use the [Visitor Interface](https://godoc.org/k8s.io/kubernetes/pkg/kubectl/resource#Result.Visit).

**Visitor Function**

Calling .Visit() on a Result object allows you to provide a function which will iterate over each Info object in the Result. Infos themselves provide some helpful metadata on the resource they describe, such as Name and Namespace, but they also give you access to the full generic runtime.Object representation of the resource. By casting these objects to their actual types, you can access the fields and methods specific to that type. As an example, let's update our **Visit()** function like so:
err := r.Visit(func(info *resource.Info, err error) error {
        switch t := info.Object.(type) {
        case *projectapi.Project:
                fmt.Printf("%s is currently %s\n", t.ObjectMeta.Name, t.Status.Phase)
        default:
                return fmt.Errorf("Unknown type")
        }
        return nil
})
And also add the following line to your imports: projectapi "github.com/openshift/origin/pkg/project/api". Save, build, and run and you'll see output like this:
default is currently Active
openshift is currently Active
openshift-infra is currently Active

Now we're casting the runtime.Object to a Project type and, using the [OpenShift Project API](https://godoc.org/github.com/openshift/origin/pkg/project/api), getting information about its name and status. As a side-note, this makes use of [Go's type switching](https://golang.org/doc/effective_go.html#type_switch) which is very cool.

## Summary

To summarize, Kubernetes' method for retrieving your objects goes through several different types and interfaces: *Builder -> Result -> Visitor -> Info -> Object -> Typecast*. Normally, this approach would be more appropriate if you were writing your own command-line arguments. It's very helpful to have an understanding of how Kubernetes interacts with your cluster on a low-level, but as you can see it's much simpler to use OpenShift client function calls to get the information you want. Our example here is a bit acrobatic, but still demonstrates the flexibility that working with OpenShift and Kubernetes provides.

In the next post, I'll go over how to actually make your controller run like a controller (asynchronously, listening for updates) using the [Watch package](https://godoc.org/k8s.io/kubernetes/pkg/watch).

[**Click for Part 3: Writing a controller**](http://www.mikeda.me/hacking-controller-openshiftkubernetes-pt-3/)
