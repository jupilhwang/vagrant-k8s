# -*- mode: ruby -*-
# vi: set ft=ruby :

MASTER_CPU = 2
MASTER_MEMORY = 2048
WORKER_NODE_CPU = 4
WORKER_NODE_MEMORY = 4096

NUM_WORKDER_NODE = 1

PROVIDER = ENV['provider'] || "virtualbox"
VAGRANT_COMMAND = ARGV[0]

bootstrap =<<-SCRIPTEND
  apt update && apt -y upgrade
  swapoff -a
  sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

  echo "overlay" >> /etc/modules-load.d/containerd.conf
  echo "br_netfilter" >> /etc/modules-load.d/containerd.conf

  modprobe overlay
  modprobe br_netfilter

  echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.d/kubernetes.conf
  echo "net.bridge.bridge-nf-call-ip6tables=1" >> /etc/sysctl.d/kubernetes.conf
  echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/kubernetes.conf

  modprobe overlay
  modprobe br_netfilter

  sysctl --system

  apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
  curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/k8s.gpg
  # curl -s https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - 
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

  apt update
  apt install -y containerd.io kubelet kubeadm kubectl kubernetes-cni
  # apt-mark hold containerd kubelet kubeadm kubectl kubernetes-cni

  containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
  sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

  systemctl restart containerd
  systemctl enable containerd

  systemctl restart kubelet
  systemctl enable kubelet

SCRIPTEND

kubeadm_master =<<-SCRIPTEND

  kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint=192.168.56.5
  mkdir ~/.kube
  ln -s /etc/kubernetes/admin.conf /root/.kube/config
  chown $(id -u):$(id -g) $HOME/.kube/config

  kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
  # kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

SCRIPTEND

kubeadm_node =<<-SCRIPTEND

  bash /vagrant/k8s-join.sh

  mkdir $HOME/.kube
  cp /vagrant/kubeconfig $HOME/.kube/config
  chown $(id -u):$(id -g) $HOME/.kube/config

SCRIPTEND


Vagrant.configure("2") do |config|
 
  config.vm.box = "generic/ubuntu2204"
  # config.vm.box = "generic/alpine317"

  config.vm.box_check_update = true
  config.vm.synced_folder "./shared", "/vagrant", mount_options: ["dmode=775","fmode=777"]

  # config.vm.shell 

  config.vm.define "master" do |master|
    if PROVIDER == "virtualbox"
      master.vm.provider "virtualbox" do |vm, override|
        vm.name = "k8s-master"

        vm.customize ["modifyvm", :id, "--memory", MASTER_MEMORY]
        vm.customize ["modifyvm", :id, "--cpus", MASTER_CPU]
        vm.customize ["modifyvm", :id, "--vram", 32]
        vm.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        vm.customize ["modifyvm", :id, "--audio", "none"]
        vm.customize ["modifyvm", :id, "--clipboard", "bidirectional"]
        vm.customize ["modifyvm", :id, "--draganddrop", "hosttoguest"]
        vm.customize ["modifyvm", :id, "--usb", "off"]

        vm.check_guest_additions = true   # virtualbox guest additions
      end
    end

    if PROVIDER == "hyperv"
      master.vm.provider "hyperv" do |vm|
        vm.name = "k8s-master"
        vm.memory = MASTER_MEMORY
        vm.cpus = MASTER_CPU

        # hyper-v 
        vm.enable_virtualization_extensions = true
        vm.vm_integration_services = { 
          guest_service_interface: true,
          heartbeat: true,
          key_value_pair_exchange: true,
          shutdown: true,
          time_synchronized: true,
          vss: true
        }
      end
    end

    master.vm.hostname = "master"
    master.vm.network "public_network", bridge: ["eth0", "en0", "en0: Wi-Fi"]
    master.vm.network "private_network", ip: "192.168.56.5", hostname: true
    
    # node.vm.network "forwarded_port", guest: 22, host: 2730

    master.vm.provision "Installing K8s-master", type: "shell", inline: "#{kubeadm_master}", privileged: true

    master.vm.provision "Print Join Command", type: "shell", inline:<<-SCRIPT
      kubeadm token create --print-join-command > /vagrant/k8s-join.sh
      cp /root/.kube/config /vagrant/kubeconfig
    SCRIPT
  end 

  (1..NUM_WORKDER_NODE).each do |i|
    config.vm.define "node-#{i}" do |node|
      node.vm.provider "virtualbox" do |vm|
        vm.name = "k8s-node-#{i}"
        vm.memory = WORKER_NODE_MEMORY
        vm.cpus = WORKER_NODE_CPU
        # vm.vm_integration_services = {      # hyper-v 
        #   guest_service_interface: true
        # }
        vm.check_guest_additions = true       # virtualbox guest additions
      end

      node.vm.hostname = "node-#{i}"
      node.vm.network "public_network", bridge: ["eth0", "en0", "en0: Wi-Fi"]
      node.vm.network "private_network", ip: "192.168.56.1#{i}", hostname: true
      # node.vm.network "private_network"
      
      node.vm.provision "Installing K8s-node", type: "shell", inline: "#{kubeadm_node}", privileged: true
    end
  end

  config.vm.provision "Installing Packages", before: :each, type: "shell", inline: "#{bootstrap}", privileged: true
  config.vm.provision "shell", reboot: true

end