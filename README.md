# CD DevOps Example

This repository provides a getting started tutorial for setting up continuous deployment workflows with ArgoCD.

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
