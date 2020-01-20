# -*- mode: ruby -*-
# vi: set ft=ruby :

servers = [
    {
        :name => "k8s-head",
        :type => "master",
        :box => "centos/7",
        :box_version => "1905.1",
        :eth1 => "192.168.205.10",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-1",
        :type => "node",
        :box => "centos/7",
        :box_version => "1905.1",
        :eth1 => "192.168.205.11",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-2",
        :type => "node",
        :box => "centos/7",
        :box_version => "1905.1",
        :eth1 => "192.168.205.12",
        :mem => "2048",
        :cpu => "2"
    }
]

$configureBox = <<-SCRIPT
	# install docker
	yum install -y yum-utils jq net-tools device-mapper-persistent-data lvm2
	yum-config-manager --add-repo \
	https://download.docker.com/linux/centos/docker-ce.repo
	yum install -y docker-ce docker-ce-cli containerd.io
	systemctl enable docker
	systemctl start docker
		
	# run docker commands as vagrant user (not required)
	usermod -aG docker vagrant

	# enable ssh by password
	sed -i 's/PasswordAuthentication\ no/PasswordAuthentication\ yes/g' /etc/ssh/sshd_config
	service sshd restart

	# Configure Kubernetes Repository
	echo "[kubernetes]
	name=Kubernetes
	baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
	enabled=1
	gpgcheck=1
	repo_gpgcheck=1
	gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg" > /etc/yum.repos.d/kubernetes.repo
	sed -i 's/\t//g' /etc/yum.repos.d/kubernetes.repo
	
	# Set SELinux in permissive mode (effectively disabling it)
	setenforce 0
	sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

	# install kubeadm
	yum install -y kubelet kubeadm kubectl
	systemctl enable kubelet
	systemctl start kubelet

	# Update Iptables Settings
	echo "net.bridge.bridge-nf-call-ip6tables = 1
	net.bridge.bridge-nf-call-iptables = 1"  > /etc/sysctl.d/k8s-head.conf
	sysctl --system

	# kubelet requires swap off
	sed -i '/swap/d' /etc/fstab
	swapoff -a

	#set cgroup driver
	sed -i "/^[^#]*KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--cgroup-driver=cgroupfs" /etc/sysconfig/kubelet
	systemctl restart kubelet
SCRIPT

$configureMaster = <<-SCRIPT
    echo "This is master"
    # ip of this box
    IP_ADDR=`ifconfig eth1 | grep mask | awk '{print $2}'| cut -f2 -d:`

	# set default ip for box
	sed 's/127\.0\.0\.1/192\.168\.205\.10/g' -i /etc/hosts
	hostname -i
	
	# install k8s master
    HOST_NAME=$(hostname -s)
    kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.160.0.0/16

    #copying credentials to regular user - vagrant
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    sudo cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    sudo chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

	# install flannel pod network
	sudo --user=vagrant kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh

    # required for setting up passwordless ssh between guest VMs
    sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    service sshd restart

SCRIPT

$configureNode = <<-SCRIPT
    echo "This is worker"
    yum install -y sshpass
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
            	v.customize ["modifyvm", :id, "--groups", "/K8S"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]

            end

            config.vm.provision "shell", inline: $configureBox

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
            else
                config.vm.provision "shell", inline: $configureNode
            end

        end

    end

end
