# Kubernetes DevOps Example

This repository provides a getting started tutorial for setting up continuous deployment workflows with ArgoCD.

## Table of Contents

- [System Setup](#system-setup)
  - [Installing Docker](#installing-docker)
  - [Installing Kubectl](#installing-kubectl)
  - [Installing Minikube](#installing-minikube)
  - [System Checks](#system-checks)
- [Setting Up Kubernetes](#setting-up-kubernetes)
  - [Cluster Deployments](#cluster-deployments)
  - [Deploying Portainer](#deploying-portainer)   


## System Setup

The following section provides instructions for setting up a development Kubernetes environment. 
These instructions are not suitable for building a production ready system. 
Examples are written for an x86-64 system running [Rocky Linux](https://rockylinux.org/). 
Links are provided to the alternative installation instructions for other system configurations.

### Installing Docker

Rocky linux maintains [official instructions](https://docs.rockylinux.org/gemstones/docker/) for installing docker.
The open source alternative Podman does come pre-installed on Rocky, but has some compatibility issues with the tools we will be using later on ([for example](https://github.com/kubernetes/minikube/issues/9120)). 

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl --now enable docker
```

In order for your `docker` commands to execute with appropriate permissions, you will need to add your user to the `docker` group: 

```bash
sudo usermod -aG docker $USER && newgrp docker
```

### Installing Kubectl

Some linux distributions come with Kubectl pre installed.
The following command can be used to check if it is already installed:

```bash
kubectl version --client
```

Following the official [Kubernetes documentation], the latest version can be installed as follows: 

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Installing Helm

The helm maintainers provide a useful setup script for automatically installing the `helm` utility:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Installing Minikube

Minikube is a lightweight Kubernetes distribution that runs locally on a single machine.
See the [official instructions](https://minikube.sigs.k8s.io/docs/start/) for alternative architectures.

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -Uvh minikube-latest.x86_64.rpm
```


Use the ``start`` command to launch the cluster.
In this example we use `docker` as the driver.
This instructs `minikube` to create Kubernetes nodes as Docker containers.
See `minikube start --help` for other options.

```bash
minikube start --driver=docker
minikube status
```

Ingress is not enabled by default:

```bash
minikube addons enable ingress
```

The running Minikube instance (including all pods running on the Kubernetes server) can be stopped using the ``stop`` command.
You will need to keep the service running for the rest of this tutorial, but the command is important to know.

```bash
minikube stop
```

### System Checks

You can check your Kubernetes cluster is ready to go using the `kubectl` utility. 

```bash
kubectl get nodes
```

You should expect to receive output siomilar to the following:

```
NAME       STATUS   ROLES           AGE     VERSION
minikube   Ready    control-plane   6m23s   v1.26.3
```

## Setting Up Kubernetes

### Cluster Deployments

For this tutorial we will spin up multiple kubernetes clusters.
The `operations` cluster will be used to host administration and monitoring tools.
The `development` and `production` environments will be used in our CI/CD examples later on.

```bash
minikube start -p operations --driver=docker --nodes=2
minikube start -p development --driver=docker --nodes=2
minikube start -p production --driver=docker --nodes=2
```

You can verify the setup executed correctly using the `profile` command:

```bash 
minikube profile list
```

The returned output should look similar to the following:

```
|-------------|-----------|---------|--------------|------|---------|---------|-------|--------|
|   Profile   | VM Driver | Runtime |      IP      | Port | Version | Status  | Nodes | Active |
|-------------|-----------|---------|--------------|------|---------|---------|-------|--------|
| development | docker    | docker  | 192.168.58.2 | 8443 | v1.26.3 | Unknown |     2 |        |
| operations  | docker    | docker  | 192.168.49.2 | 8443 | v1.26.3 | Unknown |     2 |        |
| production  | docker    | docker  | 192.168.67.2 | 8443 | v1.26.3 | Unknown |     2 |        |
|-------------|-----------|---------|--------------|------|---------|---------|-------|--------|
```

The `minikube` commandline utility will only administrate a single cluster at a time.
To switch change the default cluster, use the `profile` command.

```bash
minikube profile [CLUSTER]
```

It is important to note the above command will modify the default cluster used by the `minikube` **and** `kubectl` utilities.
The `kubectl` has equivilent commands for changing the default cluster, but they will not effect the `minikube` utility.
To avoid confusing situations where the two utilities are referencing different default clusters, it is recomended default clusters are set using `minikube`.

The equivilent `kubectl` commands are provided below for reference.

```
kubectl config get-contexts 
kubectl config use-context [CLUSTER]
```

### Deploying Portainer 

We will use portainer as out continuous deployment tool.
A deployment manifest is provided in this repository.
In the example below we deploy the application into a namespace called `portainer`.

```
minikube profile operations
kubectl apply -n portainer -f https://raw.githubusercontent.com/djperrefort/cluster/main/portainer/portainer.yml
```

Portainer with/wiothout SSL on ports 30779/30777. The command below will fetch the appropraite IP address.

```
kubectl get nodes -o wide
```

minikube profile development
kubectl apply -f https://downloads.portainer.io/ee2-18/portainer-agent-k8s-lb.yaml


minikube profile production
kubectl apply -f https://downloads.portainer.io/ee2-18/portainer-agent-k8s-lb.yaml

### Deploying Argo CD


```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

