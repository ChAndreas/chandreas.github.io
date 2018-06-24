---
layout: post
title: " Introduction to Kubernetes Cluster"
date: Sun May 8 17:31:51 EEST 2018
---
**Kubernetes**

Kubernetes is an open-source system for automating deployment, scaling, and management of containerized applications.

**Installation**

	Distro: Ubuntu 16.04
	Servers: Master
		Node Server 1
		Node Server 2
			
**Disable the swap file on all servers**

	sudo swapoff -a
	
Remove any matching reference found in /etc/fstab

**Install the Docker**

	apt-get update
	
	apt-get install -y apt-transport-https ca-certificates curl software-properties-common
	
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
	
	add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
	
	apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')

**Install Kubernetes packages**

You will install these packages on all of your machines:

kubeadm: the command to bootstrap the cluster.
kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
kubectl: the command line util to talk to your cluster.

	apt-get update && apt-get install -y apt-transport-https curl
	
	curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
	
	cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
	deb http://apt.kubernetes.io/ kubernetes-xenial main
	EOF	
	
	apt-get update
	
	apt-get install -y kubelet kubeadm kubectl kubernetes-cni

**Create the cluster**

We need to create the cluster by initiating the master with kubeadm. Only do this on the master node.

The Kubernetes master is responsible for maintaining the desired state for your cluster.

	sudo kubeadm init --pod-network-cidr=172.16.0.0/16
	
	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
	
Login to the other two nodes which are the worker nodes and use the token to join them to the master node.
The worker nodes in a cluster are the machines (VMs, physical servers, etc) that run your applications and cloud workflows.

	kubeadm join <ip>:6443 --token <token key> --discovery-token-ca-cert-hash sha256:<key>

**Install networking**

Only on master execute the following
	
Weave NET

	kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
	curl -SL "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=172.16.0.0/16" | kubectl apply -f -
	
**Creating Pods, Deployments and Services**

**Pod**

A Kubernetes Pod represents a running process on your cluster. A pod is a group of one or more containers, with shared storage/network, and a specification for how to run the containers.

Example
Create a pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:

  restartPolicy: Always

  volumes:
  - name: shared-data
    emptyDir: {}

  containers:

  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
    mountPath: /usr/share/nginx/html
```		
		
	kubectl create -f pod.yaml
	
**Deployment**

A kubernetes Deployment manages creating Pods by means of ReplicaSets. 

ReplicaSet is the next-generation Replication Controller.A ReplicaSet ensures that a specified number of pod replicas are running at any given time.
	
create a deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```					
	
	kubectl create -f deployment.yaml
	

Replicas can be changed either by editing the deployment or using scale.	
	
	kubectl edit deployment nginx
		
	kubectl scale --replicas=3 deployment/nginx


**Service**
				
A Kubernetes Service is an abstraction which defines a logical set of Pods and a policy by which to access them. . The set of Pods targeted by a Service is determined by a Label Selector 

create service.yaml file

```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```	

	kubectl create -f service.yaml
	
	
Sources

	https://kubernetes.io/docs/
