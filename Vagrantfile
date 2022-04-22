# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'

Vagrant.require_version ">= 2.0.0"

CONFIG = File.join(File.dirname(__FILE__), "vagrant/config.rb")


SUPPORTED_OS = {
  "ubuntu"        => {box: "netlox/loxilight", bootstrap_os: "ubuntu", user: "vagrant"},
}

# Defaults for config options defined in CONFIG
$num_instances = 4
$instance_name_prefix = "k8s"
$vm_gui = false
$vm_memory = 8096
$vm_cpus = 4
$shared_folders = {}
$forwarded_ports = {}
$subnet = "192.168.48"
$datasubnet = "192.168.58"
$os = "ubuntu"
# The first three nodes are etcd servers
$etcd_instances = $num_instances
# The first two nodes are kube masters
$kube_master_instances = $num_instances == 1 ? $num_instances : ($num_instances - 1)
# The following only works when using the libvirt provider
$local_release_dir = "/vagrant/temp"

host_vars = {}

if File.exist?(CONFIG)
  require CONFIG
end

$box = SUPPORTED_OS[$os][:box]

if Vagrant.has_plugin?("vagrant-proxyconf")
    $no_proxy = ENV['NO_PROXY'] || ENV['no_proxy'] || "127.0.0.1,localhost"
    (1..$num_instances).each do |i|
        $no_proxy += ",#{$subnet}.#{i+100}"
    end
end

Vagrant.configure("2") do |config|
  # always use Vagrants insecure key
  config.ssh.insert_key = false
  config.vm.box = $box
  #config.vm.box_version = "1.0.0"
  if SUPPORTED_OS[$os].has_key? :box_url
    config.vm.box_url = SUPPORTED_OS[$os][:box_url]
  end
  config.ssh.username = SUPPORTED_OS[$os][:user]
  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end
  (1..$num_instances).each do |i|
    config.vm.define vm_name = "%s-%02d" % [$instance_name_prefix, i] do |config|
      config.vm.hostname = vm_name

      if Vagrant.has_plugin?("vagrant-proxyconf")
        config.proxy.http     = ENV['HTTP_PROXY'] || ENV['http_proxy'] || ""
        config.proxy.https    = ENV['HTTPS_PROXY'] || ENV['https_proxy'] ||  ""
        config.proxy.no_proxy = $no_proxy
      end

      if $expose_docker_tcp
        config.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), auto_correct: true
      end

      $forwarded_ports.each do |guest, host|
        config.vm.network "forwarded_port", guest: guest, host: host, auto_correct: true
      end

      ["vmware_fusion", "vmware_workstation"].each do |vmware|
        config.vm.provider vmware do |v|
          v.vmx['memsize'] = $vm_memory
          v.vmx['numvcpus'] = $vm_cpus
        end
      end

      config.vm.synced_folder "loxilight_bin", "/vagrant", type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z']

      config.vm.provider :virtualbox do |vb|
        vb.gui = $vm_gui
        vb.memory = $vm_memory
        vb.cpus = $vm_cpus
      end

     config.vm.provider :libvirt do |lv|
       lv.memory = $vm_memory
     end

      ip = "#{$subnet}.#{i+100}"
      dataip = "#{$datasubnet}.#{i+100}"
      host_vars[vm_name] = {
        "ip": ip,
        "bootstrap_os": SUPPORTED_OS[$os][:bootstrap_os],
        "local_release_dir" => $local_release_dir,
        "download_run_once": "False"
      }

      config.vm.network :private_network, ip: ip
      config.vm.network :private_network, ip: dataip

     end
  end

  (1..$num_instances).each do |i|
    config.vm.define vm_name = "%s-%02d" % [$instance_name_prefix, i] do |config|
      config.vm.hostname = vm_name

      # Disable swap for each vm
      config.vm.provision "0", type: "shell", inline: "swapoff -a"
      config.vm.provision "shell", inline: <<-SHELL
        sudo apt update
        sudo apt install -y lldpd
        sudo /etc/init.d/lldpd stop
        sudo ip netns exec loxilight lldpd
        mkdir -p $HOME/.kube
        hostnamectl set-hostname node"#{i}"
      SHELL

      # Make a kubespary in the master node
      if config.vm.hostname == "node1" || config.vm.hostname == "k8s-01"
          config.vm.provision "shell", privileged: false, inline: <<-SHELL
            git clone https://github.com/kubernetes-sigs/kubespray.git
            sudo apt update; sudo apt install -y python3-pip sshpass; sudo pip install -r kubespray/requirements.txt
            cp -rfp kubespray/inventory/sample kubespray/inventory/mycluster
            CONFIG_FILE=kubespray/inventory/mycluster/hosts.yml python3 kubespray/contrib/inventory_builder/inventory.py 192.168.48.101 192.168.48.102 192.168.48.103 192.168.48.104
            ssh-keygen -b 2048 -t rsa -f /home/vagrant/.ssh/id_rsa -q -N ""
            chown vagrant:vagrant /home/vagrant/.ssh/*
            sshpass -pvagrant ssh-copy-id vagrant@192.168.48.101  -oStrictHostKeyChecking=no
            sshpass -pvagrant ssh-copy-id vagrant@192.168.48.102  -oStrictHostKeyChecking=no
            sshpass -pvagrant ssh-copy-id vagrant@192.168.48.103  -oStrictHostKeyChecking=no
            sshpass -pvagrant ssh-copy-id vagrant@192.168.48.104  -oStrictHostKeyChecking=no

            ansible-playbook -i kubespray/inventory/mycluster/hosts.yml -e kube_network_plugin=cni -become --become-user=root  kubespray/cluster.yml

            sudo kubectl delete daemonset -n kube-system kube-proxy
            sudo ip link del kube-ipvs0
            sudo iptables -F
            sudo iptables -F -t nat
            sudo ipvsadm -C
            sudo snap install helm --classic
            sudo helm repo add loxi-agent-helm-repo https://backguynn.github.io/loxi-agent-helm-repo/
            sudo helm install -f /vagrant/loxi-config.yml loxi-namu-pkg loxi-agent-helm-repo/loxi-agent
            sudo cp -r /root/.kube/config /home/vagrant/config
            sudo chown vagrant:vagrant /home/vagrant/config
          SHELL
      end
     end
  end
end

