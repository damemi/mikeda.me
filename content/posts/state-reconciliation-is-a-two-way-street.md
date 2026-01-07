---
title: "State Reconciliation is a two-way street"
date: 2022-09-12T22:10:31
draft: false
slug: state-reconciliation-is-a-two-way-street
categories:
  - Uncategorized
wordpress_id: 251
---
*If you like this post, you can learn more about operators from my book,* [The Kubernetes Operator Framework Book.](https://www.amazon.com/Kubernetes-Operator-Framework-Book-management-dp-1803232854/dp/1803232854)

Recently my team and I were working on a feature for our operator when we came across a state-reconciliation bug that was potentially serious. What's worse, we almost didn't catch it. This was especially surprising as a diverse team of developers that were both new and experienced with Kubernetes. The tl;dr was that it is important for an operator to watch both its *inputs* and *outputs*.

## The bug we missed

For some background, the operand managed by our operator accepts a fairly complex configuration to set up. So, we were working on an operator that watched [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) (created by the user) and generated a ConfigMap based on those CRs. This ConfigMap is then used by the operand component as the source of its runtime settings, and the Custom Resources abstract the complexity of the underlying config from the user.

While making changes around the code that generated the ConfigMaps, we almost introduced a regression in which the operator would have stopped watching the generated ConfigMaps that it wrote. Thankfully, this was caught before the code merged. But it was only noticed by chance during code review, when it was something an automated pre-merge test should have caught.

## Why is this important?

From a happy-path perspective, the operator worked fine. The user would create their CR, the operator would notice it, and the ConfigMap would be generated. If the user made any changes to their CR, those would be picked up too.

But what if something happened to generated ConfigMap? Say an accidental kubectl delete on the ConfigMap shuts down the component. Or worse, an attacker inserts their own malicious settings directly into the ConfigMap. If the operator isn't keeping an eye on its output (just as it does with its input) then it can't recover from situations like this. In this case, it is effectively useless as a state reconciliation controller.

## State Reconciliation in Kubernetes

State reconciliation is not just an operator topic, it is one of the foundational concepts of Kubernetes. In [my book](https://www.amazon.com/Kubernetes-Operator-Framework-Book-management-dp-1803232854/dp/1803232854), I briefly discuss this idea in comparing operators to built-in Kubernetes controllers:

> These controllers work by monitoring the current state of the cluster and comparing it to the desired state. One example is a ReplicaSet with a specification to maintain three replicas of a Pod. Should one of the replicas fail, the ReplicaSet quickly identifies that there are now only two running replicas. It then creates a new Pod to bring stasis back to the cluster.

In this example, the ReplicaSet controller that's watching the whole cluster has an input and an output:

- **Input:** The spec.replicas field on the [ReplicaSet object](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/), where the user says how many replicas they want. This is the *desired state* of the cluster.
- **Output:** The Pods created by the ReplicaSet controller that are now running in the cluster. This represents the *actual state* of the cluster.

The reconciliation of [*desired state* vs *actual state*](https://downey.io/blog/desired-state-vs-actual-state-in-kubernetes) is a core function of Kubernetes. It's almost taken for granted that Kubernetes will always maintain the number running Pods in the cluster to match the number you've set in your ReplicaSet (or Deployment).

In terms of our operator, the same idea still applies:

- **Input / *desired state*:** Custom Resources created by the user.
- **Output / *actual state*:** The generated ConfigMap written by the operator.

In our case, the operator failing to reconcile on any changes to its written ConfigMap is no different than a ReplicaSet failing to create new Pods. Obviously, that would not be an acceptable regression in Kubernetes. So why was this almost overlooked in our operator?

## Operators can be confusing

Operators present a whole new field of possibilities in cloud architecture, with unfamiliar concepts and confusing terminology that may seem intimidating to new users and experienced Kubernetes developers alike. These new concepts can make it seem like you must forget everything you thought you knew to make room for something new. With a horizon this broad, it can be easy to disconnect from the familiar as the unknown begins to overwhelm.

But at their core, operators are just Kubernetes controllers with custom logic. This took me a long time to grasp as I fought my way through buzzword soup and exuberant documentation to understand this hot new tech. But ultimately, if you can understand how ReplicaSets, Deployments, or almost any other Kubernetes resource works, then you already have a good understanding of operator fundamentals.

Reducing the idea of operators down to a familiar concept helped me understand them much better. It helped me to have some background with [upstream Kubernetes development](https://github.com/kubernetes/kubernetes), but I believe that it can also work the other way around. Lowering the barrier to working with operators helps make many of the development practices used by core Kubernetes authors more approachable to new contributors. Because at the end of the day, operators are just custom extensions of Kubernetes.

I talk about this idea with more context in my book, *The Kubernetes Operator Framework Book*, because I hope that it will help readers to understand not only operator development, but Kubernetes development as a whole.
