# -*- mode: ruby -*-
# vi: set ft=ruby :

# taken and improved from: https://github.com/ecomm-integration-ballerina/kubernetes-cluster/blob/master/Vagrantfile

KUBE_VER   = "1.15.6"

servers = [
    {
        :name => "k8s-head",
        :type => "master",
        :box => "hashicorp/bionic64",
        :box_version => "1.0.282",
        :eth1 => "192.168.205.10",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-1",
        :type => "node",
        :box => "hashicorp/bionic64",
        :box_version => "1.0.282",
        :eth1 => "192.168.205.11",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-2",
        :type => "node",
        :box => "hashicorp/bionic64",
        :box_version => "1.0.282",
        :eth1 => "192.168.205.12",
        :mem => "2048",
        :cpu => "2"
    }
]

# This script to install k8s using kubeadm will get executed after a box is provisioned
$configureBox = <<-SCRIPT
    #!/bin/bash    
    set -xe

    export DEBIAN_FRONTEND=noninteractive

    echo "install docker"
    # reason for not using docker provision is that it always installs latest version of the docker, but kubeadm requires 17.03 or older
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 18.09 | head -1 | awk '{print $3}')

    # run docker commands as vagrant user (sudo not required)
    usermod -aG docker vagrant

    echo "install kubeadm and co."
    apt-get update && sudo apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet=#{KUBE_VER}-00 kubeadm=#{KUBE_VER}-00 kubectl=#{KUBE_VER}-00


    # kubelet requires swap off
    swapoff -a

    # keep swap off after reboot
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

    # ip of this box
    IP_ADDR=`ifconfig eth1 | grep "inet " | awk '{print $2}'| cut -f2 -d:`
    
    # set node-ip
    sudo sh -c 'echo KUBELET_EXTRA_ARGS= >> /etc/default/kubelet'
    sudo sed -i "/^[^#]*KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR" /etc/default/kubelet
    sudo systemctl restart kubelet
SCRIPT

$configureMaster = <<-SCRIPT
    #!/bin/bash    
    set -xe
    
    echo "This is master"
    # ip of this box
    IP_ADDR=`ifconfig eth1 | grep "inet " | awk '{print $2}'| cut -f2 -d:`
    
    # install k8s master
    HOST_NAME=$(hostname -s)
    kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16

    #copying credentials to regular user - vagrant
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

    echo "install Calico pod network addon"
    # ref https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
    export KUBECONFIG=/etc/kubernetes/admin.conf
    kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml

    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh

    # required for setting up password less ssh between guest VMs
    sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    sudo service sshd restart

SCRIPT

$configureNode = <<-SCRIPT
    echo "This is worker"
    export DEBIAN_FRONTEND=noninteractive
    apt-get install -y sshpass
    sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.205.10:/etc/kubeadm_join_cmd.sh .
    sh ./kubeadm_join_cmd.sh
SCRIPT

Vagrant.configure("2") do |config|

    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]

            config.vm.provider "virtualbox" do |v|

                v.name = opts[:name]
            	 v.customize ["modifyvm", :id, "--groups", "/HA Kubernetes"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]

            end

            # we cannot use this because we can't install the docker version we want - https://github.com/hashicorp/vagrant/issues/4871
            #config.vm.provision "docker"

            config.vm.provision "shell", inline: $configureBox

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
            else
                config.vm.provision "shell", inline: $configureNode
            end
        end
    end
end
