# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'getoptlong'

opts = GetoptLong.new(
  ['--provider', GetoptLong::OPTIONAL_ARGUMENT],
)
PROVIDER="virtualbox"
begin
  opts.each do |opt, arg|
    case opt
      when '--provider'
        PROVIDER=arg
    end
  end
  rescue
end

MASTER_CPU = 2
MASTER_MEMORY = 2048
WORKER_NODE_CPU = 4
WORKER_NODE_MEMORY = 4096

NUM_WORKDER_NODE = 1

VAGRANT_COMMAND = ARGV[0]

bootstrap =<<-SCRIPTEND
  chpasswd vagrant vagrant
  swapoff -a
  sed -i '2s/^/#/' /etc/fstab
  # sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
  ufw disable

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

  sed -i -e 's/#DNS=/DNS=8.8.8.8/' /etc/systemd/resolved.conf
  systemctl restart systemd-resolved

  apt install -y curl gnupg2 software-properties-common ca-certificates  >/dev/null 2>&1

  mkdir -p /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

  curl -fsSLo /etc/apt/keyrings/kubernetes.gpg  https://packages.cloud.google.com/apt/doc/apt-key.gpg
  echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list >/dev/null

  apt update
  #apt -y upgrade >/dev/null 2>&1
  apt install -y containerd.io kubelet kubeadm kubectl kubernetes-cni 
  # apt-mark hold containerd kubelet kubeadm kubectl kubernetes-cni

  containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1 
  sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml >/dev/null 2>&1

  systemctl enable containerd
  systemctl restart containerd

  systemctl enable kubelet
  systemctl restart kubelet

SCRIPTEND

kubeadm_master =<<-SCRIPTEND

  kubeadm init --pod-network-cidr=10.10.0.0/16  --control-plane-endpoint=192.168.56.5 --apiserver-advertise-address=192.168.56.5 --ignore-preflight-errors=all 

  mkdir ~/.kube
  ln -s /etc/kubernetes/admin.conf /root/.kube/config
  chown $(id -u):$(id -g) $HOME/.kube/config

  echo "source <(kubectl completion bash)" >> $HOME/.bashrc
  echo "alias k=kubectl" >> $HOME/.bashrc
  echo "source <(kubectl completion bash | sed 's/kubectl/k/g')" >> $HOME/.bashrc

  kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
  #kubectl apply -f https://docs.projectcalico.org/manifests/custom-resources.yaml

  wget https://docs.projectcalico.org/manifests/custom-resources.yaml >/dev/null 2>&1
  sed -i "s/192.168.0.0/10.10.0.0/g" custom-resources.yaml
  kubectl apply -f custom-resources.yaml
  mv custom-resources.yaml /tmp

  sleep 100
  kubectl get pods -n calico-system
  kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-
  kubectl get nodes -o wide

  wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml >/dev/null 2>&1
  sed -i '/        - --metric-resolution=15s$/a ________- --kubelet-insecure-tls' components.yaml
  sed -i 's/________/        /g' components.yaml

  sed -i '/    spec:$/a ______hostNetwork: true' components.yaml
  sed -i 's/______/      /g' components.yaml

  kubectl apply -f components.yaml
  mv components.yaml /tmp >/dev/null

SCRIPTEND

kubeadm_node =<<-SCRIPTEND

  bash /vagrant/k8s-join.sh

  mkdir $HOME/.kube
  cp /vagrant/kubeconfig $HOME/.kube/config
  chown $(id -u):$(id -g) $HOME/.kube/config

  echo "source <(kubectl completion bash)" >> $HOME/.bashrc
  echo "alias k=kubectl" >> $HOME/.bashrc
  echo "source <(kubectl completion bash | sed 's/kubectl/k/g')" >> $HOME/.bashrc

  kubectl get pods -A
  kubectl get nodes -o wide

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
        # vm.customize ["modifyvm", :id, "--nic2", "natnetwork", "--nat-network2", "NatNetwork"]
        # vm.customize ["modifyvm", :id, "--nic3", "hostonly", "--hostonlyadapter3", "Host-Only Network"]

        # vm.check_guest_additions = true   # virtualbox guest additions
      end

      master.vm.network "private_network", ip: "192.168.56.5", hostname: true, virtualbox__intnet: true
    end

    if PROVIDER == "hyperv"
      master.vm.provider "hyperv" do |vm|
        vm.vmname = "k8s-master"
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

      master.vm.network "public_network", ip: "192.168.56.5", hostname: true, bridge: "InternalSwitch"
    end

    master.vm.hostname = "master"

    # config.vm.provision "shell", inline:"sed -i 's/00.00/10.0.0.5/g' /etc/netplan/01-netcfg.yaml"
    # config.vm.provision "shell", reboot: true
    
    # master.vm.network "private_network", type: "dhcp"
    # node.vm.network "forwarded_port", guest: 22, host: 2730

    master.vm.provision "Installing K8s-master", type: "shell", inline: "#{kubeadm_master}", privileged: true

    master.vm.provision "Print Join Command", type: "shell", inline:<<-SCRIPT
      kubeadm token create --print-join-command > /vagrant/k8s-join.sh
      cp /root/.kube/config /vagrant/kubeconfig
    SCRIPT
  end 

  (1..NUM_WORKDER_NODE).each do |i|
    config.vm.define "node-#{i}" do |node|
   
      if PROVIDER == "virtualbox"
        node.vm.provider "virtualbox" do |vm, override|
          vm.name = "k8s-node-#{i}"

          vm.customize ["modifyvm", :id, "--memory", WORKER_NODE_MEMORY]
          vm.customize ["modifyvm", :id, "--cpus", WORKER_NODE_CPU]
          vm.customize ["modifyvm", :id, "--vram", 32]
          vm.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
          vm.customize ["modifyvm", :id, "--audio", "none"]
          vm.customize ["modifyvm", :id, "--clipboard", "bidirectional"]
          vm.customize ["modifyvm", :id, "--draganddrop", "hosttoguest"]
          vm.customize ["modifyvm", :id, "--usb", "off"]

          # vm.check_guest_additions = true       # virtualbox guest additions
        end

        node.vm.network "private_network", ip: "192.168.56.1#{i}", hostname: true, virtualbox__intnet: true
      end

      if PROVIDER == "hyperv"
        node.vm.provider "hyperv" do |vm|
          vm.vmname = "k8s-node-#{i}"

          vm.memory = WORKER_NODE_MEMORY
          vm.cpus = WORKER_NODE_CPU

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

        node.vm.network "public_network", ip: "192.168.56.1#{i}", hostname: true, bridge: "InternalSwithch"
      end

      node.vm.hostname = "node-#{i}"
      # node.vm.network "public_network", bridge: ["eth0", "en0", "en0: Wi-Fi"]
      # node.vm.network "private_network"
   
      # config.vm.provision "shell", inline:"sed -i 's/00.00/10.0.0.1#{i}/g' /etc/netplan/01-netcfg.yaml"
      # config.vm.provision "shell", reboot: true
      node.vm.provision "Installing K8s-node", type: "shell", inline: "#{kubeadm_node}", privileged: true
    end
  end

  config.vm.provision "Installing Packages", before: :each, type: "shell", inline: "#{bootstrap}", privileged: true
  # config.vm.provision "shell", reboot: true
  config.vm.provision "shell", inline:"sudo swapoff -a", run: "always"

end