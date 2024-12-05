# K3s Setup Notes

## VM overview

| VM Name       | MAC Address       | IP Address    | Memory (Gb) | Disk Size (Gb) | CPU Cores |
|---------------|-------------------|---------------|-------------|----------------|-----------|
| k3s-server-01 | BC:24:11:66:6F:07 | 192.168.1.201 | 2           | 32             | 2         |
| k3s-server-02 | BC:24:11:02:57:C8 | 192.168.1.202 | 2           | 32             | 2         |
| k3s-server-03 | BC:24:11:4F:A3:86 | 192.168.1.203 | 2           | 32             | 2         |
| k3s-worker-01 | BC:24:11:12:B1:D2 | 192.168.1.211 | 2           | 32             | 2         |
| k3s-worker-02 | BC:24:11:07:BA:1A | 192.168.1.212 | 2           | 32             | 2         |

## Install first server (control plane node)

Use the `--cluster-init` flag to create the first server in the cluster and initialize the embedded `etcd` datastore for
high availability (HA).  
To install a specific version of k3s, set the `INSTALL_K3S_VERSION` environment variable before running the installation
script.  
All k3s versions can be found here: [k3s release](https://github.com/k3s-io/k3s/releases)

```bash
# Use a specific version of k3s.
export INSTALL_K3S_VERSION=v1.30.6+k3s1

# Installs k3s, initializes the first server in HA mode and disables Traefik and the ServiceLB load balancer.
sudo curl -sfL https://get.k3s.io | sh -s - server --cluster-init --disable="traefik" --disable="servicelb"

# Displays the kubeconfig file for accessing the k3s cluster.
sudo cat /etc/rancher/k3s/k3s.yaml

# Displays the token used for joining nodes to the k3s cluster.
sudo cat /var/lib/rancher/k3s/server/token
```

## Install additional servers (control plane nodes)

```bash
export INSTALL_K3S_VERSION=v1.30.6+k3s1
export K3S_URL=https://192.168.1.201:6443
export K3S_TOKEN=<token from first server>

sudo curl -sfL https://get.k3s.io |sh -s - server --disable="traefik" --disable="servicelb"
```

## Install agents (worker nodes)

```bash
export INSTALL_K3S_VERSION=v1.30.6+k3s1
export K3S_URL=https://192.168.1.201:6443
export K3S_TOKEN=<token from first server>

sudo curl -sfL https://get.k3s.io | sh -s - agent
```

## Set up kubectl to use the k3s kubeconfig file.

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
