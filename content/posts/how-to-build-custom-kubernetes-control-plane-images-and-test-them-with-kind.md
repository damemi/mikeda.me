---
title: "How to Build Custom Kubernetes Control Plane Images (and Test Them With Kind)"
date: 2020-11-03
slug: "how-to-build-custom-kubernetes-control-plane-images-and-test-them-with-kind"
description: ""
draft: false
---

If you are working on changes to a core Kubernetes component such as
the scheduler, controller-manager, or apiserver, you may be confused
about how to actually build and deploy your changes in a local
environment to test them. This post will go over the method I use to
build and test control plane component changes. There are probably
more efficient ways to do it, but this works for me and gives me some
flexibility over each step which I will explain.

I am usually developing custom schedulers, so that will be the focus
for these steps (though as you will see, it pretty much applies to any
kube control-plane pod). So this post may be particularly useful if
you are also working on a custom scheduler, or plugins for the
scheduler, or are simply trying to debug the default kube-scheduler.

# Building control plane component images

The first step (after making any code changes) is to run `make
quick-release-images` from the root of the k8s.io/kubernetes
repository.

This will build release images for all the core components:

```
~/go/src/k8s.io/kubernetes$ make quick-release-images 
+++ [1103 14:32:11] Verifying Prerequisites....
+++ [1103 14:32:11] Using Docker for MacOS
+++ [1103 14:32:12] Building Docker image kube-build:build-841902342a-5-v1.15.2-1
+++ [1103 14:32:18] Syncing sources to container
+++ [1103 14:32:21] Running build command...
+++ [1103 14:33:25] Building go targets for linux/amd64:
    cmd/kube-apiserver
    cmd/kube-controller-manager
    cmd/kube-scheduler
    cmd/kube-proxy
    vendor/github.com/onsi/ginkgo/ginkgo
    test/e2e/e2e.test
    cluster/images/conformance/go-runner
    cmd/kubectl
+++ [1103 14:34:31] Syncing out of container
+++ [1103 14:34:40] Building images: linux-amd64
+++ [1103 14:34:40] Starting docker build for image: kube-apiserver-amd64
+++ [1103 14:34:40] Starting docker build for image: kube-controller-manager-amd64
+++ [1103 14:34:40] Starting docker build for image: kube-scheduler-amd64
+++ [1103 14:34:40] Starting docker build for image: kube-proxy-amd64
+++ [1103 14:34:40] Building conformance image for arch: amd64
+++ [1103 14:35:05] Deleting docker image k8s.gcr.io/kube-scheduler-amd64:v1.20.0-beta.1.5_c82d5ee0486191-dirty
+++ [1103 14:35:08] Deleting docker image k8s.gcr.io/kube-proxy-amd64:v1.20.0-beta.1.5_c82d5ee0486191-dirty
+++ [1103 14:35:15] Deleting docker image k8s.gcr.io/kube-controller-manager-amd64:v1.20.0-beta.1.5_c82d5ee0486191-dirty
+++ [1103 14:35:15] Deleting docker image k8s.gcr.io/kube-apiserver-amd64:v1.20.0-beta.1.5_c82d5ee0486191-dirty
+++ [1103 14:35:24] Deleting conformance image k8s.gcr.io/conformance-amd64:v1.20.0-beta.1.5_c82d5ee0486191-dirty
+++ [1103 14:35:24] Docker builds done
```

And it places them as individual .tar files in `./_output/release-images/amd64`:

```
$ ls _output/release-images/amd64/
total 817M
drwxr-xr-x 7 mdame staff  224 Nov  3 14:35 .
drwxr-xr-x 3 mdame staff   96 Nov  3 14:34 ..
-rw-r--r-- 1 mdame staff 288M Nov  3 14:35 conformance-amd64.tar
-rw------- 2 mdame staff 159M Nov  3 14:35 kube-apiserver.tar
-rw------- 2 mdame staff 150M Nov  3 14:35 kube-controller-manager.tar
-rw------- 2 mdame staff 130M Nov  3 14:35 kube-proxy.tar
-rw------- 2 mdame staff  63M Nov  3 14:35 kube-scheduler.tar
```

These can be loaded into docker, tagged, and pushed to your registry
of choice:

```
$ docker load -i _output/release-images/amd64/kube-scheduler.tar 
46d8985b3ff2: Loading layer [==================================================>]  60.37MB/60.37MB
Loaded image: k8s.gcr.io/kube-scheduler-amd64:v1.20.0-beta.1.5_c82d5ee0486191-dirty
$ docker tag k8s.gcr.io/kube-scheduler-amd64:v1.20.0-beta.1.5_c82d5ee0486191-dirty docker.io/mdame/kube-scheduler:test
$ docker push docker.io/mdame/kube-scheduler:test
The push refers to repository [docker.io/mdame/kube-scheduler]
46d8985b3ff2: Pushed 
c12e92a17b61: Layer already exists 
79d541cda6cb: Layer already exists 
printconfig: digest: sha256:c093dad1f4a60ce1830d91ced3a350982b921183c1b4cd9bfaae6f70c67af7ee size: 949
```

# Deploying your custom image into a Kind cluster

(Note: Kind actually provides ways for you to build and load code
changes into a cluster, see
https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster
and
https://kind.sigs.k8s.io/docs/user/quick-start/#building-images. So
this may be overkill, but allows you to make changes to an
already-running cluster or with images you’ve already built).

Start up a stock Kind cluster and get the current static pod manifest
for the kube-scheduler (use `docker ps` to get the running cluster’s
container, then the static pod manifests are located at
`/etc/kubernetes/manifests`):

```
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.19.1) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:
kubectl cluster-info --context kind-kind
Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                       NAMES
b4f712eff358        kindest/node:v1.19.1   "/usr/local/bin/entr…"   8 minutes ago       Up 8 minutes        127.0.0.1:58206->6443/tcp   kind-control-plane
$ docker exec -it b4f712eff358 cat /etc/kubernetes/manifests/kube-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --port=0
    image: k8s.gcr.io/kube-scheduler:v1.19.1
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
  hostNetwork: true
  priorityClassName: system-node-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
status: {}
```

Copy this and edit it to refer to my custom scheduler image (truncated here):

```
$ cat kube-scheduler.yaml 
apiVersion: v1
kind: Pod
...
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --port=0
    image: docker.io/mdame/kube-scheduler:test
    imagePullPolicy: Always
...
```

Copy this manifest into the Kind cluster, overwriting the existing manifest:

```
$ docker cp kube-scheduler.yaml <cluster-container-ID>:/etc/kubernetes/manifests/kube-scheduler.yaml
```

And now the running pod should refer to your image:

```
$ kubectl get -o yaml pod/kube-scheduler-kind-control-plane -n kube-system
apiVersion: v1
kind: Pod
metadata:
...
  name: kube-scheduler-kind-control-plane
  namespace: kube-system
...
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --port=0
    image: docker.io/mdame/kube-scheduler:test
    imagePullPolicy: Always
```

# Deploying changes to the image

If you need to deploy changes to the running pod, repeat the `make
quick-release-images` and `docker load/tag/push` steps from
above. Your new image won’t be pulled just by deleting the pod even
with `imagePullPolicy: Always` set, because these are static pods
mirroring a running container. So, to pull a new image you need to
exec back into the cluster and stop the running container:

```
$ docker exec -it <container-ID> crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID
cfcb6d257aa59       0c3dd0a038e11       47 minutes ago      Running             kube-scheduler            7                   83163425eb615
... ... ...
$ docker exec -it <container-ID> crictl stop cfcb6d257aa59
cfcb6d257aa59
```
