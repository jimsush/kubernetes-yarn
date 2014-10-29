# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

# Require a recent version of vagrant otherwise some have reported errors setting host names on boxes
Vagrant.require_version ">= 1.6.2"

# The number of minions to provision
$num_minion = (ENV['KUBERNETES_NUM_MINIONS'] || 3).to_i

# ip configuration - also specified in cluster/vagrant/config-default.sh
$master_ip = "10.245.1.2"
$minion_ip_base = "10.245.1."
$minion_ips = $num_minion.times.collect { |n| $minion_ip_base + "#{n+3}" }
$minion_ips_str = $minion_ips.join(",")

# Determine the OS platform to use
$kube_os = ENV['KUBERNETES_OS'] || "fedora"

# OS platform to box information
$kube_box = {
  "fedora" => {
    "name" => "fedora20-salt-hadoop-s3",
    "box_url" => "http://bit.ly/fedora20-salt-hadoop-hosted-s3"
  }
}

# This stuff is cargo-culted from http://www.stefanwrobel.com/how-to-make-vagrant-performance-not-suck
# Give access to all cpu cores on the host
host = RbConfig::CONFIG['host_os']
if host =~ /darwin/
  $vm_cpus = `sysctl -n hw.ncpu`.to_i
elsif host =~ /linux/
  $vm_cpus = `nproc`.to_i
else # sorry Windows folks, I can't help you
  $vm_cpus = 2
end

# Give VM 1024MB of RAM
$vm_mem = 1024 


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  def customize_vm(config)
    config.vm.box = $kube_box[$kube_os]["name"]
    config.vm.box_url = $kube_box[$kube_os]["box_url"]

    config.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--memory", $vm_mem]
      v.customize ["modifyvm", :id, "--cpus", $vm_cpus]

      # Use faster paravirtualized networking
      v.customize ["modifyvm", :id, "--nictype1", "virtio"]
      v.customize ["modifyvm", :id, "--nictype2", "virtio"]
    end
  end

  # Kubernetes master
  config.vm.define "master" do |config|
    customize_vm config

    config.vm.provision "shell", inline: "/vagrant/cluster/vagrant/provision-master-existing-saltstack.sh #{$master_ip} #{$num_minion} #{$minion_ips_str}"
    config.vm.network "private_network", ip: "#{$master_ip}"
    config.vm.hostname = "kubernetes-master"
    config.vm.post_up_message = "Completed provisioning master. It may take some time for salt provisioning to finish."
  end

  # Kubernetes minion
  $num_minion.times do |n|
    config.vm.define "minion-#{n+1}" do |minion|
      customize_vm minion

      minion_index = n+1
      minion_ip = $minion_ips[n]
      minion.vm.provision "shell", inline: "/vagrant/cluster/vagrant/provision-minion-existing-saltstack.sh #{$master_ip} #{$num_minion} #{$minion_ips_str} #{minion_ip} #{minion_index}"
      minion.vm.network "private_network", ip: "#{minion_ip}"
      minion.vm.hostname = "kubernetes-minion-#{minion_index}"
      minion.vm.post_up_message = sprintf("Completed provisioning minion-%d. It may take some time for salt provisioning to finish.", n+1)
    end
  end

end
