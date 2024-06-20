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
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ (Debian based distibutions)

Our runtime is going to be containerd: 
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
https://unix.stackexchange.com/questions/626645/how-to-install-containerd-on-debian 
after install don't forget 
```
sudo apt-get install docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now containerd
```

DO NOT FORGET TO DO THAT FOR YOUR `containerd` CONFIGURATION
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd
```
# source https://serverfault.com/questions/1074008/containerd-1-4-9-unimplemented-desc-unknown-service-runtime-v1alpha2-runtimese
cat > /etc/containerd/config.toml <<EOF
[plugins."io.containerd.grpc.v1.cri"]
  systemd_cgroup = true
EOF
systemctl restart containerd
```

### 2. Disable swap (permanently)

```
sudo swapoff -a  
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
Source: https://askubuntu.com/questions/214805/how-do-i-disable-swap

## Step 3: kubeadm create cluster and add first node
https://www.youtube.com/watch?v=xX52dc3u2HU
https://github.com/techiescamp/kubeadm-scripts

`control-panel.sh` and `common.sh` scripts are really simple. I encourage you to read them before execution.

### In `server`

run `control-panel.sh` script 
```
# in terminal
./scripts/control-panel.sh <IP address that is in network with host of current VM>
```

### In `node0`
run `common.sh` script 
```
# in terminal
./scripts/common.sh
```
https://monowar-mukul.medium.com/kubernetes-create-a-new-token-and-join-command-to-rejoin-add-worker-node-74bbe8774808