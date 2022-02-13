# Kubernetes single master setup

## Three node cluster setup

### Prerequisites:
1. Launch three EC2 instances, one EC2 instance with 2 cpus (t2.medium) and the other two t2.micro.

1. Enable inbound rules for all traffic in security groups for all the three instances.

### Execute the below steps on all the three nodes (master,worker1,worker2)

>

```bash
    # update packages and their version
    apt-get update && apt-get upgrade -y

    # install curl and apt-transport-https
    sudo apt-get update && sudo apt-get install -y apt-transport-https curl

    # add key to verify releases
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

    # add kubernetes apt repo
    cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF

    # install kubelet, kubeadm and kubectl
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl

    # install docker
    apt-get install docker.io

    # apt-mark hold is used so that these packages will not be updated/removed automatically
    sudo apt-mark hold kubelet kubeadm kubectl

```
### After the above commands are successfully run on all the worker nodes. Below steps can be followed to initialize the Kubernetes cluster.<br/><br/>
## <u>On Master Node</u>
<br/><br/>
### Execute the below command on the master node: <br></br>
```bash
    # Replace MASTER_IP with the correct master node IP
    kubeadm init --apiserver-advertise-address=${MASTER_IP} --pod-network-cidr=10.244.0.0/16

```

## Troubleshooting tips

    After executing the kubeadm init command if you get the error as 'The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused'
    then perform the below steps.

### Configure docker's cgroup driver and execute the below commands:
    There are chances that the kubeadm init command is going to fail saying the different cgroup drvers are being used in the kubelet (systemd) and docker (cgroupfs) service. To resolve that we will make sure that both the services are running with the same cgroup driver. It's recommended that we use systemd as cgroup driver for both of the services. To restart docker with systemd as cgroup driver, change the service file (/lib/systemd/system/docker.service) for docker to accept cgroup driver

``` bash
    # To undo the kubeadm init configuration
    kubeadm reset

    # Edit the docker.service file
    ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd

    # Execute the below command to reload the daemon and restart docker service
    systemctl daemon-reload

    systemctl restart docker.service

    systemctl status docker.service

```
### After executing kubeadm init command perform the below steps on master node:
```bash
    # Execute the command on master node
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    # Execute the join command on worker nodes that was provided by kubeadm init command on master node
    kubeadm join 172.31.16.47:6443 --token ilylse.tkio183s13h2sm3n \
    --discovery-token-ca-cert-hash sha256:6854d5315aedc9b5724cef133bfa0ee8acc0b70949ddde3b54833726c737c43f

    # Exeute the below command on master node
    export KUBECONFIG=/etc/kubernetes/admin.conf

    # Check the namespaces on master node
    kubectl get namespaces
    
    # To check the status of the running pods on master node
    kubectl get pods --all-namespaces

    # Execute the below command to start the kube-flannel pods on master node to install CNI plugin
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
