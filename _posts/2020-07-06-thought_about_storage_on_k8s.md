---
layout: post
title: "Thought about Storage on Kubernetes"
date:   2020-07-06 16:57:28 +09:00
---

Kubernetes is revolutionizing the way applications are being developed and deployed.
Now, developers can focus on implementing the application itself without worrying about
the underlying infrastructure and some of distributed system concepts.

However, Kubernetes doesn't support storing state, even though most of applications are stateful.

On Kubernetes, containers can being created and destroyed. They are dynamic.
But, persistent storages cannot be dynamic as normal Pods.

Many SW teams have been using distributed storage solutions provided by cloud providers, such as
AWS or GCP. It makes developers deploy their containers to another cloud provider or any other environments.

I was curious which the cloud-native storage solutions have being discussed.


### Native Kubernetes & Storage

On Kubernetes, we can use PV and PVC. In static provisioning, admin should define PVCs in advance, 
so that storages can be mounted to a particular node before executing Pods.
It doesn't meet the philosophy of Kubernetes that allocates resources (CPU/Mem) dynamically.

In dynamic provisioning by Storage Class, we can create multiple profiles of storage, just like templates.
When a developer makes a PVC, one of these templates is created at the time of the request, and attached to the Pod.
But, honestly, I don't understand yet how it works.


### CSI (Container Storage Interface)

CSI was created by CNCF Storage Working Group. It defines a standard container storage interface which can enable
storage drivers to work on any container orchestrator.

CSI spec have already been included into Kubernetes. There are already many driver plugins.
Developers can access storage exposed by CSI-compatible volume driver.

With CSI, storage can be treated as another workload to be containerized and deployed on a Kubernetes cluster.


### Open-source Projects

Ceph is one of popular distributed storages. Ceph has been adapted into the cloud-native environment.
There are many ways you can deploy a Ceph cluster, such as with Ansible.
Using CSI and PVCs, we can deploy a Ceph cluster and have an interface into it from Kubernetes cluster.

Also, Rook is an open-source project that converge Kubernetes and Ceph.
Rook is a cloud-native orchestrator which extends Kubernetes.
It allows putting Ceph into containers, and provides cluster management logic for running Ceph on Kubernetes.
It automates deployment, bootstrapping, configuration, scaling and rebalancing.

Rook doesn't have its own persistent state. It's truly built according to the principle of Kubernetes.


### Next

Let's see how Rook deals with Ceph on Kubernetes, and how it will deal with other distributed storages.

As a developer of distributed storages, is Rook the best option for deploying storages on Kubernetes?
