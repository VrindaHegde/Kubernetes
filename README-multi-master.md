# Kubernetes multi master cluster set up

## Six node cluster set up

### Prerequisites
1. One node as the loadbalancer.
2. Two master nodes.
3. Three worker nodes.
4. ssh access from loadbalancer node to all machines (master & worker).

## Setting up loadbalancer
We will use HAPROXY as the primary loadbalancer.
We have 2 master nodes. Which means the user can connect to either of the 2 api-servers. The loadbalancer will be used to loadbalance between the 2 api-servers.

- Login to the loadbalancer node

- Switch as root: sudo su

- Update your repository and your system

>   sudo apt-get update && sudo apt-get upgrade -y

- Install haproxy

>   sudo apt-get install haproxy -y

- Edit haproxy configuration

>   nano /etc/haproxy/haproxy.cfg

- Add the below lines to create a frontend configuration for loadbalancer:

```bash
    frontend fe-apiserver
   bind 0.0.0.0:6443
   mode tcp
   option tcplog
   default_backend be-apiserver
```

- Add the below lines to create a backend configuration for master1 and master2 nodes at port 6443. Note : 6443 is the default port of kube-apiserver:

```bash
    backend be-apiserver
   mode tcp
   option tcplog
   option tcp-check
   balance roundrobin
   default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100

       server etcd-master1 10.138.0.15:6443 check
       server etcd-master2 10.138.0.16:6443 check
```

Here - etcd-master1 and etcd-master2 are the names of the master nodes and 10.138.0.15 and 10.138.0.16 are the corresponding internal IP addresses.

- Restart and Verify haproxy:
```bash
    systemctl restart haproxy
    systemctl status haproxy
```
Ensure haproxy is in running status.

- Run nc command on loadbalancer node as below:
>   nc -v localhost 6443

It should give the output as: Connection to localhost 6443 port [tcp/*] succeeded!
<br></br>

## Install kubeadm,kubelet and docker on master and worker nodes

In this step we will install kubelet and kubeadm on the below nodes
- master1
- master2
- worker1
- worker2
- worker3
<br></br>

### The below steps will be performed on all the five nodes:
- switch to root user on all the five nodes.

- update the repositories:
>   apt-get update

- turn off swap:
>   swapoff -a 

- Install Kubeadm and Kubelet
```bash
    apt-get update && apt-get install -y apt-transport-https curl

    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF

    apt-get update

    apt-get install -y kubelet kubeadm

    apt-mark hold kubelet kubeadm 

```

- Install container runtime - docker
```bash
    sudo apt-get update

    sudo apt-get install -y \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg-agent \
        software-properties-common

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

    sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"

    sudo apt-get update

    sudo apt-get install docker-ce docker-ce-cli containerd.io -y

```
## Configure kubeadm to bootstrap the cluster

- Log in to master1
- Switch to root account - sudo su
- Execute the below command to initialize the cluster:
```bash
    kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs --pod-network-cidr=192.168.0.0/16 
```

Here, LOAD_BALANCER_DNS is the IP address or the DNS name of the loadbalancer. I will use the dns name of the server, i.e. loadbalancer as the LOAD_BALANCER_DNS. In case your DNS name is not resolvable across your network, you can use the IP address for the same.

The LOAD_BALANCER_PORT is the front end configuration port defined in HAPROXY configuration, we have kept the port as 6443.

The output will look like:
```bash
    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    You can now join any number of the control-plane node running the following command on each as root:

    kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
        --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 \
        --control-plane --certificate-key 824d9a0e173a810416b4bca7038fb33b616108c17abcbc5eaef8651f11e3d146

    Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
    As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use 
    "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
        --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 
```

### From the above output we will execute commands to join the nodes to the cluster

- Login to master2 node as root user

- Setup new control plane (master) using the first join token on master2 node:
```bash
    kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 \
    --control-plane --certificate-key 824d9a0e173a810416b4bca7038fb33b616108c17abcbc5eaef8651f11e3d146

```
- Login to worker nodes as root user

- Join worker node using the second join token on all the worker nodes:
```bash
    kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 
```
**NOTE**<br></br>
Your output will be different than what is provided here. While performing the rest of the demo, ensure that you are executing the command provided by your output and dont copy and paste from here.

## Configure kubeconfig on loadbalancer node

Now that we have configured the master and the worker nodes, its now time to configure Kubeconfig (.kube) on the loadbalancer node. It is completely up to you if you want to use the loadbalancer node to setup kubeconfig. kubeconfig can also be setup externally on a separate machine which has access to loadbalancer node. For the purpose of this demo we will use loadbalancer node to host kubeconfig and kubectl.

- Log in to loadbalancer node
- Switch to root - sudo su
- Create a directory - .kube at $HOME of root
>   mkdir -p $HOME/.kube

- SCP configuration file from any one master node to loadbalancer node.
>   cd .ssh
>   scp pub_key etcd-master2:/etc/kubernetes/admin.conf $HOME/.kube/config

- Provide appropriate ownership to the copied file on the loadbalancer node.
>   chown $(id -u):$(id -g) $HOME/.kube/config

- Install kubectl binary on loadbalancer node:
>   snap install kubectl --classic

- Verify the cluster:
>   kubectl get nodes

### Install CNI and complete installation on loadbalancer node

```bash
    kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
    kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```
This installs CNI component to your cluster.

***Verifying the HA cluster***
>   kubectl get nodes 