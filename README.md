# Configuring kubernetes for LVM

## Disable SELinux and swap

See https://computingforgeeks.com/install-kubernetes-cluster-on-centos-with-kubeadm/

This needs to be done on each node in the cluster.

```console
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
```

Turn off swap

```console
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```

Disable firewall

```console
sudo systemctl disable --now firewalld
```

## Installing k3s

For the server node (`lvm-hub`) do

```console
curl -sfL https://get.k3s.io | sh -
```

This installs k3s as a service [see here](https://docs.k3s.io/quick-start). It can be started/stopped with `systemctl start/stop k3s`.

I had to use `sudo visudo` and add `/usr/local/bin` to `secure_paths` to be able to use the `k3s` command with `sudo`.

Add `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` to `.bashrc` or `.profile` to have the normal `kubectl` use the `k3s` configuration.

With this configuration `k3s` and `kubectl` need to run as sudo. You can change the permissions of `/etc/rancher/k3s/k3s.yaml` to let your user run without sudo.

For agent nodes

```console
curl -sfL https://get.k3s.io | K3S_URL=https://lvm-hub.lco.cl:6443 K3S_TOKEN=K3S_TOKEN sh -
```

where `K3S_TOKEN` is the token in `/var/lib/rancher/k3s/server/node-token` in the server node.

## Installing the dashboard

Run

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

to install the dashboard. Then create a file `~/dashboard.admin-user.yml` with contents

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Apply the file with `sudo k3s kubectl apply -f ./dashboard.admin-user.yml`. Create a bearer token with `sudo k3s kubectl -n kubernetes-dashboard create token admin-user`.

You can create the token with `--duration=87600h` to prevent the token expiring.

Proxy with `sudo k3s kubectl proxy --port 8081` and access the dashboard at `http://localhost:8081/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/workloads?namespace=_all`.

## Running deployments

Deployments are included in this repository. Run them with

```console
kubectl apply -f <DEPLOYMENT.yml>
```
