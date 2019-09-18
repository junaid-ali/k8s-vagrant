# -*- mode: ruby -*-
# vi: set ft=ruby :

### Requirements
# Vagrant version: 2.2+
# vagrant-libvirt (0.0.45, global)

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.

require 'yaml'
settings = YAML.load_file 'cluster.yaml'

numWorkers = settings['workers']['numOfWorkers']

k8s_nodes = [
  {
    :name => "k8s-master",
    :type => "master",
    :cpu => settings['master']['resources']['cpu'].to_s,
    :memory => settings['master']['resources']['memory'].to_s
  }
]


for worker in 1..numWorkers do
  k8s_nodes.push({
    :name => "k8s-worker" + worker.to_s,
    :type => "worker",
    :cpu => settings['workers']['resources']['cpu'].to_s,
    :memory => settings['workers']['resources']['memory'].to_s
  })
end

$setupK8sNodes = <<-SCRIPT
  sed -i.orig 's/us.archive/ie.archive/g' /etc/apt/sources.list
  apt-get update
  echo "############################################"
  echo "  Installing Docker 18.06.2~ce~3-0~ubuntu"
  echo "############################################"
  apt-get install -y apt-transport-https ca-certificates curl software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
  add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
  apt-get update && apt-get install docker-ce=18.06.2~ce~3-0~ubuntu -y
  cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
  sudo usermod -aG docker vagrant
  systemctl daemon-reload
  systemctl restart docker

  echo "###############################"
  echo "  Installing Kubeadm 1.13.10"
  echo "###############################"
  apt-get update && apt-get install -y apt-transport-https curl
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
  apt-get update
  apt-get install -y kubelet=1.13.10-00 kubeadm=1.13.10-00 kubectl=1.13.10-00
  apt-mark hold kubelet kubeadm kubectl

  # kubelet requires swap off
  swapoff -a
  sudo sed -i.orig '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

  echo "Dependencies installed and configured for kubernetes"
SCRIPT

$setupK8sMaster = <<-SCRIPT
  set -x
  sudo kubeadm init --pod-network-cidr=10.132.0.0/16

  su vagrant -c 'mkdir -p $HOME/.kube; \
                 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config; \
                 sudo chown $(id -u):$(id -g) $HOME/.kube/config; \
                 # Sleeping for 60secs for all Kubernetes services to come up
                 sleep 60; \
                 kubectl -n kube-system get po; \
                 kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml; \
                 wget https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml; \
                 sed -i~orig 's/192.168.0.0\/16/10.132.0.0\/16/g' calico.yaml; \
                 kubectl apply -f calico.yaml'
SCRIPT

Vagrant.configure("2") do |config|

  k8s_nodes.each do |node|

    config.vm.define node[:name] do |config|
  
      config.vm.box = settings['box']
      #config.vm.box = "ubuntu/bionic64"
      config.vm.hostname = node[:name]
      config.vm.provider "libvirt" || "virtualbox" do |vm|
      #config.vm.provider "virtualbox" do |vm|
        # Customize the amount of memory on the VM:
        vm.cpus = node[:cpu]
        vm.memory = node[:memory]
      end
  
      config.vm.provision "shell", inline: $setupK8sNodes
  
      if node[:type] == "master"
          config.vm.provision "shell", inline: $setupK8sMaster
      else
        config.vm.provision "shell", inline: <<-SHELL
         echo "It's a worker"
        SHELL
      end

      #if node[:name] == "k8s-worker"+numWorkers.to_s
      #if node[:name] == "k8s-master"
      #    config.trigger.after [:up, :provision] do |trigger|
      #      trigger.info = "Join worker nodes to the cluster"
      #      trigger.run = {path: "./join-workers.sh", args: "#{numWorkers}"}
      #    end
      #end
    end
  end
end
