$script = <<-SCRIPT
export ENV APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=DontWarn
export DEBIAN_FRONTEND=noninteractive

apt-get update

apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

sleep 3 && usermod -aG docker vagrant

kubeadm config images pull

SCRIPT

Vagrant.configure("2") do |config|
  config.vm.provider "virtualbox" do |v|
    v.memory = 2048
    v.cpus = 2
  end
  
  config.vm.define "master", primary: true do |master|
    master.vm.hostname = "k8s-master"
    master.vm.box = "ubuntu/bionic64"
    master.vm.network "private_network", ip: "192.168.2.11"
    master.vm.provision "shell", inline: $script
  end

  config.vm.define "node" do |node|
    node.vm.hostname = "k8s-node-1"
    node.vm.box = "ubuntu/bionic64"
    node.vm.network "private_network", ip: "192.168.2.12"
    node.vm.provision "shell", inline: $script
  end
end