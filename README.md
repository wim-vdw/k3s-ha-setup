# K3s Setup Notes

## VM overview

| VM Name       | MAC Address       | IP Address    | Memory (Gb) | Disk Size (Gb) | CPU Cores |
|---------------|-------------------|---------------|-------------|----------------|-----------|
| k3s-server-01 | BC:24:11:66:6F:07 | 192.168.1.201 | 2           | 32             | 2         |
| k3s-server-02 | BC:24:11:02:57:C8 | 192.168.1.202 | 2           | 32             | 2         |
| k3s-server-03 | BC:24:11:4F:A3:86 | 192.168.1.203 | 2           | 32             | 2         |
| k3s-worker-01 | BC:24:11:12:B1:D2 | 192.168.1.211 | 2           | 32             | 2         |
| k3s-worker-02 | BC:24:11:07:BA:1A | 192.168.1.212 | 2           | 32             | 2         |

## Install first server

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
