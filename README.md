Setting up a cluster using kubeadm
- CheckList
-- A compatible Linux host. 

-- 2 GB or more of RAM per machine (any less will leave little room for your apps).

-- 2 CPUs or more for control plane machines.

-- Full network connectivity between all machines in the cluster (public or private network is fine).

-- Unique hostname, MAC address, and product_uuid for every node. See here for more details.

-- Certain ports are open on your machines. [See here for more details](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)

Step1: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Visit the given link for the instructions on how to setup kubeadm. Due to versions constraints, ensure to use the link for updated versions instead of the below commands that are version biased[First 3 steps]

Run the following commands on all nodes (Master node and Worker nodes)

1. Set hostname and Update the apt package index and install packages needed to use the Kubernetes apt repository:
```bash
sudo hostnamectl set-hostname <host-name>

sudo reboot
```

```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

2. Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
```bash
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

3. Add the appropriate Kubernetes apt repository.
```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

4. Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

5. Disable Swap: On all nodes (master + workers), run:
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab  # Persist after reboot
```

6. Enable Kernel Modules & Set Sysctl Params
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

- Then apply sysctl settings:
```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```
```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```
- Apply changes:
```bash
sudo sysctl --system
```

7. Install a Container Runtime (e.g., Containerd)
- Kubernetes needs a container runtime. Let’s install Containerd:
```bash
sudo apt-get install -y containerd
```
- Configure Containerd:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```
- Set SystemdCgroup = true in /etc/containerd/config.toml:
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

- Restart Containerd:
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## On Master 
8. Initialize the Kubernetes Control Plane (On Master Node Only). On this, you need to have decided the network plugin to be used. We will use Flannel network plugin.
- Run the following on your master node:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

```
- This may come with errors: Your system may not meet all the requirements, and as such, you might need to skip those checks: Incase the above throws check errors, run the following:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=SystemVerification
```
Note: the choice of the IP range given:
-- Calico: Requires a pod CIDR (e.g., 192.168.0.0/16).
-- Flannel: Default is 10.244.0.0/16, but configurable.
Once complete, set up kubectl for the current user:- Follow the prompts given on the output above. Ensure to copy the worker join command as well and execute it on all worker nodes 
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Note: SHould you face a challenge joining the nodes, due to system pre-checks, use the following: ie add on the given command: --ignore-preflight-errors=SystemVerification
```bash
sudo kubeadm join 192.168.100.31:6443 --token z63awy.4k2itinl3o6nrud7 --discovery-token-ca-cert-hash sha256:b1f0de577aed4d34d516e0023f7427e4ad9a959c3b0a3b4101b97031b6178e7d --ignore-preflight-errors=SystemVerification
```

Incase you miss out on the join command, run the following on master node to generate kubeadm join command:
```bash
kubeadm token create --print-join-command
```

9. Install a Network Plugin (CNI)
- You need a CNI plugin for pod communication. Based on the IP range used to initialize the cluster, you should install a plugin that aligns with that IP range.
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
- Allow sometime for the network plugin to install the necessary pods/resources
Monitor:
```bash
kubectl get pods -n kube-system
```



10. Create a simple deployment.
- Create a simple deployment and expose it to the internet.
```bash
kubectl create deployment dtechnologies --image=dannywangari/dtechnologies:2.0.1 --replicas=2

kubectl expose deployment dtechnologies --type=NodePort --port=80 --target-port=8000

kubectl get services   #get the port:: 

#use the workernode IP to access your application
http://<EC2-PUBLIC-IP>:31567
```



## Challenges. 
- Creating pods: Images may not successfully be pulled!
```bash
echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" | sudo tee /etc/resolv.conf

sudo systemctl restart systemd-resolved

sudo systemctl restart docker  OR sudo systemctl restart containerd

sudo systemctl restart kubelet
```


- Kubelete fails to start:: as swap is on.
--inspect:
```bash
journactl -u kubelet.service -f
```
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab  # Persist after reboot
```
- Then restart kubelet service

- Enable autocompletion
```bash
source <(kubectl completion bash) 
echo "source <(kubectl completion bash)" >> ~/.bashrc 
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -o default -F __start_kubectl k" >> ~/.bashrc
source  ~/.bashrc
```

- Unable to ssh to worker nodes from master nodes
    - Ensure to setup hostnames in the worker nodes:
    ```bash
    sudo hostnamectl set-hostname <node01>
    ```
    - Update /etc/hosts on the Master Node
    ```bash
    sudo nano /etc/hosts
    #then add the following: ie the IPs of your work node and the hostname
    192.168.100.29 node01
    ```
    - Upload SSH keys
    ```bash
    ssh-keygen -t rsa -b 4096
    #Press enter till you get final output
    #the copy the keys to the server
    ssh-copy-id osboxes@node01
    #from here you can access the worker nodes without the need for a password.
    ```

- To everytime enter the sudo password. 
```bash
sudo visudo

#If your current user is in the sudo group:
- check
groups user-name

#Update the line 
%sudo   ALL=(ALL:ALL) ALL to %sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```


