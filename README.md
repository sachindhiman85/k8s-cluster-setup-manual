##########################################################
# Kubernetes Cluster Setup using Kubeadm (Step-by-Step Guide with Containerd, HA, and Best Practices)
3 Masters + 2 Workers + HAProxy
----------------------------------------------------------
📌 Overview

This repository provides a step-by-step guide to build a Highly Available Kubernetes cluster using kubeadm, with a fully self-managed setup.
No cloud load balancer required — everything is configured manually for maximum control and real-world understanding.
----------------------------------------------------------
##########################################################

- Steps Overview
  
##########################################################
- Architecture
- Prerequisites
- Step 1: Node Preparation
- Step 2: Install Containerd
- Step 3: Install Kubernetes Components
- Step 4: Initialize Control Plane
- Step 5: Join Worker Nodes
- Step 6: Networking (Calico)
- Step 7: Verification
- Step 8: Troubleshooting

##########################################################

This guide helps you set up a Highly Available Kubernetes cluster from scratch, using:

- kubeadm for cluster setup and management
- containerd as the container runtime
- HAProxy for load balancing (no cloud load balancer required)
- Calico for networking and policy enforcement

#########################################################

- Architecture

#########################################################

<img width="1196" height="1315" alt="image" src="https://github.com/user-attachments/assets/990ef903-8ad4-49d6-bef2-29446bfbd0b9" />

#########################################################

- Prerequisites

#########################################################

Before starting, make sure your environment meets the following requirements:
- 🖥️ Infrastructure
  - Ubuntu 22.04 installed on all nodes
  - All nodes are in the same network (same VPC or subnet)
  - Stable network connectivity between all nodes
- 🔐 Network & Security
  - Security group or firewall rules allow internal communication
  - 172.xx.xx.0/16 → All traffic (required during setup phase)
  - Required ports are open between control plane and worker nodes
- 👤 Access & Permissions
  - Root or sudo access on all nodes
  - Passwordless SSH access recommended for smooth setup

#########################################################

- Step 1: Configure HAProxy (Load Balancer)

#########################################################

- Run on LB node (172.xx.xx.10)
  - sudo apt update
  - sudo apt install -y haproxy

- Edit config:
  - sudo nano /etc/haproxy/haproxy.cfg
  - Copy and Paste below lines:
----------------------------------------------------
	global
	    log /dev/log local0
	    log /dev/log local1 notice
	
	defaults
	    log global
	    mode tcp
	    option tcplog
	    timeout connect 10s
	    timeout client 1m
	    timeout server 1m
	
	frontend kubernetes
	    bind *:6443
	    default_backend kubernetes_masters
	
	backend kubernetes_masters
	    balance roundrobin
	    option tcp-check
	    server master1 172.xx.xx.11:6443 check
	    server master2 172.xx.xx.12:6443 check
	    server master3 172.xx.xx.13:6443 check
-----------------------------------------------------
- Start HAProxy:
  - sudo systemctl restart haproxy
  - sudo systemctl enable haproxy

- Verify:
  - ss -lntp | grep 6443
    
#########################################################

- Step 2: Prepare All Kubernetes Nodes

#########################################################

- Run on all masters and workers
  - Disable swap
    - sudo swapoff -a
    - sudo sed -i '/ swap / s/^/#/' /etc/fstab
- Kernel modules

			cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
			overlay
			br_netfilter
			EOF
- run below cmd 
    - sudo modprobe overlay
    - sudo modprobe br_netfilter
      
- update Sysctl settings
  
			cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
			net.bridge.bridge-nf-call-iptables = 1
			net.ipv4.ip_forward = 1
			EOF
- run below cmd 
	 - sudo sysctl --system
    
#########################################################

- Step 3: Install containerd - ALL vms
  
#########################################################
- run below cmd
  - sudo apt update
  - sudo apt install -y containerd
  - sudo mkdir -p /etc/containerd
  - containerd config default | sudo tee /etc/containerd/config.toml
  - sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
  - sudo systemctl restart containerd
  - sudo systemctl enable containerd

#########################################################

- Step 4: Install Kubernetes tools -  ALL vms

#########################################################

- sudo apt update
-	sudo apt install -y apt-transport-https ca-certificates curl gpg
	
-	sudo mkdir -p /etc/apt/keyrings
-	curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
	
-	echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
	
-	sudo apt update
-	sudo apt install -y kubelet kubeadm kubectl
-	sudo apt-mark hold kubelet kubeadm kubectl

#########################################################

- Step 5: Initialize (master1)

#########################################################

- Run on master1 (172.xx.xx.11)
---------------------------------------------------------
	cat <<EOF | sudo tee /root/kubeadm-ha.yaml
	apiVersion: kubeadm.k8s.io/v1beta4
	kind: InitConfiguration
	nodeRegistration:
	  criSocket: unix:///run/containerd/containerd.sock
	localAPIEndpoint:
	  advertiseAddress: 172.xx.xx.11
	---
	apiVersion: kubeadm.k8s.io/v1beta4
	kind: ClusterConfiguration
	kubernetesVersion: v1.35.0
	controlPlaneEndpoint: "172.xx.xx.10:6443"
	networking:
	  podSubnet: "192.168.0.0/16"
	apiServer:
	  certSANs:
	    - "172.xx.xx.10"
	    - "172.xx.xx.11"
	    - "172.xx.xx.12"
	    - "172.xx.xx.13"
	EOF
----------------------------------------------------------

- Run below cmd:
	- sudo kubeadm init --config /root/kubeadm-ha.yaml --upload-certs

- Note :
  - copy the output. Values will be used when other master and worker will join the cluster.
  ---------------------------------------
	certSANs:
    - "172.xx.xx.10"
    - "172.xx.xx.11"
    - "172.xx.xx.12"
    - "172.xx.xx.13"
      
    These are masternode ips and lb ip.
----------------------------------------
#########################################################

- Step 6: Configure kubectl  (master1)

#########################################################

 - mkdir -p $HOME/.kube
 - sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
 - sudo chown $(id -u):$(id -g) $HOME/.kube/config

#########################################################

- Step 7: Install Calico Network (master1)

#########################################################

-Run on master1 (172.xx.xx.11)
 - kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/operator-crds.yaml
 - kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml

---------------------------------------------------------
	cat <<EOF | kubectl apply -f -
	apiVersion: operator.tigera.io/v1
	kind: Installation
	metadata:
	  name: default
	spec:
	  calicoNetwork:
	    ipPools:
	    - cidr: 192.168.0.0/16
	EOF

---------------------------------------------------------
- Wait until:
  - kubectl get nodes

#########################################################

- Step 8: Join Other Masters  (master1)

#########################################################
- Join both the master to control plan 
  
		kubeadm join 172.xx.xx.10:6443 --token <token> \
		--discovery-token-ca-cert-hash sha256:<hash> \
		--control-plane --certificate-key <key>

- Note:
  - How to Get Join Command (Token, Hash, Certificate Key)
  - When you run kubeadm init, Kubernetes automatically prints the join commands.
  - Important: Save this output immediately.

#########################################################

- Step 9: Join Workers  (master1)

#########################################################
- Join the worker to control plan
  
		kubeadm join 172.xx.xx.10:6443 --token <token> \
		--discovery-token-ca-cert-hash sha256:<hash>

#########################################################

- Step 10: Verify Cluster

#########################################################

- kubectl get nodes -o wide
- kubectl get pods -A

#########################################################

🎯 Final Result

#########################################################

After completing this setup, you will have:

- 3 control plane nodes for high availability
- 2 worker nodes running workloads
- HAProxy as load balancer (no cloud dependency)
- Calico networking for pod communication
- Fully functional Highly Available Kubernetes cluster

########################################################

- Pro Tips

########################################################
- Always use private IPs
- Allow internal CIDR during setup
- Join masters within 2 hours
- Save join command
