---
title: "Hacking a Controller for OpenShift/Kubernetes, Pt. 3"
date: 2016-08-25T14:25:14
draft: false
slug: hacking-controller-openshiftkubernetes-pt-3
categories:
  - Uncategorized
wordpress_id: 96
---
**[Part 1: Introduction to the OpenShift client](http://www.mikeda.me/hacking-controller-openshiftkubernetes/)
[Part 2: Coding for Kubernetes](http://www.mikeda.me/hacking-controller-openshiftkubernetes-pt-2/)**

In the [previous post](http://www.mikeda.me/hacking-controller-openshiftkubernetes-pt-2/), I went more in-depth into how Kubernetes works in the command line. For this post, let's step back to the code from Part 1 of this series and modify it to run continuously, showing the list of pods currently in the cluster and updating every time a new pod is created.

## The Watch Interface

Kubernetes' [Watch interface](https://godoc.org/k8s.io/kubernetes/pkg/watch#Interface) provides the ability to listen for [several different types of events](https://godoc.org/k8s.io/kubernetes/pkg/watch#Event) in the cluster. Using the [channel functionality built into Go](https://www.golang-book.com/books/intro/10#section2), this is perfect for us to set up an asynchronous controller that can run continuously. First we need to update our code to take advantage of channels. Go back to the code we had in Part 1 and change the **main()** function in your **cmd/controller/cmd.go** file to look like this:
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
        stopChan := make(chan struct{})
        c.Run(stopChan)

What we've done is created a channel that will be used to safely send "stop" signals to our Go routines when the program ends, and passed that channel to our **Run()** function. Now, update your **pkg/controller/controller.go** file to look like this:
package controller

import (
        "fmt"
        "time" // New import

        osclient "github.com/openshift/origin/pkg/client"
        "github.com/openshift/origin/pkg/cmd/util/clientcmd"

        "github.com/spf13/pflag"
        kapi "k8s.io/kubernetes/pkg/api"
        "k8s.io/kubernetes/pkg/api/meta"
        kclient "k8s.io/kubernetes/pkg/client/unversioned"
        "k8s.io/kubernetes/pkg/runtime"
        "k8s.io/kubernetes/pkg/util/wait" // New import
        "k8s.io/kubernetes/pkg/watch"  // New import
)

type Controller struct {
        openshiftClient *osclient.Client
        kubeClient      *kclient.Client
        mapper          meta.RESTMapper
        typer           runtime.ObjectTyper
        f               *clientcmd.Factory
}

func NewController(os *osclient.Client, kc *kclient.Client) *Controller {

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

func (c *Controller) Run(stopChan
What we've done here is change our **Run()** function to:

- [*41*] Accept our stop channel as a parameter
- [*43*] Spawn a Go routine that will [run continuously using Kubernetes' wait.Until() function](https://godoc.org/k8s.io/kubernetes/pkg/util/wait#Until)
- [*45*] Create a Watch Interface using the Kubernetes client to listen for pods in all namespaces
- [*56*] Listen for events over the Watch Interface's result channel inside a [select statement](https://tour.golang.org/concurrency/5).
- [*73-78*] Process the event for the object type expected (in this case "pod", but we could handle events for multiple types of objects) and print metadata about that event and the relevant resource object.

Deploy any simple app on your running and configured OpenShift cluster, then build and run your controller. You should see output similar to this:
ADDED pod ruby-hello-world-1-build in namespace test
ADDED pod docker-registry-1-deploy in namespace default
(Your output will obviously vary based on the app you use. I just used the [sample Ruby hello world app](https://github.com/openshift/ruby-hello-world).)

These events show up because, when starting up, a Watch Interface will receive ADDED events for all currently running pods. If I leave the controller running and, in another terminal, delete my ruby-hello-world pod, I see this output added:
MODIFIED pod ruby-hello-world-1-build in namespace test
MODIFIED pod ruby-hello-world-1-build in namespace test
DELETED pod ruby-hello-world-1-build in namespace test
So you can see how different interactions on your cluster can trigger different types of events.

*Note that the OpenShift 3.3 client package includes easy access to Watch Interfaces for several additional types, such as [Projects](https://godoc.org/github.com/openshift/origin/pkg/client#ProjectInterface).*

## Fun: Cumulative Runtimes

As an exercise, let's modify our controller to keep track of the cumulative runtime of all the pods for each namespace. Update your **pkg/controller/controller.go** file to change your **ProcessEvent()** function and add a new function, **TimeSince()**, like so:
func (c *Controller) ProcessEvent(event watch.Event, ok bool) {
        if !ok {
                fmt.Println("Error received from watch channel")
        }
        if event.Type == watch.Error {
                fmt.Println("Watch channel error")
        }

        var namespace string
        var runtime float64
        switch t := event.Object.(type) {
        case *kapi.Pod:
                podList, err := c.kubeClient.Pods(t.ObjectMeta.Namespace).List(kapi.ListOptions{})
                if err != nil {
                        fmt.Println(err)
                }
		for _, pod := range podList.Items {
                        runtime += c.TimeSince(pod.ObjectMeta.CreationTimestamp.String())
                }
                namespace = t.ObjectMeta.Namespace
        default:
                fmt.Printf("Unknown type\n")
        }
        fmt.Printf("Pods in namespace %v have been running for %v minutes.\n", namespace, runtime)
}

func (c *Controller) TimeSince(t string) float64 {
        startTime, err := time.Parse("2006-01-02 15:04:05 -0700 EDT", t)
        if err != nil {
                fmt.Println(err)
        }
        duration := time.Since(startTime)
        return duration.Minutes()
}

Now, whenever a new pod event is received (such as adding or deleting), we'll trigger a client call to gather a list of all the running pods in the relevant namespace. From the CreationTimestamp of each pod, we use the **time.Since()** method to calculate how long it's been running in minutes. From there it's just a matter of summing up all the runtimes we've calculated. When you run it, the output should be similar to this:
Pods in namespace default have been running for 1112.2476349382832 minutes.
Pods in namespace test have been running for 3.097702110216667 minutes.
Try scaling the pods in a project up or down, and see how it triggers a new calculation each time. This is a very simple example, but hopefully it's enough to get you on your way writing your own controllers for OpenShift!
