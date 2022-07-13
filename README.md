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

    vi /usr/lib/systemd/system/docker.service
    
    ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd

    sudo systemctl daemon-reload
    sudo systemctl restart docker
    sudo docker run hello-world

You should now have a file /etc/containerd/config.toml

/etc/containerd/config.toml changes

remove cri from disabled plugins:


    disabled_plugins = [""]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true

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

Set SELinux in permissive mode (effectively disabling it)
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config


    sudo systemctl stop firewalld
    sudo systemctl disable firewalld

    sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

    sudo systemctl enable --now kubelet


disable swap from fstab
    sudo swapoff -a

now kubelet will be restarting because kubeadm hasn't told it what to do. Need to configure kubelet to use approproate cgroup (systems)
 but default cgroup is systemd ACCORDING TO BELOW **




# 3 - Create cluster with kubeadm

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/


    kubeadm  init --pod-network-cidr=192.168.78.0/16 


kubeadm will set up the cluster, and finish with some instructions to setup a pod network and how to join worker nodes to the cluster


*
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join .......:6443 --token ......... \
        --discovery-token-ca-cert-hash sha256:..........
*


The initial token has 24 hr ttl. To generate a new token "kubadm token create --print-join-command"


 The coredns pods will show as pending on a "kubectl get pods -n kube-system -o wide " , until a network has been installed,
 covered in step 4 

# 4 - Apply network model


 This howto uses calico
https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises

curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
    kubectl apply -f calico.yaml







#
