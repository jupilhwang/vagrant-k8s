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

  swapoff -a
  echo "http://dl-cdn.alpinelinux.org/alpine/edge/community" >> /etc/apk/repositories
  echo "http://dl-cdn.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories
  echo "br_netfilter" > /etc/modules-load.d/k8s.conf

  apk add cni-plugin-flannel \
    cni-plugins \
    flannel \
    flannel-contrib-cni \
    containerd \
    kubelet \
    kubeadm \
    kubectl \
    uuidgen \
    nfs-utils

  ln -sf /usr/libexec/cni/flannel-amd64 /usr/libexec/cni/flannel

  uuidgen > /etc/machine-id

  echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf
  echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
  sysctl -w net.bridge.bridge-nf-call-iptables=1
  sysctl -w net.ipv4.ip_forward=1

  rc-update add containerd
  rc-update add kubelet
  # rc-update add ntpd

  # /etc/init.d/ntpd start
  /etc/init.d/kubelet start
  /etc/init.d/containerd start

SCRIPTEND

kubeadm_master =<<-SCRIPTEND

  kubeadm init --pod-network-cidr=10.244.0.0/16 --node-name=master
  mkdir ~/.kube
  ln -s /etc/kubernetes/admin.conf /root/.kube/config
  kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

SCRIPTEND

Vagrant.configure("2") do |config|
 
  config.vm.box = "generic/ubuntu2204"
  # config.vm.box = "generic/alpine317"

  config.vm.box_check_update = true
  config.vm.synced_folder ".", "/vagrant", disabled: true

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
    master.vm.network "private_network", type: "dhcp"#, ip: "192.168.50.200", :bridge => "NAT"
    
    # node.vm.network "forwarded_port", guest: 22, host: 2730
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
      node.vm.network "private_network", type: "dhcp"
      # node.vm.network "private_network"
      #node.vm.network "private_network", ip: "192.168.50.20#{i}", :bridge => "NAT"
    end
  end

  config.vm.provision "Installing Packages", before: :each, type: "shell", inline: "#{bootstrap}", privileged: true
  config.vm.provision "shell", reboot: true
  # config.vm.provision "Swap off", type: "shell", inline: "sudo swapoff -a", run: "always"
  config.vm.provision "Good Bye", type: "shell", after: :all, inline: <<-SHELL
    echo "======================"
  SHELL

end