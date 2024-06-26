# Setup a Highly Available Kubernetes Cluster using kubeadm

Follow this documentation to set up a highly available Kubernetes cluster using Ubuntu 20.04 LTS.

This documentation guides you in setting up a cluster with two master nodes, two worker node and a load balancer node using HAProxy.




| Role          | IP            | OS     | RAM | CPU |
|---------------|---------------|--------|-----|-----|
| Load Balancer | 10.0.2.5 | Ubuntu 20.04 | 2G  | 1   |
| Master-01     | 10.0.2.4 | Ubuntu 20.04 | 2G  | 2   |
| Master-02     | 10.0.2.6 | Ubuntu 20.04 | 2G  | 2   |
| WorkerNode-01 | 10.0.2.7 | Ubuntu 20.04 | 2G  | 1   |
| WorkerNode-02 | 10.0.2.8 | Ubuntu 20.04 | 2G  | 1   |

##### Switch to Root user on all nodes
```
sudo -i
```

## Set up load balancer node
##### Install Haproxy
``` 
apt update && apt install -y haproxy
```
##### Configure haproxy

Append the below lines to /etc/haproxy/haproxy.cfg

```
frontend kubernetes-frontend
    bind 10.0.2.5:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server Master-01 10.0.2.4:6443 check fall 3 rise 2
    server Master-02 10.0.2.6:6443 check fall 3 rise 2
```
    
##### Restart haproxy service
```
systemctl restart haproxy
```


## Run on all kubernetes nodes except Load Balancer Node (Master-01, Master-02, WorkerNode-01, WorkerNode-02)
##### Disable Firewall
```
ufw disable
```
##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Update sysctl settings for Kubernetes networking
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

##### Install Docker Engine or runtime for Ubuntu on each of the Nodes in the cluster (except Load balancer Node) so that Pods can run there.

1. Run the following command to uninstall all conflicting packages:
```
 for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo
```
2. Set up Docker's apt repository.

```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

```

3. Install the docker package
   ```
   sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   
   ```

##### Installing kubeadm, kubelet and kubectl

1. Update the apt package
   ```
   # Rotate the Key Again else you wont be able to uupdate
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

   sudo apt-get update
   # apt-transport-https may be a dummy package; if so, you can skip that package
   sudo apt-get install -y apt-transport-https ca-certificates curl gpg
   ```
2. Download the public signing Key
   ```
   # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
   sudo mkdir -p -m 755 /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```

3. Add the appropriate Kubernetes apt repository to setup Kubernetes
   ```
   # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

3. Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:

   ```
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

4. (Optional) Enable the kubelet service before running kubeadm:

  ```
  sudo systemctl enable --now kubelet
  ```
NOTE: The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.

##### configure cgroup Driver 

The kubelet and the container runtime need to use a cgroup driver. It's critical that the kubelet and the container runtime use the same cgroup driver and are configured the same. Here since we are using kubeadm will MUST use ```systemd``` as cgroupfs 

1. Append the below
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd into /etc/sysconfig/kubelet
  ```
  cat /etc/sysconfig/kubelet
  ```
2. edit docker cgroup drive
   First confirm that the "Cgroup Driver: cgroupfs" and change it to "Cgroup Driver:systemd"
   ```
   docker info
   ``` 
3. Create (or edit) the /etc/docker/daemon.json configuration file and to include the following:
```
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

4. Run this on all the Nodes (Else you will get this "Error "[ERROR CRI]: container runtime is not running: output: time="2024-04-16T20:23:48Z" level=fatal msg="validate service connection: validate CRI v1 
runtime API for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService")
   ```
   sudo rm /etc/containerd/config.toml
   sudo systemctl restart containerd
   ```

## On any one of the Kubernetes master node (Eg: Master-01)
Initialize Kubernetes Cluster

```
kubeadm init --control-plane-endpoint="10.0.2.5:6443" --upload-certs --apiserver-advertise-address=10.0.2.4 --pod-network-cidr=192.168.0.0/16
```

Copy the commands to join other master nodes and worker nodes.

Note: If you run the command below on any of the WorkerNodes, you will find out that the nodes are not Ready yet thus we need to deploy a network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf get node
```

##### Deploy Calico network. Run the command below on one of the WorkerNode.
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml
```

Join other nodes to the cluster (kmaster2 & kworker1).

#### NOTE: MAKE SURE YOU KEEP THE JOIN COMMAND SAFE AS YOU WILL BE USING THE SAME COMMAND TO JOIN MORE NODE TO THE CLUSTER


##### Downloading kube config to your local machine
###### On your host machine make a directory
```
mkdir ~/.kube
scp root@132.16.16.101:/etc/kubernetes/admin.conf ~/.kube/config
```
#### OR copy the content of the admin.conf into your local host file  ~/.kube/config


##### Verifying the cluster
```
kubectl cluster-info
kubectl get nodes
kubectl -n kube-system get all
kubectl get cs
```

## THE END !!!




# Troubleshooting Kubernetes

###  Error message: If you get the error below when running ```sudo apt update```, 
Hit:1 https://packages.microsoft.com/ubuntu/22.04/prod jammy InRelease.

Hit:2 http://azure.archive.ubuntu.com/ubuntu jammy InRelease.

Get:3 http://azure.archive.ubuntu.com/ubuntu jammy-updates InRelease [109 kB].


Get:4 http://azure.archive.ubuntu.com/ubuntu jammy-backports InRelease [99.8 kB].

Get:5 http://azure.archive.ubuntu.com/ubuntu jammy-security InRelease [110 kB].

Get:6 https://cloud.r-project.org/bin/linux/ubuntu jammy-cran40/ InRelease [3626 B]

### Solution: Check the apt directory for any unfamilar pkg. Here I found kubernetes.list and docker.list when I checked /etc/apt/sources.list.d

$ rm /etc/apt/sources.list.d/kubernetes.list.

$ rm /etc/apt/sources.list.d/docker.list.

$ sudo reboot.

