# Kubernetes Multinode Cluster

This is a small Kubernetes lab project where I built a **3-node Kubernetes cluster** using **Debian 13 VMs** and `kubeadm`. <br>

If you don't know what a VM is, what are you doing here??

The goal of this repo is mainly to document the process of setting up a Kubernetes cluster from scratch, including the container runtime, `kubeadm`, `crictl`, networking, CNI installation, troubleshooting, and a few basic tests to make sure everything is actually working.

The cluster uses **bridged networking**, so the VMs can communicate with each other like regular machines on the same network. (Each machine gets its own IP with DHCP from router)

This isn't meant to be a production-ready Kubernetes deployment per say, but It's a starting point for beginners to learn how the different pieces fit together.


## Environment

This Kubernetes cluster was deployed as a 3-node cluster using Debian 13 virtual machines.

- 3 Debian 13 VMs
- 1 control-plane node
- 2 worker nodes
- Bridged networking enabled on all VMs
- Nodes communicate over the same network
- Cluster bootstrapped using `kubeadm`
- Container runtime: `containerd`
- CNI: Flannel

## Installation
### Get docker from the website for the latest containerd version
https://docs.docker.com/engine/install/debian/ <br>
### Install crictl and kubeadm
https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/ <br>
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ <br>
https://kubernetes.io/docs/reference/setup-tools/kubeadm/ <br>
https://github.com/kubernetes-sigs/cri-tools <br>


### If Anything fails in the process and you need to reset everything
```bash
sudo kubeadm reset -f
sudo rm -rf /var/lib/etcd /etc/kubernetes/manifests/* $HOME/.kube
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni
```

## Getting Started 
### Swap needs to be disabled  
```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### Enable CRI
```bash
sudo cat /etc/containerd/config.toml | grep -A2 disabled_plugins

# If you see disabled_plugins = ["cri"], remove "cri" from that list (or comment the line out), then:

sudo systemctl restart containerd
```

```bash
# After installing crictl
sudo tee /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

sudo crictl info
```


### Node warning fix
```bash
# If you encounter any errors with your nodes not resolving the IP addresses use this
echo "192.168.0.1xx  node1" | sudo tee -a /etc/hosts
```

### Initializing the control-plane (leader)
```bash
# Use ip a | grep inet to find the IP of your node
sudo kubeadm init \
  --apiserver-advertise-address=192.168.0.192 \
  --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install a CNI (Flannel or Calico):
```bash
# Flannel (Aka the easiest)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Calico
https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises
```

### Join your worker nodes
```bash
# or grab a fresh join command if you lost the token from the terminal
sudo kubeadm token create --print-join-command
```

## Debugging and Logging
```bash
sudo systemctl status kubelet --no-pager -l

kubectl get nodes
kubectl get pods -A

# Checking if apiserver container is running
sudo crictl ps -a
sudo crictl logs $(sudo crictl ps -a --name kube-apiserver -q)
sudo crictl pods
sudo crictl ps -a --name kube-apiserver
```

## Deploying a test deployment to check if everything is working
```bash
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --port=80 --type=NodePort

kubectl get pods -o wide
kubectl get svc nginx

kubectl get pods -n kube-flannel
kubectl get pods -n kube-system | grep flannel
kubectl logs -n kube-flannel <flannel-pod-name>
```

## Kubernetes certifications and keys (optional, just for education)
```bash
sudo cat /etc/kubernetes/admin.conf

# For a kubeadm installation, you can typically find the Kubernetes CA here:
/etc/kubernetes/pki/ca.crt

# The admin client certificate/key are stored inside admin.conf as base64 data.
# You can extract them:


sudo kubectl config view \
  --kubeconfig=/etc/kubernetes/admin.conf \
  --raw \
  -o jsonpath='{.users[0].user.client-certificate-data}' |
  base64 -d > /tmp/admin.crt

sudo kubectl config view \
  --kubeconfig=/etc/kubernetes/admin.conf \
  --raw \
  -o jsonpath='{.users[0].user.client-key-data}' |
  base64 -d > /tmp/admin.key


# Checking if the api is accepting the certs
curl \
  --cacert /etc/kubernetes/pki/ca.crt \
  --cert /tmp/admin.crt \
  --key /tmp/admin.key \
  https://127.0.0.1:6443/api/v1
```
