# # SPDX-License-Identifier: GPL-3.0-only
# Copyright (C) 2025 Cloud Society Org.
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License version 3,
# as published by the Free Software Foundation.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <https://www.gnu.org/licenses/>.

# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'
require 'ipaddr'
require 'socket'

Vagrant.require_version ">= 2.0.0"

CONFIG = File.join(File.dirname(__FILE__), ENV['MOLECULE_VAGRANT_CONFIG'] || 'vagrant/config.rb')

# Uniq disk UUID
DISK_UUID = Time.now.utc.to_i

SUPPORTED_OS = {
  "rockylinux9"         => {box: "rockylinux/9",               user: "vagrant"}
}

if File.exist?(CONFIG)
  require CONFIG
end

def find_vbox_network()
  Socket.getifaddrs.each do |iface|
    # We are looking for a VirtualBox host-only interface
    next unless iface.name.start_with?('vboxnet')
    addr = iface.addr
    next unless addr && addr.afamily == Socket::AF_INET

    host_ip = addr.ip_address
    # For an IP like '192.168.39.1', the subnet prefix is '192.168.39'
    subnet_prefix = host_ip.split('.')[0..2].join('.')
    puts "Found VirtualBox network '#{iface.name}' with IP #{host_ip}. Using subnet #{subnet_prefix}.0/24."
    return { subnet: subnet_prefix, gateway: host_ip }
  end

  return nil
end

network_info = find_vbox_network()

if network_info.nil?
  puts "Error: Could not find a VirtualBox host-only network interface (e.g., 'vboxnet0')."
  puts "Please create one in VirtualBox: File -> Host Network Manager."
  exit 1
end

$subnet  = network_info[:subnet]
$gateway = network_info[:gateway]
$dns_servers = "8.8.8.8 1.1.1.1"  # Space separated!

# Defaults for config options defined in CONFIG
$num_instances ||= 6
$instance_name_prefix ||= "kafka"
$vm_gui ||= false
$vm_memory ||= 1024
$vm_cpus ||= 1
$shared_folders ||= {}
$forwarded_ports ||= {}
$os ||= "rockylinux9"
$inventories ||= []
$first_node ||= 1
$first_kafka_broker ||= 1
$first_kafka_controller ||= 4

$kafka_broker_instances ||= [$num_instances, 3].min
$kafka_broker_instances_name ||= "#{[$instance_name_prefix, '-broker'].join('')}"
$kafka_controller_instances ||= [$num_instances, 3].min
$kafka_controller_instances_name ||= "#{[$instance_name_prefix, '-ctrl'].join('')}"
$kafka_node_instances ||= $num_instances - $first_node + 1

$override_disk_size ||= false
$disk_size ||= 20480
$local_path_provisioner_enabled ||= "False"
$local_path_provisioner_claim_root ||= "/opt/local-path-provisioner/"
$ansible_verbosity ||= false
$ansible_tags ||= ENV['VAGRANT_ANSIBLE_TAGS'] || ""
$vagrant_dir ||= File.join(File.dirname(__FILE__), ".vagrant")
$playbook ||= "cluster.yml"
$extra_vars ||= {}

host_vars = {}

# throw error if os is not supported
if ! SUPPORTED_OS.key?($os)
  puts "Unsupported OS: #{$os}"
  puts "Supported OS are: #{SUPPORTED_OS.keys.join(', ')}"
  exit 1
end

$box = SUPPORTED_OS[$os][:box]

if Vagrant.has_plugin?("vagrant-proxyconf")
  $no_proxy = ENV['NO_PROXY'] || ENV['no_proxy'] || "127.0.0.1,localhost"
  (1..$num_instances).each do |i|
      $no_proxy += ",#{$subnet}.#{i+100}"
  end
end

Vagrant.configure("2") do |config|
  puts "Configuring VMs on the #{$subnet}.0/24 network with gateway #{$gateway}"

  config.vm.box = $box
  if SUPPORTED_OS[$os].has_key? :box_url
    config.vm.box_url = SUPPORTED_OS[$os][:box_url]
  end
  config.ssh.username = SUPPORTED_OS[$os][:user]

  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  config.ssh.insert_key = false

  if ($override_disk_size)
    unless Vagrant.has_plugin?("vagrant-disksize")
      system "vagrant plugin install vagrant-disksize"
    end
    config.disksize.size = $disk_size
  end

  (1..$num_instances).each do |i|
    if i.between?($first_kafka_broker, $kafka_broker_instances)
	vm_name = "#{$kafka_broker_instances_name}%02d" % i
    elsif i.between?($first_kafka_controller, $num_instances)
	vm_name = "#{$kafka_controller_instances_name}%02d" % (i - $kafka_broker_instances)
    else
	vm_name = "%s-%01d" % [$instance_name_prefix, i]
    end

    config.vm.define vm_name do |node|

      node.vm.hostname = vm_name

      if Vagrant.has_plugin?("vagrant-proxyconf")
        node.proxy.http     = ENV['HTTP_PROXY'] || ENV['http_proxy'] || ""
        node.proxy.https    = ENV['HTTPS_PROXY'] || ENV['https_proxy'] ||  ""
        node.proxy.no_proxy = $no_proxy
      end

      node.vm.provider :virtualbox do |vb|
	vb.name = vm_name
        vb.memory = $vm_memory
        vb.cpus = $vm_cpus
        vb.gui = $vm_gui
        vb.linked_clone = true
        vb.customize ["modifyvm", :id, "--vram", "8"]
        vb.customize ["modifyvm", :id, "--audio", "none"]
        vb.customize ['storagectl', :id, '--name', 'SATA Controller', '--add', 'sata', '--controller', 'IntelAhci', '--portcount', '8']
      end

      if $expose_docker_tcp
        node.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), auto_correct: true
      end

      $forwarded_ports.each do |guest, host|
        node.vm.network "forwarded_port", guest: guest, host: host, auto_correct: true
      end

      if ["rhel8"].include? $os
        node.vm.synced_folder ".", "/vagrant", disabled: false
        $shared_folders.each do |src, dst|
          node.vm.synced_folder src, dst
        end
      else
        node.vm.synced_folder ".", "/vagrant", disabled: false, type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z'] , rsync__exclude: ['.git','venv']
        $shared_folders.each do |src, dst|
          node.vm.synced_folder src, dst, type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z']
        end
      end
      
      ip = "#{$subnet}.#{i+100}"
      node.vm.network :private_network,
        :ip => ip

      # Disable swap for each vm
      node.vm.provision "shell", inline: "swapoff -a"


      node.vm.provision "shell", path: "detect_nm_dev.sh", env: {
	"DNS_SERVERS" => $dns_servers
      }

      # Provisioning block to update system and install base packages based on OS
      node.vm.provision "shell", inline: <<-SHELL
	echo ">>> Checking for package manager..."

	if command -v dnf &> /dev/null; then
	  # --- Red Hat-based Systems (dnf) ---
	  echo "DNF detected. Running updates for Red Hat-based OS."
	  dnf clean all
	  dnf makecache
	  dnf update -y
	  dnf install -y net-tools vim curl wget git

	elif command -v apt-get &> /dev/null; then
	  # --- Debian-based Systems (apt) ---
	  echo "APT detected. Running updates for Debian-based OS."
	  export DEBIAN_FRONTEND=noninteractive
	  apt-get clean
	  apt-get update -y
	  apt-get upgrade -y
	  apt-get install -y net-tools vim curl wget git

	else
	  echo "Unsupported package manager. Cannot provision packages." >&2
	  exit 1
	fi

	echo ">>> System provisioning complete."
      SHELL


      # Disable firewalld on redhat based vms
      if ["rhel8","rockylinux8","rhel9","rockylinux9"].include? $os
        node.vm.provision "shell", inline: "systemctl stop firewalld; systemctl disable firewalld"
      end

      host_vars[vm_name] = {
        "ip": ip,
        "download_localhost": "False",
        "local_path_provisioner_enabled": "#{$local_path_provisioner_enabled}",
        "local_path_provisioner_claim_root": "#{$local_path_provisioner_claim_root}",
        "ansible_ssh_user": SUPPORTED_OS[$os][:user],
        "ansible_ssh_private_key_file": File.join(Dir.home, ".vagrant.d", "insecure_private_key"),
        "unsafe_show_logs": "True"
      }

      if i == $num_instances
        node.vm.provision "ansible" do |ansible|
          ansible.playbook = $playbook
          ansible.compatibility_mode = "2.0"
          ansible.verbose = $ansible_verbosity
          ansible.become = true
          ansible.limit = "all,localhost"
          ansible.host_key_checking = false
          ansible.raw_arguments = ["--forks=#{$num_instances}",
                                   "--flush-cache",
                                   "-e ansible_become_pass=vagrant"] +
                                   $inventories.map {|inv| ["-i", inv]}.flatten
          ansible.host_vars = host_vars
          ansible.extra_vars = $extra_vars
          if $ansible_tags != ""
            ansible.tags = [$ansible_tags]
          end
          ansible.groups = {
            "kafka_broker" => (1..$kafka_broker_instances).map { |n| sprintf("#{$kafka_broker_instances_name}%02d", n) },
            "kafka_controller" => (1..$kafka_controller_instances).map { |n| sprintf("#{$kafka_controller_instances_name}%02d", n) },
            "kafka_cluster:children" => ["kafka_broker", "kafka_controller"],
          }
        end
      end

    end
  end
end
