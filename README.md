# K3s High Availability (HA) Setup

## Introduction

This project involves deploying a Kubernetes cluster using `K3s` on a set of 7 virtual machines (VMs) running on
Proxmox Virtual Environment.  
The setup is designed to provide a lightweight, efficient and scalable Kubernetes environment for testing and
development.  
The cluster consists of 3 control plane nodes, 2 worker nodes, and 2 additional VMs for `HAProxy` load balancer and
`Keepalived`, providing a solid foundation for deploying and managing containerized applications.

Ensure high availability (HA) for the Kubernetes API server by:

* **HAProxy** Load balances traffic across multiple `K3s` server nodes, ensuring even distribution and redundancy.
* **Keepalived** Provides a virtual IP for `HAProxy`, enabling automatic failover between HAProxy instances to ensure
  high availability.

If a `K3s` server node goes down, `HAProxy` routes traffic to the remaining healthy servers.  
If an `HAProxy` instance fails, `Keepalived` shifts the virtual IP to another `HAProxy` node, maintaining API
availability.

References:

* [K3s - High Availability Embedded etcd](https://docs.k3s.io/datastore/ha-embedded)
* [K3s - Cluster Load Balancer](https://docs.k3s.io/datastore/cluster-loadbalancer)

## VM overview + IP addresses

| VM Name       | MAC Address       | IP Address    | Memory (Gb) | Disk Size (Gb) | CPU Cores |
|---------------|-------------------|---------------|-------------|----------------|-----------|
| k3s-server-01 | BC:24:11:66:6F:07 | 192.168.1.201 | 4           | 32             | 2         |
| k3s-server-02 | BC:24:11:02:57:C8 | 192.168.1.202 | 4           | 32             | 2         |
| k3s-server-03 | BC:24:11:4F:A3:86 | 192.168.1.203 | 4           | 32             | 2         |
| k3s-worker-01 | BC:24:11:12:B1:D2 | 192.168.1.211 | 8           | 32             | 4         |
| k3s-worker-02 | BC:24:11:07:BA:1A | 192.168.1.212 | 8           | 32             | 4         |
| k3s-lb-01     | BC:24:11:EA:6D:1F | 192.168.1.221 | 2           | 32             | 1         |
| k3s-lb-02     | BC:24:11:6B:DC:F9 | 192.168.1.222 | 2           | 32             | 1         |

Keepalived virtual IP address: **192.168.1.220**

## Create VMs in Proxmox

[Proxmox Cloud-Init Support](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_cloud_init)  
[Ubuntu Cloud Images](https://cloud-images.ubuntu.com)

Create template based on cloud-init image:

```bash
#!/bin/bash

qm create 9000 --name ubuntu-cloud-init --memory 2048 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci
qm set 9000 --scsi0 local-lvm:0,import-from=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img
qm set 9000 --ide1 local-lvm:cloudinit
qm set 9000 --boot order=scsi0
qm set 9000 --serial0 socket --vga serial0
qm template 9000
```

Create VMs based on template and start them:

```bash
#!/bin/bash

qm clone 9000 100 --name k3s-server-01
qm resize 100 scsi0 32G
qm set 100 --net0 virtio,bridge=vmbr0,macaddr=BC:24:11:66:6F:07
qm set 100 --cpu host
qm set 100 --memory 4096
qm set 100 --cores 2
qm set 100 --ciuser wim
qm set 100 --sshkey id_rsa.pub
qm set 100 --ciupgrade 1
qm set 100 --ipconfig0 ip=dhcp

qm clone 9000 101 --name k3s-server-02
qm resize 101 scsi0 32G
qm set 101 --net0 virtio,bridge=vmbr0,macaddr=BC:24:11:02:57:C8
qm set 101 --cpu host
qm set 101 --memory 4096
qm set 101 --cores 2
qm set 101 --ciuser wim
qm set 101 --sshkey id_rsa.pub
qm set 101 --ciupgrade 1
qm set 101 --ipconfig0 ip=dhcp

qm clone 9000 102 --name k3s-server-03
qm resize 102 scsi0 32G
qm set 102 --net0 virtio,bridge=vmbr0,macaddr=BC:24:11:4F:A3:86
qm set 102 --cpu host
qm set 102 --memory 4096
qm set 102 --cores 2
qm set 102 --ciuser wim
qm set 102 --sshkey id_rsa.pub
qm set 102 --ciupgrade 1
qm set 102 --ipconfig0 ip=dhcp

qm clone 9000 200 --name k3s-worker-01
qm resize 200 scsi0 32G
qm set 200 --net0 virtio,bridge=vmbr0,macaddr=BC:24:11:12:B1:D2
qm set 200 --cpu host
qm set 200 --memory 8192
qm set 200 --cores 4
qm set 200 --ciuser wim
qm set 200 --sshkey id_rsa.pub
qm set 200 --ciupgrade 1
qm set 200 --ipconfig0 ip=dhcp

qm clone 9000 201 --name k3s-worker-02
qm resize 201 scsi0 32G
qm set 201 --net0 virtio,bridge=vmbr0,macaddr=BC:24:11:07:BA:1A
qm set 201 --cpu host
qm set 201 --memory 8192
qm set 201 --cores 4
qm set 201 --ciuser wim
qm set 201 --sshkey id_rsa.pub
qm set 201 --ciupgrade 1
qm set 201 --ipconfig0 ip=dhcp

qm clone 9000 300 --name k3s-lb-01
qm resize 300 scsi0 32G
qm set 300 --net0 virtio,bridge=vmbr0,macaddr=BC:24:11:EA:6D:1F
qm set 300 --cpu host
qm set 300 --memory 2048
qm set 300 --cores 1
qm set 300 --ciuser wim
qm set 300 --sshkey id_rsa.pub
qm set 300 --ciupgrade 1
qm set 300 --ipconfig0 ip=dhcp

qm clone 9000 301 --name k3s-lb-02
qm resize 301 scsi0 32G
qm set 301 --net0 virtio,bridge=vmbr0,macaddr=BC:24:11:6B:DC:F9
qm set 301 --cpu host
qm set 301 --memory 2048
qm set 301 --cores 1
qm set 301 --ciuser wim
qm set 301 --sshkey id_rsa.pub
qm set 301 --ciupgrade 1
qm set 301 --ipconfig0 ip=dhcp

qm start 100
qm start 101
qm start 102
qm start 200
qm start 201
qm start 300
qm start 301
```

## Install and setup HAProxy and Keepalived

Install HAProxy and Keepalived on `k3s-lb-01` and `k3s-lb-02`:

```bash
sudo apt-get install haproxy keepalived
```

Add (at the end of file) the following to `/etc/haproxy/haproxy.cfg` on `k3s-lb-01` and `k3s-lb-02`:

```bash
frontend k3s-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend k3s-backend

backend k3s-backend
    mode tcp
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s
    server k3s-server-01 192.168.1.201:6443 check
    server k3s-server-02 192.168.1.202:6443 check
    server k3s-server-03 192.168.1.203:6443 check
````

Add the following to `/etc/keepalived/keepalived.conf` on `k3s-lb-01` (MASTER):

```bash
global_defs {
  enable_script_security
  script_user root
}

vrrp_script chk_haproxy {
    script 'killall -0 haproxy' # faster than pidof
    interval 2
}

vrrp_instance haproxy-vip {
    interface eth0
    state MASTER # MASTER on k3s-lb-01, BACKUP on k3s-lb-02
    priority 200 # 200 on k3s-lb-01, 100 on k3s-lb-02

    virtual_router_id 51

    virtual_ipaddress {
        192.168.1.220/24
    }

    track_script {
        chk_haproxy
    }
}
```

Add the following to `/etc/keepalived/keepalived.conf` on `k3s-lb-02` (BACKUP):

```bash
global_defs {
  enable_script_security
  script_user root
}

vrrp_script chk_haproxy {
    script 'killall -0 haproxy' # faster than pidof
    interval 2
}

vrrp_instance haproxy-vip {
    interface eth0
    state BACKUP # MASTER on k3s-lb-01, BACKUP on k3s-lb-02
    priority 100 # 200 on k3s-lb-01, 100 on k3s-lb-02

    virtual_router_id 51

    virtual_ipaddress {
        192.168.1.220/24
    }

    track_script {
        chk_haproxy
    }
}
```

Restart `HAProxy` and `Keepalived` on `k3s-lb-01` and `k3s-lb-02`:

```bash
sudo systemctl restart haproxy
sudo systemctl restart keepalived
```

> The VIP `192.168.1.220` can now be used to access the Kubernetes API server and to join `K3s` worker nodes (for
> control plane nodes this is not required).  
> All of this will be explained in the next chapters.

## Install first server (control plane node)

Use the `--cluster-init` flag to create the first server in the cluster and initialize the embedded `etcd` datastore for
high availability (HA).  
Use the `--tls-san` flag to specify the fixed IP address that will be used by `Keepalived` as virtual IP to access the
cluster.  
To install a specific version of `K3s`, set the `INSTALL_K3S_VERSION` environment variable before running the
installation
script.  
All `K3s` versions can be found here: [k3s release](https://github.com/k3s-io/k3s/releases)

```bash
# Use a specific version of K3s.
export INSTALL_K3S_VERSION=v1.31.6+k3s1

# Installs K3s, initializes the first server in HA mode and disables Traefik and the ServiceLB load balancer.
sudo curl -sfL https://get.k3s.io | sh -s - server --cluster-init --disable="traefik" --disable="servicelb" --tls-san=192.168.1.220 --node-taint CriticalAddonsOnly=true:NoExecute

# Displays the kubeconfig file for accessing the K3s cluster.
sudo cat /etc/rancher/k3s/k3s.yaml

# Displays the token used for joining nodes to the K3s cluster.
sudo cat /var/lib/rancher/k3s/server/token
```

## Install additional servers (control plane nodes)

```bash
export INSTALL_K3S_VERSION=v1.31.6+k3s1
export K3S_URL=https://192.168.1.201:6443
export K3S_TOKEN=<token from first server>

sudo curl -sfL https://get.k3s.io |sh -s - server --disable="traefik" --disable="servicelb" --node-taint CriticalAddonsOnly=true:NoExecute
```

## Install agents (worker nodes)

```bash
export INSTALL_K3S_VERSION=v1.31.6+k3s1
export K3S_URL=https://192.168.1.220:6443
export K3S_TOKEN=<token from first server>

sudo curl -sfL https://get.k3s.io | sh -s - agent
```

## Set up kubectl to use the K3s kubeconfig file

From one of the servers copy the `k3s.yaml` file to your local machine (example to $HOME/.kube/k3s.yaml).  
Edit the `k3s.yaml` file and correct the server address with the `Keepalived` virtual IP address **192.168.1.220**, this
ensures high availability for the Kubernetes API Server.   
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

## Tainting Control Plane Nodes

To prevent workloads from running on the control plane nodes, you can taint them.  
This ensures that only specific pods with the corresponding tolerations can be scheduled on these nodes.

Taint `--node-taint CriticalAddonsOnly=true:NoExecute` has been added during the installation of the `K3s` server nodes.

## Bootstrapping with Argo CD and Deploying Workloads

Now that your cluster is set up, you can deploy workloads on it. However, before doing so it is recommended to bootstrap
the cluster with Argo CD for easier and more efficient management of your Kubernetes resources.  
Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. By integrating it into your cluster, you can
manage deployments from Git repositories and streamline your deployment process.  
After bootstrapping with Argo CD, you can deploy your workloads seamlessly, with Argo CD taking care of the application
management and lifecycle.

Argo CD setup and application deployment can be done using the following repositories:

* Argo CD installation and
  configuration: [https://github.com/wim-vdw/argocd-setup](https://github.com/wim-vdw/argocd-setup)
* Argo CD application definitions: [https://github.com/wim-vdw/argocd-apps](https://github.com/wim-vdw/argocd-apps)
* Kubernetes manifests or Helm charts used by Argo CD
  applications: [https://github.com/wim-vdw/argocd-k8s-resources](https://github.com/wim-vdw/argocd-k8s-resources)

## Example Workload after Bootstrapping with Argo CD

After bootstrapping the cluster with Argo CD and deploying core and test applications the workload is distributed across
the nodes.

Core applications:

* `Ingress NGINX` controller for Kubernetes
* `MetalLB` load-balancer implementation for bare metal Kubernetes clusters
* `Reloader` controller to watch changes in Kubernetes ConfigMaps and Secrets
* `cert-manager` controller to obtain and renew TLS certificates before they expire

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

- [X] Add workload example after bootstrapping Argo CD and deploying core and test applications.
- [X] Add VM creation with Proxmox and cloud-init.
- [X] Add new Argo CD configuration repositories.
- [X] Analyse performance/stability of etcd datastore.
- [X] Make Kubernetes API Server high available with `HAProxy` and `Keepalived`.
