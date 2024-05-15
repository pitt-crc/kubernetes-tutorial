# Kubernetes DevOps Example

A tutorial for configuring GitOps driven deployments to a development Kubernetes cluster.

## Table of Contents

- [Overview](#overview)
- [System Setup](#system-setup)
  - [Installing Docker](#installing-docker)
  - [Installing Kubectl](#installing-kubectl)
  - [Installing Helm](#installing-helm)
  - [Installing Minikube](#installing-minikube)
- [Setting Up the Cluster](#setting-up-the-cluster)
  - [Configuring Kubernetes](#configuring-kubernetes)
  - [Deploying Argo CD](#deploying-argo-cd)

## Overview

This tutorial walks through the creation of a local, virtualized Kubernetes cluster using Minikube.
Instructions are included for getting a development ArgoCD installation running on the cluster.

## System Setup

The following section provides instructions for setting up a development Kubernetes environment and are not suitable for building a production ready system. 
Command line examples are written for an x86 system running [Rocky Linux 9](https://rockylinux.org/). 
Links are provided to alternative instructions for other system configurations.

For this tutorial, you will need the following software stack available on your machine:

- [Docker](https://www.docker.com/)
- [Kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [Helm](https://helm.sh)
- [Minikube](https://minikube.sigs.k8s.io/docs/)

### Installing Docker

Docker is used to package and deploy software as portable, self-sufficient containers. Rocky Linux maintains [official instructions](https://docs.rockylinux.org/gemstones/docker/) for installing docker within Rocky OS.
For installation instructions on other platforms, see the official [Docker documentation](https://docs.docker.com/engine/install/).

**Note:** The open source alternative Podman comes pre-installed on many systems, but has compatibility issues with the tools we will be using later on ([for example](https://github.com/kubernetes/minikube/issues/9120)). Podman is not recommended when following this tutorial.

Installing docker is accomplished using a few `dnf` commands:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl --now enable docker
```

In order for `docker` to execute with the appropriate permissions, you will need to add your user to the `docker` group: 

```bash
sudo usermod -aG docker $USER && newgrp docker
```

### Installing Kubectl

Kubectl is a command-line tool used to manage Kubernetes clusters.
Following the official [Kubernetes documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/), the latest version can be installed as follows: 

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Installing Helm

Helm is a package manager for Kubernetes. The Helm maintainers provide a useful setup script for automatically installing the `helm` utility:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Installing Minikube

Minikube is a lightweight Kubernetes distribution that runs locally on a single machine.
See the [official instructions](https://minikube.sigs.k8s.io/docs/start/) for architecture specific installation instructions. On x86 platforms, the latest version can be installed as follows: 

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -Uvh minikube-latest.x86_64.rpm
```

## Setting Up the Cluster

### Configuring Kubernetes

For this tutorial we will use Minikube to create a single Kubernetes cluster called `tutorial`.
Use the ``start`` demonstrated below to configure and launch a new cluster.
Note the usage of `docker` as the driver.
This instructs `minikube` to create Kubernetes nodes as Docker containers instead of running nodes on extra physical/virtual machines.

```bash
minikube start -p tutorial --driver=docker --nodes=4
minikube profile tutorial  # set the tutorial cluster as the current default cluster
```

When using Docker as a driver, your OS may run into `inotify` limits.
This typically results in an error similar to `Failed to create control group inotify object: Too many open files` (the exact wording may vary).
The error can be resolved by increasing the inotify watch limits:

```bash
sysctl fs.inotify.max_user_watches=1048576
sysctl fs.inotify.max_user_instances=8192
```

The above commands do not persist between reboots.
For the changes to persist, you'll need to add the following lines to the appropriate configuration file in `/etc/sysctl.d/` (`99-inotify.conf` is a common choice):

```conf
fs.inotify.max_user_watches=1048576
fs.inotify.max_user_instances=8192
```

After starting the Minikube cluster, verify your setup using the `profile` command.

```bash 
minikube profile list
```

The returned output should look similar to the following:

```
|----------|-----------|---------|--------------|------|---------|---------|-------|--------|
| Profile  | VM Driver | Runtime |      IP      | Port | Version | Status  | Nodes | Active |
|----------|-----------|---------|--------------|------|---------|---------|-------|--------|
| tutorial | docker    | docker  | 192.168.58.2 | 8443 | v1.26.3 | Unknown |     4 |     *  |
|----------|-----------|---------|--------------|------|---------|---------|-------|--------|
```

Like most kubernetes installations, ingress is not enabled by default.
You will need to enable the associated addon for each cluster:

```bash
minikube addons enable ingress
```
### Deploying Argo CD

We will use Argo CD as our GitOps / continuous deployment tool.
Following the [official docs](https://argo-cd.readthedocs.io/en/stable/getting_started/), we install the official manifest.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Argo automatically creates an `admin` user with a random password.
Use the following command to fetch the password:

```bash
echo $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
```

To access the Argo web interface, forward the port from the ArgoCD server to the local machine.

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

The webpage is now availible at https://localhost:8080/

