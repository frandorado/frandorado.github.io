---
layout: post
title:  Raspberry Pi Cluster with Kubernetes
author: frandorado
categories: [kubernetes]
tags: [kubernetes, k8s, raspberry, pi, cluster, k3s, home-assistant, kubectl]
image: assets/images/posts/2022-07-28/header.jpg
toc: true
hidden: true
---

The goal of this project is to create a HomeLab Server to deploy our own applications using a cluster of Raspberry Pi with Kubernetes. 

## Materials

* Two Rasbperry Pi (ideally Rpi4 with 4Gb RAM at least for master node)
* Wifi or Ethernet connection
* Two SD Cards

## Master node installation

1. Install the SO (see [Appendix A](#appendix-a-how-to-install-raspberry-pi-os))
2. Put the SD Card in the Rasbperry Pi and turn on it
3. Access to the node with ssh
```bash
ssh master01@master01.local
```
4. Modify the file `/boot/cmdline.txt` with sudo adding next value. Edit the ip address value for your case. In my case I decided to put the static ip address 192.168.1.200 
```text
cgroup_memory=1 cgroup_enable=memory ip=192.168.1.200::192.168.1.1:255.255.255.0:master01.local:eth0:off
```
5. Restart the server and access again with ssh
```bash
sudo reboot
```
```bash
ssh master01@master01.local
```
6. Install k3s server. Replace the static address with your value.
```bash
curl -sfL https://get.k3s.io | sh -s - --bind-address 192.168.1.200 --write-kubeconfig-mode 644
```
7. Check that the cluster is running correctly
```bash
> kubectl cluster-info
Kubernetes control plane is running at https://192.168.1.200:6443
CoreDNS is running at https://192.168.1.200:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://192.168.1.200:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

8. Copy the value of node-token necessary to install the Agent
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

## Agent node installation

1. Install the SO (see [Appendix A](#appendix-a-how-to-install-raspberry-pi-os))
2. Put the SD Card in the Rasbperry Pi and turn on it
3. Access to the node with ssh
```bash
ssh agent01@agent01.local
```
4. Modify the file `/boot/cmdline.txt` with sudo adding next value. Edit the ip address value for your case. In my case I decided to put the static ip address 192.168.1.200
```text
cgroup_memory=1 cgroup_enable=memory
```
5. Restart the server and access again with ssh
```bash
sudo reboot
```
```bash
ssh agent01@agent01.local
```
6. Go to master node with ssh and copy the noden-token value required for the connection between the Agent and the Master
```bash
ssh master01@master01.local
```
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
7. Install k3s agent and replace the ip and the value of the token
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.200:6443 K3S_TOKEN=XXXXXX sh -
```
8. Test the agent in the master node
```
> master01@master01:~ $ k3s kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
master01   Ready    control-plane,master   57m   v1.24.3+k3s1
agent01    Ready    <none>                 20s   v1.24.3+k3s1
```

## Access to the cluster 

Configure kubectl from your client to access to the cluster

1. Install kubectl
2. Copy the configuration from master node
```
sudo cat /etc/rancher/k3s/k3s.yaml
```
3. Create a file `~/.kube/config` and paste the content replacing the values from master node
```yml
apiVersion: v1
kind: Config
clusters:
  - cluster:
      server: "https://192.168.1.200:6443"
      certificate-authority-data: LS0....
    name: k3s-cluster
contexts:
  - context:
      cluster: "k3s-cluster"
      user: "k3s-admin"
    name: k3s
    current-context: k3s
users:
  - name: k3s-admin
    user:
      client-certificate-data: LS0t...
      client-key-data: LS0t...
```
4. Invoke use context and check the connection
```bash
kubectl config use-context k3s
```
```
> kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
agent01    Ready    <none>                 23m   v1.24.3+k3s1
master01   Ready    control-plane,master   81m   v1.24.3+k3s1
```

## Example: Install Home Assistant

Installation:
```bash
helm repo add k8s-at-home https://k8s-at-home.com/charts/
helm repo update
helm install home-assistant k8s-at-home/home-assistant
```

Now with a port-forward:
```bash
kubectl port-forward deployment/home-assistant 8123:8123
```
you can check your installation in your browser `localhost:8123`

Uninstallation:
```bash
helm uninstall home-assistant
```

More info in [Helm home-assistant](https://artifacthub.io/packages/helm/k8s-at-home/home-assistant)

## Appendix A: How to install Raspberry Pi OS

* Install Raspberry Pi OS using Raspberry Pi Imager in the SD Card http://raspberrypi.com/
* Use Raspberry Pi OS Lite (64 bits)
* Advanced options:
  * Set hostname. Example: "master01.local" or "agent01.local" depending on the type of node
  * Enable SSH using password authentication
  * Define user/password. Example: master01 or agent01
  * Set locale settings
  * Wifi is your connection is not using ethernet
  
## Appendix B: Uninstall K3S

Master
```bash
/usr/local/bin/k3s-uninstall.sh
```
Agent
```bash
/usr/local/bin/k3s-agent-uninstall.sh
```


