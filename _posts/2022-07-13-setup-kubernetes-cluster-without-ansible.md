---
layout: post
title: Setup Kubernetes cluster without Ansible
subtitle: 
header-img: img/in-post/2022-07-13/kubernetes-cluster.jpeg
header-style: text
catalog: true
tags:
  - cloud
---

How to setup Kubernetes cluster with the AWS EC2 nodes RHEL/CentOS OS and without Ansible?
In this blog, I will introduce you about installing Kubernetes cluster in REHL7 OS.

## Prerequisite
- kubeadm
- docker/containerd (container runtime)
- kubectl

## Install kubeadm, kubectl, kubelet
In almose cases, all you guys only want install specific kubernetes version. In my case, I need to install kubernetes version 1.22.10.
I will guide you how to install kubernetes cluster version 1.22.10. If you want to install another version, you can modify version of kubeadm, kubectl, kubelet when you install.

You need to install kubeadm, kubectl, kubelet in all nodes of cluster. I will do example with 1 master node and 1 worker node.
Please bootstrap script below on each node:

```bash
cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

sudo yum update -y
sudo yum install kubeadm-1.22.10 kubectl-1.22.10 kubelet-1.22.10
```

If you want to install another version, you can follow this below:

```bash
sudo yum install kubeadm-<VERSION> kubectl-<VERSION> kubelet-<VERSION>
```

Please note that dependencies between version of kubeadm, kubectl and kubelet.

If you intalled latest version and want to downgrade, you can run this command:

```bash
sudo yum downgrade  kubeadm-<VERSION> kubectl-<VERSION> kubelet-<VERSION>
```

Make sure you exclude kube* packages in /etc/yum.conf by add this line to end of file /etc/yum.conf

```txt
exclude=kube*
```


## Install Docker
In this document, I use docker as container runtime for kubernetes cluster. You can use containerd instead.

Run script below:

```bash
sudo yum update -y
sudo yum install fuse3-devel
sudo wget http://mirror.centos.org/centos/7/extras/x86_64/Packages/fuse-overlayfs-0.7.2-6.el7_8.x86_64.rpm
sudo yum localinstall fuse-overlayfs-0.7.2-6.el7_8.x86_64.rpm -y
sudo wget http://mirror.centos.org/centos/7/extras/x86_64/Packages/slirp4netns-0.4.3-4.el7_8.x86_64.rpm
sudo yum localinstall slirp4netns-0.4.3-4.el7_8.x86_64.rpm
sudo wget http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.107-3.el7.noarch.rpm
sudo yum localinstall container-selinux-2.107-3.el7.noarch.rpm -y
sudo yum install docker-ce docker-ce-cli containerd.io -y
```

## Config master node
Configure IP tables:

```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# reload
sudo sysctl --system
```

Update <b>sysctl</b>:

```bash
# if these attributes are existed, please update these value to 1 and do not append these lines below
sudo echo net.bridge.bridge-nf-call-iptables = 1 >> /etc/sysctl.conf 
sudo echo net.ipv4.ip_forward = 1 /etc/sysctl.conf 
```

Configure cgroup driver:

In my case, I use docker as container runtime and cgroup will be <b>cgroupfs</b>. If you use containerd as container runtime, cgroup will be <b>systemd</b>

Kubeconfig:

```yaml
# kubeadm-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: cgroupfs # Modify it to systemd if you use containerd
containerRuntime: remote
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
localAPIEndpoint:
  advertiseAddress: "10.10.0.216" # Modify it with your IP of Master node
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  timeoutForControlPlane: 4m0s
  certSANs:
    - "10.100.1.1"
    - "10.10.0.216"  # Modify it with your IP of Master node
clusterName: demo # Modify it if you want to change cluster name
etcd:
  local:
    dataDir: /var/lib/etcd
networking:
  podSubnet: "10.124.0.0/16" # Modify it so that it does not match with you subnet IP
```

Suppose that you store this kubeconfig in file $HOME/kube-config.yaml

Init kubeadm:

```bash
kubeadm init --config $HOME/kube-config.yaml
```

After init successfully, you can check it:

```bash
kubectl get nodes
kubectl get pods -A
```

In stdout, you will have token to join worker node. If you lose that token, you can re-generate it by:
```bash
kubectl token create --print-join-command
```

Install Container Network Interface (CNI): You can chose Flannel or Calico or anything else. In my case, I installed Flannel because it's very simple but Canico may be best practice for almose cases.

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml 
```

After apply this, you can see a pod running in namespace <b>kube-flannel</b>

```bash
kubectl get pods -n kube-flannel
```

## Configure worker node
Make sure version of kubeadm, kubectl, kubelet is same with master node.

Join the node to cluster:
```bash
# Join with your join command
# This is example
kubeadm join kubemaster.bpic.local:6443 --token cll0gw.50jagb64e80uw0da \ 
    --discovery-token-ca-cert-hash sha256:4d699e7f06ce0e7e80b78eadc47453e465358021aee52d956dceed1dfbc0ee34
```

After join successfully, you can check:

```bash
kubectl get nodes
```

## Conclusion
One important thing is that version of kubeadm, kubectl, kubectl. If you have any error, please check this first.

We completed to set up Kubernetes cluster with RHEL7 OS. And now, We try it :smile:



