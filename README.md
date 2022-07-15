# Kubernetes 1.24 install on CentOS7 using containerd

# 1 - containerd install



https://kubernetes.io/docs/setup/_print/#containerd

getting started with containerd-
https://github.com/containerd/containerd/blob/main/docs/getting-started.md
centos - https://docs.docker.com/engine/install/centos/

steps - install latest docker engine,containerd docker compose

    sudo yum install -y yum-utils
    sudo yum-config-manager \
        --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
    sudo systemctl start docker


## setup containerd for kubernetes (CRI)
https://kubernetes.io/docs/setup/production-environment/container-runtimes/

Forwarding IPV4/ lets iptables see bridged traffic


    lsmod | grep br_netfilter
    sudo modprobe br_netfilter

    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter

 sysctl params required by setup, params persist across reboots
 
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

Apply sysctl params without reboot

    sudo sysctl --system

change cgroup docker driver to systemd

    sudo vi /usr/lib/systemd/system/docker.service
    
    ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd

    sudo systemctl daemon-reload
    sudo systemctl restart docker
    sudo docker run hello-world

You should now have a file /etc/containerd/config.toml

## /etc/containerd/config.toml changes

remove cri from disabled plugins, and add configuration for Systemd Cgroup


    disabled_plugins = [""]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true

Restart containerd

    sudo systemctl restart containerd




# 2 - Install kubeadm

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/



    cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
    enabled=1
    gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    exclude=kubelet kubeadm kubectl
    EOF

Set SELinux in permissive mode 

    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

Disable Firewalld

    sudo systemctl stop firewalld
    sudo systemctl disable firewalld

disable swap from fstab

    sudo vi /etc/fstab
    
Switch off any running swap

    sudo swapoff -a

Install KUBE software

    sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
    sudo systemctl enable --now kubelet

Note that kubelet will be restarting at this point because kubeadm hasn't told it what to do. 





# 3 - Create cluster with kubeadm

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/


    sudo kubeadm  init --pod-network-cidr=192.168.78.0/16 

> *At version 1.24 the default Cgroup is systemd, so there is no need to specify this here.*

kubeadm will set up the cluster, and finish with some instructions to setup a pod network and how to join worker nodes to the cluster:





> You should now deploy a pod network to the cluster.
> Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
>   https://kubernetes.io/docs/concepts/cluster-administration/addons/
> 
> Then you can join any number of worker nodes by running the following on each as root:
> 
>       kubeadm join .......:6443 --token ......... \
>               --discovery-token-ca-cert-hash sha256:..........


Copy this information to a notepad, you will be needing the kubeadm join command iin a short while to join the worker nodes to the master.


The initial token has 24 hr ttl. To generate a new token "kubadm token create --print-join-command"


 The coredns pods will show as pending on a "kubectl get pods -n kube-system -o wide " , until a network has been installed,
 covered in step 4 
 
 It is important at this stage to set up your login user's access to this cluster

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

Now you can run kubectl commmand as your user

    kubectl get nodes

# 4 - Apply network model


This howto uses calico
https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises

    curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
    kubectl apply -f calico.yaml


# 5 - Join worker nodes

On each worker node run the advertised join command to join the cluster

    sudo kubeadm join .......:6443 --token ......... --discovery-token-ca-cert-hash sha256:..........


Now check the cluster status by running  on the control plane, master node. Nodes should be in ready state

    kubectl get nodes
    
    NAME        STATUS   ROLES           AGE   VERSION
    k2master    Ready    control-plane   83m   v1.24.2
    k2minion1   Ready    <none>          61m   v1.24.2

If they aren't ready check pod state in kube-system and troubleshoot

    kubectl get pods -n kube-system -o wide
    NAME                                       READY   STATUS    RESTARTS   AGE   IP               NODE        NOMINATED NODE   READINESS GATES
    calico-kube-controllers-6766647d54-g9kkb   1/1     Running   0          49m   192.168.11.1     k2minion1   <none>           <none>
    calico-node-hw5ch                          1/1     Running   0          49m   10.7.3.243       k2minion1   <none>           <none>
    calico-node-rr7bl                          1/1     Running   0          49m   10.7.3.105       k2master    <none>           <none>
    coredns-6d4b75cb6d-jshsv                   1/1     Running   0          84m   192.168.27.130   k2master    <none>           <none>
    coredns-6d4b75cb6d-tzwvn                   1/1     Running   0          84m   192.168.27.129   k2master    <none>           <none>
    etcd-k2master                              1/1     Running   0          85m   10.7.3.105       k2master    <none>           <none>
    kube-apiserver-k2master                    1/1     Running   0          85m   10.7.3.105       k2master    <none>           <none>
    kube-controller-manager-k2master           1/1     Running   0          85m   10.7.3.105       k2master    <none>           <none>
    kube-proxy-lg57k                           1/1     Running   0          63m   10.7.3.243       k2minion1   <none>           <none>
    kube-proxy-mt7q5                           1/1     Running   0          84m   10.7.3.105       k2master    <none>           <none>
    kube-scheduler-k2master                    1/1     Running   0          85m   10.7.3.105       k2master    <none>           <none>

## Debugging command tips

    journalctl -u kubelet

    journalctl -u containerd

    crictl --runtime-endpoint /run/containerd/containerd.sock ps

    kubectl describe node <master_node_name>
