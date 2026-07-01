# Installing Kubernetes (containerd + calico)

## Specs:
**Control-Plane:**
- CPU: 2vCPU
- MEM: 8GiB
- DISK: 32GB
- SO: Ubuntu Server 22.04

**Worker(s):**
- CPU: 2vCPU
- MEM: 4GiB
- DISK: 32GB
- SO: Ubuntu Server 22.04

---
 
## Step by Step:

#### 1. Pre-installing 
```bash
### IP Fowarding
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Reload the configs
sudo sysctl --system

### SWAP
swapoff -a
# Go to /etc/fstab and remove the swap from there too!!!
```

**-> After, reboot the system**

---

#### 2. Installing containerd
Install containerd:
```bash
sudo apt-get install containerd
```

After installing we need to force containerd to use systemd:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Restart the service
systemctl restart containerd
```

---

#### 3. Installing kubeadm, kubelet and kubectl
```bash
sudo apt get update

# Necessary packets
sudo apt-get install -y apt-transport-https ca-certificates curl gpg 

#  Getting Pub key to repo
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Adding the repo
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d kubernetes.list

# Installing the K8S packets
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Stopping the update to K8S packets
sudo apt-mark hold kubelet kubeadm kubectl
```

**K8S after v1.22 uses systemd as cgroup driver for default, so we don't have to force it.**

---

#### 4. Creating a cluster with kubeadm
First we need to get the ssh keys from the nodes and archive.
To achieve this you have to connect by ssh on each node from all nodes.
For example, in my case:
```bash
ssh 192.168.50.220 #Control-Plane
ssh 192.168.50.221 #Worker-1
ssh 192.168.50.222 #Worker-2
```

### ⚠️These steps are on Control-Plane only⚠️
**Kube init:**
```bash
kubeadm init --apiserver-advertise-address=[K8S Control-Plane] --pod-network-cidr=[CIDR to Internal Network]
```
**In my case I used:**
```bash
kubeadm init --apiserver-advertise-address=192.168.50.220 --pod-network-cidr=192.168.55.0/24
```

**If was everything fine, now we gonna do:**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
****⚠️ Remember to save the kubejoin for later!****

---

#### Setting up the Calico
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

cd /etc/kubernetes
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O
```
**Edit the custom-resource.yaml using the pod-network-cidr if you haven't changed leave it as default**
```bash
vim custom-resources.yaml

From
> cidr: 192.168.0.0/16
To
> cidr: 192.168.55.0/24
```

**And apply the custom-resource:**
```kubectl apply -f custom-resources.yaml```
#### ⚡ Bazinga!
```
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-8f4f68cc5-vcswx   1/1     Running   0          3m36s
calico-node-6b9c8                         1/1     Running   0          3m3s
calico-typha-5c4f465cd8-9gg5n             1/1     Running   0          3m41s
csi-node-driver-fd7rz                     2/2     Running   0          3m38s
```

---

#### 5. Adding the workes to the cluster
**Run the kubeadm join in the workers, and to check if they was added to cluster:**
```bash
$ kubectl get nodes

NAME           STATUS     ROLES           AGE   VERSION
k8s-master     Ready      control-plane   23m   v1.36.2
k8s-worker-1   NotReady   <none>          8s    v1.36.2
k8s-worker-2   NotReady   <none>          13s   v1.36.2
```

**If you didn't save the join:**
```
kubeadm token create --print-join-command
```
---

#### 6. Extra (Life Savers)
#### Kubectl completion
**Bash:**
```bash
kubectl completion bash > ~/.kube/completion.bash.inc
printf "
# kubectl shell completion
source '$HOME/.kube/completion.bash.inc'
" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
