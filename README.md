# Kubernetes DevOps Example

This repository provides a getting started tutorial for setting up continuous deployment workflows with ArgoCD.

## Table of Contents

- [System Setup](#system-setup)
  - [Installing Docker](#installing-docker)
  - [Installing Kubectl](#installing-kubectl)
  - [Installing Helm](#installing-helm)
  - [Installing Minikube](#installing-minikube)
  - [System Checks](#system-checks)
- [Setting Up Kubernetes](#setting-up-kubernetes)
  - [Configuring Kubernetes](#configuring-kubernetes)
  - [Notes on Minikube](#notes-on-minikube)
- [CI/CD Deployments](#cicd-deployments)
  - [Deploying Portainer](#deploying-portainer)
  - [Deploying Argo CD](#deploying-argo-cd)
  - [Cluster Monitoring (Prometheus and Grafana)](#cluster-monitoring-prometheus-and-grafana)

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

## Setting Up Kubernetes

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

### Notes on Minikube

You can change the default cluster being administrated by `minikube` using the `profile` command:

```bash
minikube profile [CLUSTER]
```

It is important to note the above command will modify the default cluster used by the `minikube` **and** `kubectl` utilities.
The `kubectl` utility has equivalent commands for changing the default cluster, but they will **not** effect the `minikube` utility.
To avoid confusing situations where the two utilities are referencing different default clusters, it is recommended to switch between clusters using `minikube`.

## CI/CD Deployments

### Deploying Portainer 

We will use portainer as out continuous deployment tool.
A deployment manifest is provided in this repository.
In the example below we deploy the application into a namespace called `portainer`.

```bash
minikube profile tutorial
kubectl apply -n portainer -f https://raw.githubusercontent.com/djperrefort/cluster/main/portainer/portainer.yml
```

Portainer with/wiothout SSL on ports 30779/30777. The command below will fetch the appropraite IP address.

```bash
kubectl get service -n portainer
```

### Deploying Argo CD

To install the CLI

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

ArgoCD is installed using the official installation manifest.
In order to access the application GUI, the `argocd-server` service is to updated to a `NodePort` type.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

The cluster IP can be found using the `manifest profile list` command.
The port number for ArgoCD is found using `kubectl`:

```
kubectl get svc -n argocd

NAME                                      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP      10.104.235.186   <none>        7000/TCP,8080/TCP            3m52s
argocd-dex-server                         ClusterIP      10.103.95.187    <none>        5556/TCP,5557/TCP,5558/TCP   3m52s
argocd-metrics                            ClusterIP      10.109.192.232   <none>        8082/TCP                     3m52s
argocd-notifications-controller-metrics   ClusterIP      10.103.20.126    <none>        9001/TCP                     3m52s
argocd-redis                              ClusterIP      10.100.137.60    <none>        6379/TCP                     3m51s
argocd-repo-server                        ClusterIP      10.99.41.34      <none>        8081/TCP,8084/TCP            3m51s
argocd-server                             LoadBalancer   10.101.110.199   <pending>     80:30103/TCP,443:32420/TCP   3m51s
argocd-server-metrics                     ClusterIP      10.103.145.178   <none>        8083/TCP                     3m51s
```

In the example above, the port number is 30103.

Argo automatically creates an `admin` user with a random password.
Use the following command to fetch the password:

```bash
echo $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
```

### Cluster Monitoring (Prometheus and Grafana)

We will use The Prometheus and Grafana utilities to monitor/visualize the status of each cluster.
Start by adding the appropriate repositories to Helm:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Prometheus needs to be installed on each cluster you want to collect metrics for.
In the example below, we install Prometheus on all of the clusters we created earlier:

```bash
for CLUSTER in operations development production; do
  minikube profile $CLUSTER
  helm install prometheus-operator prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
done
```

To visualize the collected metrics, we install grafana on the `operations` cluster.
By default, Grafana does not persist it's data to disk.
This means Grafana's data is lost each time the Grafana pod is terminated.
Grafana can be configured to use a variety of storage types (see [the docs](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/) for more information).
In this case we set up local storage using a Docker volume:

```bash
minikube profile operations
helm install grafana grafana/grafana \
  --namespace grafana --create-namespace \
  --set persistence.enabled=true \
  --set persistence.storageClassName="" \
  --set persistence.size=10Gi \
  --set persistence.accessModes={ReadWriteOnce}
```

As part of the installation process, Grafana will automatically create an `admin` user with a random password.
Use the `kubectl` command to fetch the generated password and then decode it into plane text:

```bash
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

To access Grafana, expose the node the service on port 3000:

```bash
export POD_NAME=$(kubectl get pods --namespace grafana -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace grafana port-forward $POD_NAME 3000
```

After logging in to Grafana, the service can be configured in the standard fashion.
URLS for each Prometheus installation follow the following general format:

```
http://<prometheus-service-name>.<namespace>.svc.cluster.local:<prometheus-port>
```

For the operations cluster this should be:

```
http://prometheus-operated.monitoring.svc.cluster.local:9090
```

