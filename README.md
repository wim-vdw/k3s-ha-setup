# K3s High Availability (HA) Setup

## Introduction

This project involves deploying a Kubernetes cluster using `K3s` on a set of 5 virtual machines (VMs). The VMs are
running on Proxmox hosted on an old x86 PC with 16Gb of memory. The setup is designed to provide a lightweight,
efficient and scalable Kubernetes environment for testing and development.  
The cluster consists of 3 control plane nodes and 2 worker nodes, providing a solid foundation for deploying and
managing containerized applications. The use of `K3s` ensures minimal resource overhead, making it ideal for
environments
with limited hardware resources.

## VM overview

| VM Name       | MAC Address       | IP Address   | Memory (Gb) | Disk Size (Gb) | CPU Cores |
|---------------|-------------------|--------------|-------------|----------------|-----------|
| k3s-server-01 | BC:24:11:66:6F:07 | 172.16.12.22 | 8           | 32             | 2         |
| k3s-server-02 | BC:24:11:02:57:C8 | 172.16.12.23 | 8           | 32             | 2         |
| k3s-server-03 | BC:24:11:4F:A3:86 | 172.16.12.24 | 8           | 32             | 2         |
| k3s-worker-01 | BC:24:11:12:B1:D2 | 172.16.12.25 | 16          | 32             | 4         |
| k3s-worker-02 | BC:24:11:07:BA:1A | 172.16.12.26 | 16          | 32             | 4         |
| k3s-worker-03 | BC:24:11:07:BA:1A | 172.16.12.27 | 16          | 32             | 4         |

## Create VMs in Proxmox

[Proxmox Cloud-Init Support](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_cloud_init)  
[Ubuntu Cloud Images](https://cloud-images.ubuntu.com)

Create template based on cloud-init image:

```bash
#!/bin/bash

qm create 9000 --name ubuntu-cloud-init --memory 2048 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci
qm set 9000 --scsi0 data:0,import-from=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img
qm set 9000 --ide1 data:cloudinit
qm set 9000 --boot order=scsi0
qm set 9000 --serial0 socket --vga serial0
qm template 9000
```

Create VMs based on template and start them:

```bash
#!/bin/bash

qm clone 9000 220 --name k3s-server-01
qm resize 220 scsi0 32G
qm set 220 --net0 virtio,bridge=vmbr0,tag=12
qm set 220 --cpu host
qm set 220 --memory 8192
qm set 220 --cores 2
qm set 220 --ostype l26
qm set 220 --tags k3s
qm set 220 --localtime 1
qm set 220 --onboot 1
qm set 220 --ciuser jurre
qm set 220 --sshkey ~/.ssh/id_rsa.pub
qm set 220 --ciupgrade 1
qm set 220 --ipconfig0 ip=172.16.12.22/24,gw=172.16.12.1

qm clone 9000 221 --name k3s-server-02
qm resize 221 scsi0 32G
qm set 221 --net0 virtio,bridge=vmbr0,tag=12
qm set 221 --cpu host
qm set 221 --memory 8192
qm set 221 --cores 2
qm set 221 --ostype l26
qm set 221 --tags k3s
qm set 221 --localtime 1
qm set 221 --onboot 1
qm set 221 --ciuser jurre
qm set 221 --sshkey ~/.ssh/id_rsa.pub
qm set 221 --ciupgrade 1
qm set 221 --ipconfig0 ip=172.16.12.23/24,gw=172.16.12.1

qm clone 9000 222 --name k3s-server-03
qm resize 222 scsi0 32G
qm set 222 --net0 virtio,bridge=vmbr0,tag=12
qm set 222 --cpu host
qm set 222 --memory 8192
qm set 222 --cores 2
qm set 221 --ostype l26
qm set 222 --tags k3s
qm set 222 --localtime 1
qm set 222 --onboot 1
qm set 222 --ciuser jurre
qm set 222 --sshkey ~/.ssh/id_rsa.pub
qm set 222 --ciupgrade 1
qm set 222 --ipconfig0 ip=172.16.12.24/24,gw=172.16.12.1

qm clone 9000 223 --name k3s-worker-01
qm resize 223 scsi0 32G
qm set 223 --net0 virtio,bridge=vmbr0,tag=12
qm set 223 --cpu host
qm set 223 --memory 16384
qm set 223 --cores 4
qm set 223 --ostype l26
qm set 223 --tags k3s
qm set 223 --localtime 1
qm set 223 --onboot 1
qm set 223 --ciuser jurre
qm set 223 --sshkey ~/.ssh/id_rsa.pub
qm set 223 --ciupgrade 1
qm set 223 --ipconfig0 ip=172.16.12.25/24,gw=172.16.12.1

qm clone 9000 224 --name k3s-worker-02
qm resize 224 scsi0 32G
qm set 224 --net0 virtio,bridge=vmbr0,tag=12
qm set 224 --cpu host
qm set 224 --memory 16384
qm set 224 --cores 4
qm set 224 --ostype l26
qm set 224 --tags k3s
qm set 224 --localtime 1
qm set 224 --onboot 1
qm set 224 --ciuser jurre
qm set 224 --sshkey ~/.ssh/id_rsa.pub
qm set 224 --ciupgrade 1
qm set 224 --ipconfig0 ip=172.16.12.26/24,gw=172.16.12.1

qm clone 9000 225 --name k3s-worker-03
qm resize 225 scsi0 32G
qm set 225 --net0 virtio,bridge=vmbr0,tag=12
qm set 225 --cpu host
qm set 225 --memory 16384
qm set 225 --cores 4
qm set 225 --ostype l26
qm set 225 --tags k3s
qm set 225 --localtime 1
qm set 225 --onboot 1
qm set 225 --ciuser jurre
qm set 225 --sshkey ~/.ssh/id_rsa.pub
qm set 225 --ciupgrade 1
qm set 225 --ipconfig0 ip=172.16.12.27/24,gw=172.16.12.1

qm start 220
qm start 221
qm start 222
qm start 223
qm start 224
qm start 225
```

## Install first server (control plane node)

Use the `--cluster-init` flag to create the first server in the cluster and initialize the embedded `etcd` datastore for
high availability (HA).  
To install a specific version of `K3s`, set the `INSTALL_K3S_VERSION` environment variable before running the
installation
script.  
All `K3s` versions can be found here: [k3s release](https://github.com/k3s-io/k3s/releases)

```bash
# Use a specific version of K3s.
export INSTALL_K3S_VERSION=v1.31.5+k3s1

# Installs K3s, initializes the first server in HA mode and disables Traefik and the ServiceLB load balancer.
sudo curl -sfL https://get.k3s.io | sh -s - server --cluster-init --disable="traefik" --disable="servicelb"

# Displays the kubeconfig file for accessing the K3s cluster.
sudo cat /etc/rancher/k3s/k3s.yaml

# Displays the token used for joining nodes to the K3s cluster.
sudo cat /var/lib/rancher/k3s/server/token
```

## Install additional servers (control plane nodes)

```bash
export INSTALL_K3S_VERSION=v1.30.6+k3s1
export K3S_URL=https://172.16.12.22:6443
export K3S_TOKEN=<token from first server>

sudo curl -sfL https://get.k3s.io |sh -s - server --disable="traefik" --disable="servicelb"
```

## Install agents (worker nodes)

```bash
export INSTALL_K3S_VERSION=v1.31.5+k3s1
export K3S_URL=https://172.16.12.22:6443
export K3S_TOKEN=<token from first server>

sudo curl -sfL https://get.k3s.io | sh -s - agent
```

## Set up kubectl to use the K3s kubeconfig file

From one of the servers copy the `k3s.yaml` file to your local machine (example to $HOME/.kube/k3s.yaml).  
Edit the `k3s.yaml` file and correct the server address with the IP address of one of the servers (control planes).   
The `KUBECONFIG` environment variable can now be used to specify the `k3s.yaml` file and use it with `kubectl`.

```bash
export KUBECONFIG=$HOME/.kube/k3s.yaml

kubectl get nodes
NAME            STATUS   ROLES                       AGE    VERSION
k3s-server-01   Ready    control-plane,etcd,master   163m   v1.30.6+k3s1
k3s-server-02   Ready    control-plane,etcd,master   17m    v1.30.6+k3s1
k3s-server-03   Ready    control-plane,etcd,master   15m    v1.30.6+k3s1
k3s-worker-01   Ready    <none>                      108s   v1.30.6+k3s1
k3s-worker-02   Ready    <none>                      23s    v1.30.6+k3s1

kubectl get nodes -o wide
NAME            STATUS   ROLES                       AGE     VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k3s-server-01   Ready    control-plane,etcd,master   163m    v1.30.6+k3s1   192.168.1.201   <none>        Ubuntu 24.04.1 LTS   6.8.0-49-generic   containerd://1.7.22-k3s1
k3s-server-02   Ready    control-plane,etcd,master   17m     v1.30.6+k3s1   192.168.1.202   <none>        Ubuntu 24.04.1 LTS   6.8.0-49-generic   containerd://1.7.22-k3s1
k3s-server-03   Ready    control-plane,etcd,master   15m     v1.30.6+k3s1   192.168.1.203   <none>        Ubuntu 24.04.1 LTS   6.8.0-49-generic   containerd://1.7.22-k3s1
k3s-worker-01   Ready    <none>                      2m10s   v1.30.6+k3s1   192.168.1.211   <none>        Ubuntu 24.04.1 LTS   6.8.0-49-generic   containerd://1.7.22-k3s1
k3s-worker-02   Ready    <none>                      45s     v1.30.6+k3s1   192.168.1.212   <none>        Ubuntu 24.04.1 LTS   6.8.0-49-generic   containerd://1.7.22-k3s1
```

**Using the IP address of a single server does not ensure high availability for the Kubernetes API Server.**  
Implementing a highly available Kubernetes API Server setup is still on the to-do list.

## Tainting Control Plane Nodes

To prevent workloads from running on the control plane nodes, you can taint them.  
This ensures that only specific pods with the corresponding tolerations can be scheduled on these nodes.  
Run the following `kubectl` commands to taint your control plane nodes:

```
kubectl taint nodes k3s-server-01 node-role.kubernetes.io/control-plane:NoSchedule
kubectl taint nodes k3s-server-02 node-role.kubernetes.io/control-plane:NoSchedule
kubectl taint nodes k3s-server-03 node-role.kubernetes.io/control-plane:NoSchedule
```

This will prevent workloads from being scheduled on the `k3s-server-*` nodes unless they have a matching toleration.

## Bootstrapping with ArgoCD and Deploying Workloads

Now that your cluster is set up, you can deploy workloads on it. However, before doing so it is recommended to bootstrap
the cluster with ArgoCD for easier and more efficient management of your Kubernetes resources.  
ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. By integrating it into your cluster, you can
manage deployments from Git repositories and streamline your deployment process.  
After bootstrapping with ArgoCD, you can deploy your workloads seamlessly, with ArgoCD taking care of the application
management and lifecycle.

Example: [https://github.com/wim-vdw/kubernetes-tests](https://github.com/wim-vdw/kubernetes-tests)

## Example Workload after Bootstrapping with ArgoCD

After bootstrapping the cluster with ArgoCD and deploying core and test applications the workload is distributed across
the nodes.

Core applications:

* `Ingress NGINX` controller for Kubernetes
* `MetalLB` load-balancer implementation for bare metal Kubernetes clusters
* `Reloader` controller to watch changes in Kubernetes ConfigMaps and Secrets

Control Plane Taints:

The control plane nodes are tainted to prevent most workloads from running on them, ensuring they remain dedicated to
cluster management.  
All workloads are primarily scheduled on worker nodes.

```bash
kubectl get pods -A -o wide
NAMESPACE        NAME                                                READY   STATUS      RESTARTS        AGE   IP              NODE            NOMINATED NODE   READINESS GATES
argocd           argocd-application-controller-0                     1/1     Running     0               32m   10.42.3.8       k3s-worker-01   <none>           <none>
argocd           argocd-applicationset-controller-6899bbc884-hnsdv   1/1     Running     0               32m   10.42.3.5       k3s-worker-01   <none>           <none>
argocd           argocd-dex-server-bcf886644-j7f95                   1/1     Running     0               32m   10.42.4.7       k3s-worker-02   <none>           <none>
argocd           argocd-notifications-controller-64fb4bc4f9-mt7bh    1/1     Running     0               32m   10.42.3.6       k3s-worker-01   <none>           <none>
argocd           argocd-redis-6cbf9bf4c5-9x84v                       1/1     Running     0               32m   10.42.4.8       k3s-worker-02   <none>           <none>
argocd           argocd-repo-server-b5d548b6c-42l7r                  1/1     Running     0               32m   10.42.3.7       k3s-worker-01   <none>           <none>
argocd           argocd-server-65bc9998bd-6q67m                      1/1     Running     0               32m   10.42.4.9       k3s-worker-02   <none>           <none>
ingress-nginx    ingress-nginx-admission-create-zx7hl                0/1     Completed   2               12m   10.42.3.10      k3s-worker-01   <none>           <none>
ingress-nginx    ingress-nginx-admission-patch-rwb9h                 0/1     Completed   3               12m   10.42.3.9       k3s-worker-01   <none>           <none>
ingress-nginx    ingress-nginx-controller-85bc8b845b-4xg48           1/1     Running     0               12m   10.42.3.11      k3s-worker-01   <none>           <none>
kube-system      coredns-7b98449c4-d7khv                             1/1     Running     2 (62m ago)     74m   10.42.4.5       k3s-worker-02   <none>           <none>
kube-system      local-path-provisioner-595dcfc56f-glm8h             1/1     Running     2 (62m ago)     76m   10.42.3.4       k3s-worker-01   <none>           <none>
kube-system      metrics-server-cdcc87586-47pfb                      1/1     Running     1 (62m ago)     65m   10.42.4.6       k3s-worker-02   <none>           <none>
metallb-system   controller-6dd967fdc7-kgphg                         1/1     Running     0               11m   10.42.3.12      k3s-worker-01   <none>           <none>
metallb-system   speaker-5plfh                                       1/1     Running     0               11m   192.168.1.211   k3s-worker-01   <none>           <none>
metallb-system   speaker-f77nr                                       1/1     Running     0               11m   192.168.1.201   k3s-server-01   <none>           <none>
metallb-system   speaker-m6cmd                                       1/1     Running     1 (8m57s ago)   11m   192.168.1.203   k3s-server-03   <none>           <none>
metallb-system   speaker-nw8kq                                       1/1     Running     2 (8m57s ago)   11m   192.168.1.212   k3s-worker-02   <none>           <none>
metallb-system   speaker-rk25l                                       1/1     Running     0               11m   192.168.1.202   k3s-server-02   <none>           <none>
reloader         reloader-reloader-795f5cbd78-zpnjl                  1/1     Running     0               12m   10.42.4.10      k3s-worker-02   <none>           <none>
test04           blue-dep-7457ff8c59-6w64p                           1/1     Running     0               45s   10.42.3.13      k3s-worker-01   <none>           <none>
test04           blue-dep-7457ff8c59-qfwfj                           1/1     Running     0               45s   10.42.4.13      k3s-worker-02   <none>           <none>
test04           blue-dep-7457ff8c59-rdx74                           1/1     Running     0               45s   10.42.4.11      k3s-worker-02   <none>           <none>
test04           green-dep-5d77bd8d4d-5gbxv                          1/1     Running     0               45s   10.42.4.12      k3s-worker-02   <none>           <none>
test04           green-dep-5d77bd8d4d-6srq9                          1/1     Running     0               45s   10.42.3.15      k3s-worker-01   <none>           <none>
test04           green-dep-5d77bd8d4d-hvsfr                          1/1     Running     0               45s   10.42.3.14      k3s-worker-01   <none>           <none>
```

## Task list

- [X] Add workload example after bootstrapping ArgoCD and deploying core and test applications.
- [X] Add VM creation with Proxmox and cloud-init.
- [ ] Analyse performance/stability of etcd datastore.
- [ ] Make Kubernetes API Server high available with `HAProxy` and `KeepAlived`.
