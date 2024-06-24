# kubeadm-adventures

I have been jamming with kubeadm and I wanted to capture all my adventures and errors during the process. 

Here I will tell how I: 
- Configured kubernetes cluster (control panel + worker node) on virtual machines (kvm virt-manager) using *kubeadm* from *scratch*
- Enabled web ui dashboard for k8s cluster and accessed it from host
- Deployed jenkins application and accessed it with external load balancer from host 

## Prereq

This is my configuration
1. host Ubuntu 23.10 (I think you can use any ubuntu version and maybe even other distros)
2. virt-manager (just google `<your distro name> install virt-manager`)
3. Debian 12 iso (we are going to use them as a control panel vm and node vm)

## Step 1: Create VMs

I hope you know how to create VMs, because I won't go into details. 

During installation process, set hostname for control panel VM to `server`, for node VM to `node0`.

VM configuration is:
- Disk size: `20GB`
- CPU: `1` for `server`, `2` for `node0`
- RAM: `2GB` for `server`, `4GB` for `node0`

Do not install any desktop environment, instead install `ssh server`. 

Do not set any password for `root` account.

User name used in this guide is going to be `biden`.

### IP configuration and SSH access 

`virt-manager` create network between your host and your VM. What does it mean? It means you can access your VMs from host by plain IP address in VM. This is very important, I recommend you to write down ip addresses of your machines. 

In my case I got 
```
# THIS IS MY SERVER IP NOT YOURS
SERVER_IP=192.168.122.52
# THIS IS MY NODE IP NOT YOURS 
NODE_IP=192.168.122.224
```

You can check your VM ip and your VM network using this command 
```
ip a
```

## Step 2: Installing all tools

You should repeat these installation steps on `server` and on `node0` machines 


### 1. Install kube tools

```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

### 2. Install containerd 
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt install -y containerd.io
sudo systemctl enable --now containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
Source: https://docs.docker.com/engine/install/debian/
### 3. Disable swap (permanently)

```
sudo swapoff -a  
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
Source: https://askubuntu.com/questions/214805/how-do-i-disable-swap

## Step 3: kubeadm create cluster and add first node

`control-panel.sh` script is really simple. I encourage you to read it before execution.

### In `server`

run `control-panel.sh` script 
```
git clone https://github.com/mv-kan/kubeadm-adventures.git
cd kubeadm-adventures
# in terminal
# 192.168.122.52 - is my ip address of server 
./scripts/control-panel.sh 192.168.122.52
```

After that your gonna see command that looks like this 

```
# 192.168.122.52 - ip address of server (control plane)
sudo kubeadm join 192.168.122.52:6443 --token qvcj8q.mfl5eurdwfh6u92i --discovery-token-ca-cert-hash sha256:bd5626b7a4f8f4fc1927f5e5b63287e6147a36e17b42d5c44f874d6af43d26f3
``` 

Paste this command into your `node0` machine. 

If you miss this output you can create join command whenever you want. 
```
[root@devops ~]# kubeadm token generate
hp9b0k.1g9tqz8vkf78ucwf
[root@devops ~]# kubeadm token create hp9b0k.1g9tqz8vkf78ucwf --print-join-command
[W1022 19:51:03.885049   26680 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
kubeadm join 192.168.56.110:6443 --token hp9b0k.1g9tqz8vkf78ucwf     --discovery-token-ca-cert-hash sha256:32eb67948d72ba99aac9b5bb0305d66a48f43b0798cb2df99c8b1c30708bdc2c
# Source https://monowar-mukul.medium.com/kubernetes-create-a-new-token-and-join-command-to-rejoin-add-worker-node-74bbe8774808
```

Sources:
- https://www.youtube.com/watch?v=xX52dc3u2HU
- https://github.com/techiescamp/kubeadm-scripts

## Step 4: Dashboard, jenkins, external load balancer on premises 

### 1. External Load Balancer

On `server` machine
```
cd ~
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
metallb 
```
kubectl create ns metallb-system
helm upgrade --install -n metallb-system metallb \
    oci://registry-1.docker.io/bitnamicharts/metallb
```

Before applying L2-range-allocation.yaml, please change ip address range if applicable 
```
cd <this repo folder>
kubectl apply -f ./k8s/metallb/L2-range-allocation.yaml
```

### 2. Dashboard
Dashboard web ui
```
# Add kubernetes-dashboard repository
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
# Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
# enter this repo directory and create new service for kong proxy (with metallb annotation)
cd kubeadm-adventures
kubectl apply -f ./k8s/dashboard/ -n kubernetes-dashboard
```

### 3. Jenkins 
Jenkins is in submodule `k8s-jenkins-setup`
```
# on server machine
cd ~
git clone https://github.com/mv-kan/k8s-jenkins-setup.git
cd k8s-jenkins-setup
git checkout kubeadm-adventures
kubectl create namespace devops
kubectl apply -f ./k8s/jenkins  
``` 

### Metallb static IP 

https://metallb.universe.tf/usage/