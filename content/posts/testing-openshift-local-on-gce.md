---
title: "Testing Openshift Local (CRC) on a GCE VM"
date: 2026-01-06T15:27:16-05:00
slug: "testing-openshift-local-on-gce"
description: ""
draft: false
---

To test the [Odigos
Operator](https://odigos.io/blog/openshift-operator) we wanted to have
a realistic OpenShift environment, but even minimal clusters on GKE or
EKS were costing us thousands of dollars a month due to OpenShift's
high requirements. That's when I learned about [OpenShift
Local](https://developers.redhat.com/products/openshift-local) (CRC or
Code Ready Containers), which spins up a minimal development
environment for OpenShift.

However, I found that running it locally on my Mac was frustratingly
slow and required a lot of
[resources](https://crc.dev/docs/installing/#minimum-system-requirements)
(35GB of disk and 10GB of RAM!), so I looked into running it in a
cloud VM. For GCP, that meant figuring out a couple kinks to run it in
Compute Engine.

Side note, this made me sorely miss the simple days of `oc cluster up`.

# Setup

The [CRC docs](https://crc.dev/docs/getting-started/) have a lot of
information to download the binary. You'll need a Red Hat account with
a valid pull secret, which is easily provided along with the binary on
the [OpenShift Local download
page](https://console.redhat.com/openshift/create/local). I'll be
running on a CentOS VM (I had trouble figuring out how to SSH into a
GCE Fedora VM and gave up), so download the Linux x86 archive and save
your pull secret.

## GCE VM

For your VM, you need a machine with enough resources but also the
right machine type, as CRC requires [nested
virtualization](https://docs.cloud.google.com/compute/docs/instances/nested-virtualization/overview). This
means that you can't use an E2 or M* machine type.

I created a VM with the following specs:

* **N1 standard 8 (8 vCPU, 30GB memory)** - The CRC minimums are 4 cores and 10.5GB memory.
* **CentOS image**
* **1024GB disk**

When your VM is ready, update it to enable nested virtualization
[following the GCP
docs](https://docs.cloud.google.com/compute/docs/instances/nested-virtualization/enabling#enable_nested_virtualization_directly_on_an_existing_vm).

When the VM restarts, [confirm that nested virtualization is
enabled](https://docs.cloud.google.com/compute/docs/instances/nested-virtualization/enabling#confirm_that_nested_virtualization_is_enabled_on_the_vm).

Now, connect to your VM with the GCP in-browser SSH and upload the CRC
bundle and your pull secret file. Follow the [CRC installing
steps](https://crc.dev/docs/installing/#installing) to extract the
binary and add it to your PATH.

# Starting CRC

I followed the [CRC docs for the OpenShift
preset](https://crc.dev/docs/using/#openshift-preset) but found that
the defaults quickly had my test cluster hitting DiskPressure and CPU
quota issues. So before you run `crc start`, configure the environment
with the following settings:

```
crc config set cpus 8
crc config set disk-size 250
crc config set memory 15360
```

These should be enough but you can adjust as you find necessary. Now
run `crc start` and grab some coffee while things start up.

When your cluster is ready, CRC will print out the console and
kubeadmin login info. But how do we access the console from our local
machine? Using an SSH tunnel and some local DNS settings!

# Local setup

If you ever ran CRC locally, you may already have the right local DNS
in your `/etc/hosts` file. If not, check for this section and add it:

```
# Added by CRC
127.0.0.1        canary-openshift-ingress-canary.apps-crc.testing console-openshift-console.apps-crc.testing default-route-openshift-image-registry.apps-crc.testing downloads-openshift-console.apps-crc.testing api.crc.testing canary-openshift-ingress-canary.apps.crc.testing console-openshift-console.apps.crc.testing default-route-openshift-image-registry.apps.crc.testing downloads-openshift-console.apps.crc.testing
127.0.0.1        oauth-openshift.apps-crc.testing host.crc.testing oauth-openshift.apps.crc.testing
# End of CRC section
```

# Connecting

To set up the SSH tunnel to your VM, use the `gcloud ssh` command like so:

```
USER=<GCP VM LINUX USER>
GCE_VM=<VM NAME>
REGION=<REGION>
sudo gcloud compute ssh $USER@$GCE_VM \
  --zone $REGION -- \
  -L 443:console-openshift-console.apps-crc.testing:443 \
  -L 6443:api.crc.testing:6443 \
  -L 9443:oauth-openshift.apps-crc.testing:443
```

This sets up local SSH tunnels on ports `443`, `6443`, and `9443` to
the paths served by the OpenShift console.

Note that it is run with `sudo`, otherwise tunneling on port `443`
will be blocked on Mac.

If that succeeds, you should be SSH'd into your VM. Open your local
browser and go to the OpenShift console from the URL that CRC provided
(should be `https://console-openshift-console.apps-crc.testing`).

You can also now locally log in with the `oc` commands that CRC
provides and use `oc`/`kubectl` to interact with the cluster.
